# Bub v0.3.0 — 執行路徑審計

**路徑**：`/Users/lesprivilege/Downloads/bub-main`
**語言**：Python 3.12+，~3,300 行核心 (src/bub/)
**外部核心**：`republic` (LLM/tool/tape runtime)，`pluggy` (hook)

---

## 模組依賴圖

```
__main__.py  →  framework.py
                     ├── configure.py
                     ├── envelope.py
                     ├── hook_runtime.py
                     │    └── hookspecs.py  (pluggy contract)
                     ├── types.py
                     ├── builtin/hook_impl.py
                     │    ├── builtin/agent.py  (~657行)
                     │    │    ├── builtin/tape.py  (TapeService)
                     │    │    ├── builtin/store.py  (ForkTapeStore, FileTapeStore)
                     │    │    ├── builtin/settings.py
                     │    │    ├── skills.py  (discovery)
                     │    │    └── tools.py  (REGISTRY)
                     │    ├── builtin/tools.py  (@tool decorators)
                     │    ├── builtin/cli.py
                     │    ├── builtin/context.py
                     │    ├── builtin/shell_manager.py
                     │    └── builtin/store.py
                     ├── channels/handler.py  (BufferedMessageHandler)
                     ├── channels/manager.py
                     ├── channels/base.py  (Channel ABC)
                     ├── channels/cli/
                     └── channels/telegram.py
```

**關鍵觀察**：`framework.py:305` 行 — 核心極輕。真正的 LLM/tool 循環在 `republic` 庫中（不在此 repo）。Bub 主要是 pluggy hook 膠水 + tape (session) 管理。

---

## 1. 入口 → dispatch

```
__main__.py:43  app = create_cli_app()
  └─ __main__.py:28  create_cli_app()
        ├─ _instrument_bub()                           :12    — logfire 選配
        ├─ BubFramework()                             :30    — 建構 pluggy PM + 載入 config
        │    └─ framework.py:43  __init__
        │         ├─ pluggy.PluginManager("bub")      :46
        │         ├─ add_hookspecs(BubHookSpecs)       :47
        │         ├─ HookRuntime(plugin_manager)       :48
        │         └─ configure.load(config_file)       :52    — YAML config
        ├─ framework.load_hooks()                     :31
        │    └─ framework.py:66  load_hooks()
        │         ├─ _load_builtin_hooks()             :54
        │         │    └─ plugin_manager.register(BuiltinImpl, "builtin")  :60
        │         └─ entry_points(group="bub")         :72    [PLUGIN DISCOVERY]
        │              └─ plugin_manager.register(plugin, name)            :85
        └─ framework.create_cli_app()                  :32
             └─ framework.py:92  create_cli_app()
                  ├─ typer.Typer(name="bub")           :94
                  ├─ @app.callback(...)                :96    — --workspace 選項
                  └─ hook_runtime.call_many_sync(      :105
                       "register_cli_commands", app=app)
                     → builtin/hook_impl.py:168  BuiltinImpl.register_cli_commands
                       └─ app.command("run")(cli.run)  :171
                       └─ app.command("chat")(cli.chat):172
                       └─ app.command("gateway")(cli.gateway):176
```

**錯誤路徑**：
- Plugin 載入失敗 → `logger.warning` + 記入 `_plugin_status`，不中斷啟動 (`framework.py:74-78`)
- Builtin 註冊失敗 → `PluginStatus(is_success=False)`，不中斷 (`framework.py:61-62`)

---

## 2. 完整 Turn Pipeline (`bub run "message"`)

```
cli.py:38  run(ctx, message)
  └─ ChannelMessage(session_id, content, channel="cli", ...)   :49
  └─ asyncio.run(framework.process_inbound(inbound))           :61
       framework.py:108  process_inbound()

       [STAGE 1: resolve_session]
       → hook_runtime.call_first("resolve_session", message=inbound)    :112
         → builtin/hook_impl.py:105  resolve_session
           { session_id from message field, or fallback "channel:chat_id" }
         [fallback] framework._default_session_id(inbound)              :114 → :211

       [STAGE 2: load_state]
       → hook_runtime.call_many("load_state", message, session_id)     :118
         → builtin/hook_impl.py:114  load_state
           { session_id, _runtime_agent, context }
           [NON-FATAL] lifespan.__aenter__() if present        :117
         ← state dict merged (later hooks override earlier)    :119-122

       [STAGE 3: build_prompt]
       → hook_runtime.call_first("build_prompt", message, session_id, state)  :123
         → builtin/hook_impl.py:131  build_prompt
           { content_of(message) → context prefix → media parts → text or list[dict] }
         [fallback] content_of(inbound) if prompt is empty     :126-127

       [STAGE 4: run_model]
       → _run_model(inbound, prompt, session_id, state, stream_output)  :130
         framework.py:149  _run_model()
           if not stream_output:
             → hook_runtime.run_model(prompt, session_id, state)       :158
               hook_runtime.py:163  run_model()
                 for plugin in reversed(list_name_plugin()):
                   if hasattr(plugin, "run_model"):
                     → call_first("run_model", ...)
                       → builtin/hook_impl.py:160  run_model
                         → Agent.run(session_id, prompt, state)        :161
                           agent.py:87  run()
                             → tapes.session_tape(session_id, workspace)  :99
                             → fork_tape(tape.name, merge_back=True)    :102
                               [ASYNC context manager]
                               → ensure_bootstrap_anchor                :103
                               → _agent_loop(tape, prompt, ...)         :106

       [STAGE 4b: Agent Loop]
       agent.py:216  _agent_loop()
         → _run_tools_with_auto_handoff(tape, prompt, ...)            :250
           for step in range(1, max_steps=50):                         :269
             → _run_once(tape, prompt, model, tools, skills)           :274
               agent.py:521  _run_once()
                 → _system_prompt(prompt, state, allowed_skills)       :545
                 → tape.run_tools_async(prompt, system_prompt,         :553
                     max_tokens, tools, model)
                   [ASYNC] → republic library → LLM API call
                     ← ToolAutoResult (kind="text"|"tools"|"error")
             → _resolve_tool_auto_result(output)                       :296
               ↓ kind:
               "text"     → return outcome.text                        :309
               "continue" → next_prompt = CONTINUE_PROMPT              :310-314
               "error"    → auto_handoff if context-length error       :328-353
                          → raise RuntimeError                         :366
             ↑ loop repeats if "continue"
           → raise RuntimeError("max_steps_reached=50")                :368

       [ASYNC] fork_tape exit → merge_back (in-memory → FileTapeStore)

       [STAGE 5: save_state — finally block]
       framework.py:131  finally:
         hook_runtime.call_many("save_state", session_id, state, ...)  :132
           → builtin/hook_impl.py:124  save_state
             [NON-FATAL] lifespan.__aexit__(exc_type, value, tb)       :128

       [STAGE 6: render_outbound]
       → _collect_outbounds(message, session_id, state, model_output) :140
         framework.py:219  _collect_outbounds()
           → hook_runtime.call_many("render_outbound", ...)            :226
             → builtin/hook_impl.py:282  render_outbound
               ← [ChannelMessage(session_id, channel, content=model_output)]
           [fallback] if no outbounds: rebuild from model_output + metadata :239-249

       [STAGE 7: dispatch_outbound]
       for outbound in outbounds:                                      :141
         hook_runtime.call_many("dispatch_outbound", message=outbound) :142
           → builtin/hook_impl.py:274  dispatch_outbound
             → framework.dispatch_via_router(message)                  :279
               → outbound_router.dispatch_output(message)              :204
                 (CLI channel: prints to stdout; Telegram: sends API)

       ← TurnResult(session_id, prompt, model_output, outbounds)      :143

     [CRITICAL] outer try/except → logger.exception + on_error hook   :145-146
```

**錯誤路徑**：
- `save_state` 在 `finally` 中 — 即使 model 拋錯也會執行 (`framework.py:131`)
- `on_error` hook 吞掉 observer 的失敗 (`hook_runtime.py:83`)
- Context-length 錯誤觸發 auto-handoff（最多 1 次 retry, `agent.py:328-353`）
- `run_model` 返回 None → fallback 到 prompt 原文 (`framework.py:159-165`)

---

## 3. Tape (Session) 生命週期

```
Agent.run()
  └─ tapes.session_tape(session_id, workspace)     tape.py:120
       → hash(workspace)[:16] + "__" + hash(session_id)[:16]  :121-124
       → LLM.tape(tape_name)  ← republic library
  └─ tapes.fork_tape(tape.name, merge_back=True)    tape.py:127
       store.py:101  fork()
         token = current_store.set(InMemoryTapeStore())   :104
         → [yield] → Agent loop runs here
         ← [finally]
           if merge_back:
             entries = store.read(tape)                    :117
             for entry in entries:
               parent.append(tape, entry)                  :121  [WRITE-THROUGH]
           current_store.reset(token)                      :111
```

**持久化** (`store.py:144 FileTapeStore`)：
- JSONL 檔案在 `~/.bub/tapes/{hash}.jsonl`
- 每行一個 `TapeEntry` JSON，含 `id`, `kind`, `payload`, `meta`, `date`
- `TapeFile` 用 `threading.Lock` 保護讀寫 (`store.py:248`)
- 增量讀取：維護 `_read_offset` 只讀新增行 (`store.py:271-294`)
- Reset：`path.unlink()` + 清除記憶體快取 (`store.py:261-265`)
- 查詢：FTS 用 `rapidfuzz` WRatio fuzzy match (`store.py:160-217`)

---

## 4. Hook 系統

| Hook | 類型 | 目的 | Builtin 實作 |
|------|------|------|-------------|
| `resolve_session` | `firstresult` | 決定 session ID | `ChannelMessage` 的 session_id 或 `channel:chat_id` |
| `load_state` | `call_many` | 累積 session state | `{session_id, _runtime_agent, context}` |
| `build_prompt` | `firstresult` | 組裝 LLM prompt | content + context prefix + media parts |
| `run_model` / `run_model_stream` | `firstresult` | LLM 調用 | `Agent.run()` → republic |
| `save_state` | `call_many` | 持久化 state | lifespan cleanup |
| `render_outbound` | `call_many` | model output → outbound messages | `ChannelMessage` 包裝 |
| `dispatch_outbound` | `call_many` | 發送 outbound | `framework.dispatch_via_router` |
| `system_prompt` | `call_many` | 提供 system prompt 區塊 | `DEFAULT_SYSTEM_PROMPT` + `AGENTS.md` |
| `provide_tape_store` | `firstresult` | TapeStore 實例 | `FileTapeStore(~/.bub/tapes/)` |
| `provide_channels` | `call_many` | Channel 列表 | CLI + Telegram |
| `register_cli_commands` | `call_many` | CLI 子命令 | run/chat/gateway/onboard/install/hooks |
| `onboard_config` | `call_many` | 互動配置 | 問 provider/model/api_key/channels |
| `on_error` | `call_many` | 錯誤通知 | 發送 error ChannelMessage + log |
| `build_tape_context` | `firstresult` | Tape context selector | `default_tape_context()` |

**關鍵設計**：
- `firstresult=True` → 最高優先級 hook 勝出（按 register 順序 reverse）
- `call_many` → 所有 hook 累積結果（state update, outbound 收集）
- Plugin 優先級：builtin 先註冊 → 外部 plugin 後註冊 → runtime 時 reversed → 外部優先 (`hook_runtime.py:157`)
- `_SKIP_VALUE` sentinel：hook 可選擇跳過 (`hook_runtime.py:150`)

---

## 5. 並行模型

```
BubFramework     — single-threaded asyncio
Agent            — asyncio (async LLM calls via republic)
ForkTapeStore    — asyncio context manager (fork/merge)
FileTapeStore    — threading.Lock per tape file
BufferedMessageHandler — asyncio.Event + call_later debounce
ChannelManager   — asyncio.gather channels
TelegramChannel  — asyncio (python-telegram-bot's update loop)
```

沒有 threading 除了 JSONL file I/O 鎖。沒有 producer-consumer 佇列——channel message handler 是直接 callback。

---

## 6. 關鍵設計觀察

| 觀察 | 位置 | 說明 |
|------|------|------|
| **Framework 極輕** | `framework.py:305` | 核心 logic 幾乎全在 pluggy hook 調度。真正的 LLM/tool loop 在 `republic` |
| **Fork-merge tape isolation** | `store.py:31-123` | `contextvars` 實現的 transaction-like fork：turn 期間寫入 in-memory store，完成後 merge 到 parent。失敗時不寫入 |
| **Session 是 hash-based** | `tape.py:121-125` | Session ID 不直接當檔名，而是 `md5(workspace)[:16] + "__" + md5(session_id)[:16]`。沒有 session listing/搜尋 |
| **Auto-handoff 只針對 context-length** | `agent.py:328-353` | 唯一自動恢復的錯誤類型。regex 匹配 `_CONTEXT_LENGTH_PATTERNS`。僅 1 次 retry |
| **無 persist actor** | — | 每次 entry append 直接寫 JSONL (`store.py:320-328`)。無 coalescing、無 batch write、無 background flush |
| **Error 回注給 LLM** | `agent.py:296-366` | Tool 失敗不中斷 loop——`ToolAutoResult` 編碼錯誤，LLM 在下輪看到錯誤訊息自行決策 |
| **System prompt 是累積而非取代** | `framework.py:278-283` | `system_prompt` hook 是多個結果的 `\n\n` 拼接，不是 winner-takes-all |
| **CLI 和 Telegram 共用管線** | `builtin/hook_impl.py:252-258` | 同一 `message_handler` callback 同時餵給 CLI 和 Telegram channel |

---

## 7. 驗證命令

```bash
grep -n "def process_inbound" src/bub/framework.py              # 108
grep -n "def _run_model" src/bub/framework.py                    # 149
grep -n "def _collect_outbounds" src/bub/framework.py            # 219
grep -n "def load_hooks" src/bub/framework.py                    # 66
grep -n "class BubHookSpecs" src/bub/hookspecs.py                # 21
grep -n "class HookRuntime" src/bub/hook_runtime.py              # 16
grep -n "class BuiltinImpl" src/bub/builtin/hook_impl.py         # 59
grep -n "class Agent" src/bub/builtin/agent.py                   # 50
grep -n "async def _agent_loop" src/bub/builtin/agent.py         # 216
grep -n "async def _run_tools_with_auto_handoff" src/bub/builtin/agent.py  # 258
grep -n "async def _run_once" src/bub/builtin/agent.py           # 521
grep -n "class ForkTapeStore" src/bub/builtin/store.py           # 31
grep -n "class FileTapeStore" src/bub/builtin/store.py           # 144
grep -n "class TapeService" src/bub/builtin/tape.py              # 36
grep -n "def session_tape" src/bub/builtin/tape.py               # 120
grep -n "def run\b" src/bub/builtin/cli.py                       # 38
grep -n "def chat\b" src/bub/builtin/cli.py                      # 93
grep -n "class Channel" src/bub/channels/base.py                 # 11
grep -n "class BufferedMessageHandler" src/bub/channels/handler.py  # 9
grep -n "class ToolAutoResult\|def _resolve_tool_auto_result" src/bub/builtin/agent.py  # 592
```
