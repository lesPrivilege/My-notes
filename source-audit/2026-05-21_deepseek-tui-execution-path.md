# DeepSeek-TUI 執行路徑審計報告

日期：2026-05-21  
專案：/Users/lesprivilege/Projects/DeepSeek-TUI/  
版本：0.8.39（14-crate Rust workspace，edition 2024）  
審計範圍：架構依賴圖、單輪 Turn 完整呼叫鏈、Snapshot Rollback 生命週期

---

## 任務 A：Crate 依賴圖

### 14 個 workspace crate

| Crate | Binary | 描述 | 依賴 | 被依賴 |
|---|---|---|---|---|
| `deepseek-tui` | `deepseek-tui` | TUI 主應用（含內建 engine、client、tools） | secrets, tools | （無） |
| `deepseek-tui-cli` | `deepseek` | CLI 入口 facade | agent, app-server, config, execpolicy, mcp, secrets, state | （無） |
| `deepseek-app-server` | （library） | Codex 風格的 app-server transport | agent, config, core, execpolicy, hooks, mcp, protocol, state, tools | cli |
| `deepseek-core` | （library） | 核心執行時邊界 | agent, config, execpolicy, hooks, mcp, protocol, state, tools | app-server |
| `deepseek-agent` | （library） | Model/provider registry + fallback | config | core, app-server, cli |
| `deepseek-config` | （library） | Config schema + precedence | secrets | agent, core, app-server, cli |
| `deepseek-execpolicy` | （library） | Execution policy + approval | protocol | core, app-server, cli |
| `deepseek-hooks` | （library） | Hook dispatch + notifications | protocol | core, app-server |
| `deepseek-mcp` | （library） | MCP server lifecycle | （none） | core, app-server, cli |
| `deepseek-protocol` | （library） | Protocol frames | （none） | execpolicy, hooks, core, app-server, tools |
| `deepseek-secrets` | （library） | Secret storage backends | （none） | config, tui, cli |
| `deepseek-state` | （library） | SQLite session/thread persistence | （none） | core, app-server, cli |
| `deepseek-tools` | （library） | Tool invocation lifecycle | protocol | core, app-server, tui |
| `deepseek-tui-core` | （library） | TUI state machine scaffold | （none） | （none） |

### 關鍵角色

- **CLI 入口 binary**：`deepseek` — `crates/cli/src/main.rs:1`
- **TUI 入口 binary**：`deepseek-tui` — `crates/tui/src/main.rs:671` (async main)
- **Engine 主循環**：持有在 `deepseek-tui` crate 內部 — `crates/tui/src/core/engine.rs:576` (`Engine::run`)
- **Leaf crates（不被任何內部 crate 依賴）**：
  - `deepseek-mcp`
  - `deepseek-protocol`
  - `deepseek-secrets`
  - `deepseek-state`
  - `deepseek-tui-core`
  - `deepseek-tui`（TUI binary，leaf 在 workspace 層級）

### 重要觀察

1. `deepseek-tui` crate 不依賴 `deepseek-core`、`deepseek-state`、`deepseek-protocol`、`deepseek-hooks`、`deepseek-agent`。TUI 內建了自己的 engine (`crates/tui/src/core/engine.rs`)、session (`crates/tui/src/core/session.rs`)、client (`crates/tui/src/client.rs`)、工具執行 (`crates/tui/src/core/engine/tool_execution.rs`) 等。
2. `deepseek-core` crate 只有 `lib.rs`（定義了 runtime boundaries），實際的 engine 實作在 TUI crate 內。
3. `deepseek-cli` crate (`deepseek` binary) 是輕量級 facade：解析 CLI args → 調用 `deepseek-app-server` 或通過 `exec`/`resume` 子命令分發到 TUI binary。

---

## 任務 B：單輪 Turn 完整 Call Chain

### 1. 用戶輸入 → Engine 收到 prompt

```
crates/tui/src/main.rs:671  async fn main()
  → crates/tui/src/tui/ui.rs:223  async fn run_tui()
    → crates/tui/src/tui/ui.rs:797  async fn event_loop(app, engine_handle, ...)
      → crates/tui/src/tui/ui.rs:1865  dispatch_user_message(app, config, &engine_handle, message)
        [PERSIST] crates/tui/src/tui/persistence_actor.rs:100  persist(Checkpoint(...)) — 非同步 checkpoint 儲存
        [ASYNC] crates/tui/src/tui/ui.rs:3918–3922  auto_router::resolve_auto_model_selection — 如果 auto_model
        → crates/tui/src/tui/ui.rs:3967  engine_handle.send(Op::SendMessage{...})
          → crates/tui/src/core/engine/handle.rs:18  EngineHandle::send
            → self.tx_op.send(op).await  — 送入 channel，由 engine loop 接收
              [ASYNC BOUNDARY]
              → crates/tui/src/core/engine.rs:577  Engine::run — rx_op.recv().await
                → crates/tui/src/core/engine.rs:593  Engine::handle_send_message
```

**錯誤路徑**：
- `dispatch_user_message` (ui.rs:3967) 如果 send 失敗 (channel closed)，設 `is_loading = false` 並 return Err。
- `Engine::run` 的 `rx_op.recv().await` 如果回傳 `None`（channel 關閉），loop 自然結束。

### 2. Mode 分支點

Mode 分支點在 `handle_send_message` 內部（engine.rs:879），但 mode 的實際影響表現在：
- **System prompt 組裝**：`prompts::system_prompt_for_mode_with_context_skills_session_and_approval(mode, ...)` — engine.rs:1783
- **Tool catalog 構建**：`build_tool_context(mode, auto_approve)` — engine.rs:1376
- **Tool registry builder**：`build_turn_tool_registry_builder(mode, ...)` — engine.rs:992
- **工具執行時的檢查**：在 `handle_deepseek_turn` 中，Plan mode 封鎖 exec_shell 等工具 — turn_loop.rs:1115–1130
- **Loop guard 的策略**：`AppMode` 的 `label()` 影響 guard 行為

Mode 的枚舉定義在 `crates/tui/src/tui/app.rs:126`：
- `AppMode::Agent` — 完整工具權限（需審批）
- `AppMode::Yolo` — 完整工具權限（自動審批）
- `AppMode::Plan` — 唯讀模式（shell/exec 被封鎖）

**分支判斷位置**：mode 的影響散布在多個點，沒有單一的 if/match 分支點。最集中的分支在 `handle_deepseek_turn` 中工具規劃階段（turn_loop.rs:1115）：
```
if mode == AppMode::Plan && matches!(tool_name, "exec_shell"|...)
    → blocked_error = ToolError::permission_denied
```

以及 `build_tool_context` (engine.rs:1376) 中對 mode 的全面判斷。

### 3. Auto Mode 的 model/effort selection

**實際文件名與入口**：
- `crates/tui/src/tui/auto_router.rs` — `resolve_auto_model_selection` (line 23)
- `crates/tui/src/auto_reasoning.rs` — `select` (line 21)

**調用鏈**：
```
dispatch_user_message (ui.rs:3918–3922)
  [if auto_model] auto_router::should_resolve_auto_model_selection(app) — auto_router.rs:17
    → auto_router::resolve_auto_model_selection(app, config, &message, &content) — auto_router.rs:23
      → commands::resolve_auto_route_with_flash(config, latest_request, recent_context, ...) — commands/config.rs:1040
        [Heuristic fast-path]
        → auto_model_heuristic_selection_with_bias — commands/config.rs:1048
          [If Heuristic::Decisive → return immediately]
        [Flash router slow-path (4s timeout)]
        → auto_route_flash_recommendation — commands/config.rs:1086
          → DeepSeekClient::new(config) — commands/config.rs:1097
          → client.create_message(request) — commands/config.rs:1129
            [4s timeout, model="deepseek-v4-flash", stream=false]
          → parse_auto_route_recommendation — commands/config.rs:1130
        [Fallback to heuristic on error/timeout]
      → auto_route_from_heuristic — commands/config.rs:1072
        → auto_reasoning::select(false, latest_request) — auto_reasoning.rs:21
          [Keyword-based effort selection]

  effective_model = auto_selection?.model || auto_model_heuristic(&message.display, &app.model) — ui.rs:3924

  effective_reasoning_effort = auto_selection?.reasoning_effort || normalize_auto_routed_effort(auto_reasoning::select(...)) — ui.rs:3933
```

**錯誤路徑**：
- `auto_route_flash_recommendation` 4s timeout → `Err` → fallback 到 heuristic。
- 如果 `auto_router` 模組返回 `None` → `auto_route_from_heuristic` 做 keyword-based 選擇。
- `auto_reasoning::select` 永不失敗（純 function）。

### 4. Context 組裝

組裝順序（全部在 `handle_deepseek_turn` 內，turn_loop.rs）：

```
1. refresh_system_prompt(mode) — turn_loop.rs:74
   → crates/tui/src/core/engine.rs:1780
     → prompts::system_prompt_for_mode_with_context_skills_session_and_approval
       (mode, workspace, memory_block, skills_dir, instructions, session_context, approval_mode)
     → merge_system_prompts(base, compaction_summary_prompt)  — engine.rs:1799
     → hash check → update session.system_prompt if changed

2. Auto-compaction (pre-request) — turn_loop.rs:90–167
   [if compaction.enabled && should_compact()]
     → compact_messages_safe(client, messages, config, ...) — 調用 LLM 做 summary
     → merge_compaction_summary — engine.rs:1811

3. LSP diagnostics flush — turn_loop.rs:212
   → flush_pending_lsp_diagnostics() — lsp_hooks.rs:110
     → 將累積的 diagnostics 作為 synthetic user message 插入

4. Layered context checkpoint (#159) — turn_loop.rs:217
   → layered_context_checkpoint() — engine.rs:1484

5. Build MessageRequest — turn_loop.rs:285–306
   {
     model: self.session.model,
     messages: messages_with_turn_metadata() — turn_loop.rs:1937
       (messages + <turn_meta> blocks on user messages),
     system: self.session.system_prompt,
     tools: active_tools_for_step(tool_catalog, active_tool_names, ...),
     reasoning_effort: resolve_auto_effort(),
     stream: Some(true),
   }

6. Tool list assembly — engine.rs:1098
   → build_model_tool_catalog(registry.to_api_tools_with_cache(true), mcp_tools, mode)
     (per-turn tool set, filtered by mode)
```

**錯誤路徑**：
- Auto-compaction 失敗 → log error + 繼續使用原 messages (turn_loop.rs:159-165)
- LSP diagnostics flush 無資料 → 靜默返回 (lsp_hooks.rs:111)
- refresh_system_prompt hash 未變 → 跳過更新

### 5. API 調用

**Context 組裝完畢 → HTTP 請求的完整路徑**：

```
handle_deepseek_turn — turn_loop.rs:312
  → client.create_message_stream(stream_request)  — client.rs:816–821
    [ASYNC BOUNDARY — crate internal, no trait dispatch]
    → DeepSeekClient::handle_chat_completion_stream(request) — client/chat.rs:144
      → build_chat_messages_for_request_and_provider(request, provider) — client/chat.rs:149
      → sanitize_thinking_mode_messages(&mut body, ...) — client/chat.rs:193
      → self.send_with_retry(|| http_client.post(&url).json(&body))  — client/chat.rs:202
        [ASYNC — with_retry 封裝 — llm_client/mod.rs]
        → reqwest POST 到 {base_url}/chat/completions
```

**API Response 格式**：**SSE streaming chunks**，OpenAI-compatible Chat Completions 格式。

**Response Parsing**：
```
HTTP Response Body (bytes_stream)
  → async_stream::stream! — client/chat.rs:227–end
    → 每個 chunk: parse_sse_chunk 或逐行解析
      → yield Ok(StreamEvent)
        • StreamEvent::MessageStart — 合成（client/chat.rs:231）
        • StreamEvent::ContentBlockStart — line buffer 解析
        • StreamEvent::ContentBlockDelta — 文字/thinking/JSON 增量
        • StreamEvent::ContentBlockStop — content block 結束
        • StreamEvent::MessageDelta — 包含 usage
        • StreamEvent::MessageStop — stream 結束

StreamEvent 枚舉 — crates/tui/src/models.rs
  各種 variant 定義在 models crate，但具體 parsing 在 client/chat.rs

[ASYNC] Stream events 通過 async_stream::stream! 逐個 yield
       → engine 端用 StreamExt::next() 接收（turn_loop.rs:395）
```

**錯誤路徑**：
- HTTP 非 2xx 狀態碼 → 檢查 `reasoning_content` 錯誤，log thinking_mode violations，bail
- SSE idle timeout (300s default) → yield Err，engine 端決定是否透明 retry
- Stream decode error (reqwest) → error chain logging，count errors

### 6. Tool Call 解析

DeepSeek API 使用 **標準 OpenAI tool_use 格式**，通過 SSE `ContentBlockStart::ToolUse` 事件傳遞。

```
Stream parsing (client/chat.rs — SSE 處理 loop):
  SSE chunk → 解析為 ContentBlockStart/Delta/Stop
    → ContentBlockStart::ToolUse { id, name, input, caller }
      → tool_uses.push(ToolUseState{...})  — turn_loop.rs:584
    → Delta::InputJsonDelta { partial_json }  — turn_loop.rs:646
      → 累積到 tool_state.input_buffer
      → 試解析 JSON — turn_loop.rs:655
    → ContentBlockStop { index }  — turn_loop.rs:665
      → 最終解析 input_buffer 為完整 JSON — turn_loop.rs:697
      → 發送 Event::ToolCallStarted — turn_loop.rs:728

[#103 transparent retry] 如果 stream 在 tool call 完成前中斷且無內容
  → should_transparently_retry_stream() — streaming.rs:81
    → client.create_message_stream(stream_request) 重新發送

[Fallback: markdown-based tool call parsing]
  turn_loop.rs:815 — if tool_uses.is_empty() && has_tool_call_markers(&current_text_raw)
    → tool_parser::parse_tool_calls(&current_text_raw)  — crates/tui/src/core/tool_parser.rs
      (支援 [TOOL_CALL], <deepseek:tool_call>, <invoke>, <function_calls> 等格式)
```

**錯誤路徑**：
- Input buffer 無法解析為 JSON → log warn，使用初始 `{}` input → 發送 ToolCallStarted
- Stream 中斷且已收到 content → 不回傳，直接 fail turn（避免 double-bill）
- `transparent_stream_retries` 超過 `MAX_TRANSPARENT_STREAM_RETRIES (2)` → 放棄 retry

### 7. 權限判斷

**Allow/Deny/Ask 三值判定** — 在 tool 規劃階段 (turn_loop.rs:1154–1204)：

```
per tool call:
  if mode == AppMode::Plan && destructive tool → blocked_error = permission_denied
  if caller not allowed → blocked_error = permission_denied
  if tool_def.none() && not MCP → blocked_error = not_available
  if MCP tool → approval_required = !read_only
  if registry tool → approval_required = (spec.approval_requirement != Auto)
  if code_execution → approval_required = true

  [Loop guard check]
  if loop_guard.record_attempt(name, input) == Block → guard_result = error

Execution phase:
  if plan.approval_required == true → 進入 approval flow
    → 發送 Event::ApprovalRequired — turn_loop.rs:1606
    [ASYNC] → await_tool_approval(&tool_id) — approval.rs:67
      → rx_approval.recv() 阻塞等待
        → 用戶在 UI 選擇：
          • ApprovalDecision::Approved → 繼續執行
          • ApprovalDecision::Denied → return Err(permission_denied)
          • ApprovalDecision::RetryWithPolicy → 建立 elevated context 重試
        → 或 cancel_token.cancelled() → return Err(cancelled)

[ASYNC] await_tool_approval 是 blocking wait，resume point 是 rx_approval.recv() 返回
```

**錯誤路徑**：
- approval channel 關閉 → `Err(ToolError::execution_failed("channel closed"))`
- cancel 觸發 → `Err(ToolError::execution_failed("Request cancelled while awaiting approval"))`

### 8. 工具執行

**批次規劃** (turn_loop.rs:1258)：
```
→ plan_tool_execution_batches(plans)  — dispatch.rs
  讀取每個 plan 的:
    • supports_parallel (read-only tools)
    • approval_required (sequential)
    • blocked_error / guard_result (先處理)
  輸出: Vec<ToolExecutionBatch>
    • Parallel(Vec<plan>) — 唯讀且可並行
    • Serial(plan) — 需要 write lock 或需要審批
```

**執行入口** (turn_loop.rs:1287)：
```
for batch in batches:
  if parallel_allowed:
    → FuturesUnordered — 每個 plan 一個 async task
      → Engine::execute_tool_with_lock(lock, supports_parallel, ...) — tool_execution.rs:263
    → 所有 task 完成後收集 outcomes
  else:
    → 依序執行每個 plan：
      // 先處理 guard_result / blocked_error（不執行）
      // 處理 multi_tool_use.parallel → execute_parallel_tool — tool_execution.rs:154
      // 處理 code_execution / js_execution → 專用函數
      // 處理 request_user_input → await_user_input — approval.rs:105
      // 處理需要審批的工具 → approval flow → execute_tool_with_lock
      // 直接執行 → Engine::execute_tool_with_lock(lock, ...)
```

**[ASYNC] 並行工具鎖**：
```
tool_exec_lock (Arc<RwLock<()>>) — engine.rs:402
  if supports_parallel → ToolExecGuard::Read(lock.read().await)
  else → ToolExecGuard::Write(lock.write().await)
  確保所有 write tool 序列化，read-only tool 可並行
```

**InteractiveTerminalGuard** (tool_execution.rs:35)：
```
[ASYNC] → engage(tx, interactive) — tool_execution.rs:42
  發送 Event::PauseEvents → 等 ack (750ms timeout)
  工具完成/取消時 Drop → 發送 Event::ResumeEvents
```

**錯誤路徑**：
- 工具執行失敗 (`ToolError`) → 包裝為 `ToolResult { success: false }` → 記錄到 session
- 並行工具中有部分失敗 → 收集所有結果，不中斷其他並行工具
- Loop guard 檢測到重複調用 → `loop_guard_block_tool_result` — 不回傳 error 但標記 failed
- Post-edit 的 spillover (`apply_spillover_with_artifact`) 失敗 → 跳過（非致命）

### 9. LSP Diagnostics Injection

**介入點：在工具執行成功之後、session 更新之前**

```
turn_loop.rs:1805 — 每個工具成功執行後：
  if output.success && tool_was_executed:
    → run_post_edit_lsp_hook(&outcome.name, &tool_input)  — lsp_hooks.rs:80
      [ASYNC] 提取 edited path，呼叫 LSP manager：
      → self.lsp_manager.diagnostics_for(&absolute, seq)
        → diagnostics 結果存入 self.pending_lsp_blocks

turn_loop.rs:212 — 下次 API 請求之前（next step 或下一個 turn）：
  → flush_pending_lsp_diagnostics()  — lsp_hooks.rs:110
    → 將 pending_lsp_blocks 渲染為合成 user message
    → 調用 add_session_message 插入到 messages
```

**LSP timeout**：LSP 內部有獨立 timeout（`crates/tui/src/lsp/`），但 engine 層不 await LSP 結果——它是 fire-and-forget 通過 channel 到 LSP manager。

**錯誤路徑**：
- LSP server 未配置或禁用 → 跳過 (`lsp_hooks.rs:85-87`)
- LSP diagnostics 為空 → 跳過 (`lsp_hooks.rs:111`)
- diagnostics_for 失敗 → 靜默吞掉錯誤 (doc: "Failure is silent by design")

### 10. Sub-agent

**Spawn 路徑 — 僅追蹤本輪的 spawn**：

```
turn_loop.rs:903 — 收到 sub-agent completion 時：
  while let Ok(c) = self.rx_subagent_completion.try_recv() {
    completions.push(c);
  }
  
  [如果子 agent 仍在運行，turn 保持開啟]
  → should_hold_turn_for_subagents(completions.len(), running) — turn_loop.rs:1964
    → true if completions == 0 && running > 0

  [ASYNC] tokio::select! {
    cancel_token.cancelled() → 中斷
    rx_subagent_completion.recv() → 收到 completion sentinel
    rx_steer.recv() → 收到用戶 steer 輸入
  }

  收到 completion 後：
  → subagent_completion_runtime_message(&c.payload) — turn_loop.rs:1946
  → add_session_message (插入 sentinel 到 transcript)
  → turn.next_step();
  → continue; (回到 loop 頂部，讓 model 看到 sentinel)
```

**Sub-agent spawn**（透過 `Op::SpawnSubAgent`）：
```
engine.rs:626 — Op::SpawnSubAgent handler:
  → SubAgentRuntime::new(client, model, tool_context, ...)
    .with_role_models(...)
    .background_runtime()
  → resolve_subagent_assignment_route(&runtime, None, &prompt) — 分配 model
  → manager.spawn_background(manager, runtime, SubAgentType::General, prompt, None) 
    → 進入 sub-agent 獨立循環（不追入）
```

**Parent 接收 sentinel 的點**：`self.rx_subagent_completion.recv()` (turn_loop.rs:929)

### 11. Session Save

**觸發時機**：

1. **Checkpoint**（crash recovery）：在 `dispatch_user_message` 中發送請求前
   - `crates/tui/src/tui/ui.rs:3915` — `persist(Checkpoint(session))`
   - 異步寫入 `~/.deepseek/sessions/<id>/checkpoints/latest.json`

2. **Session Snapshot**（持久化保存）：在 TurnComplete 事件處理後
   - `crates/tui/src/tui/ui.rs:1410` — `persist(SessionSnapshot(session))`
   - `crates/tui/src/tui/ui.rs:4377` — `persist(SessionSnapshot(session))`
   - 異步寫入 `~/.deepseek/sessions/<id>/<id>.json`

**保存內容** (`SavedSession`) — `crates/tui/src/session_manager.rs`：
- Session metadata（id, model, created_at, workspace, etc.）
- Messages（最多 `MAX_PERSISTED_MESSAGES = 500`）
- System prompt
- Usage statistics
- Artifact records
- Artifact registry paths

**Persistence Actor** (`crates/tui/src/tui/persistence_actor.rs:100`)：
```
[ASYNC BOUNDARY] — 專用 tokio task
  使用 unbounded channel (try_send 永不阻塞)
  Latest-wins coalescing: 同一類型的多個請求只寫最後一個
  → save_checkpoint(session)  — session_manager.rs:254
  → save_session(session)  — session_manager.rs:238
    → serde_json::to_string_pretty
    → write_atomic (NamedTempFile + fsync + rename)
    → cleanup_old_sessions (keep MAX_SESSIONS = 50)
```

### 12. 下一輪狀態更新

在 `handle_send_message` 結束後 (engine.rs:879-1149)，以下 session 字段已被更新：

```rust
// engine.rs 內部
self.session.messages          // 添加了 user message + assistant messages + tool results
self.session.model             // 可能被 auto_router 更新
self.session.system_prompt     // refresh_system_prompt 可能更新
self.session.reasoning_effort  // 從 Op 參數設置
self.session.auto_model        // 同上
self.session.allow_shell       // 同上
self.session.trust_mode        // 同上
self.session.auto_approve      // 同上
self.session.approval_mode     // 同上
self.session.total_usage       // += turn.usage
self.session.compaction_summary_prompt  // 如果 auto-compaction 發生則更新
self.config.model              // 同步 session.model
self.config.goal_objective     // 從 Op 設置
self.turn_counter              // += 1
self.capacity_controller       // mark_turn_start / mark_turn_end
```

UI 側的狀態更新 (`dispatch_user_message` 返回後)：
```
app.is_loading = true    → 設為 false (on TurnComplete)
app.api_messages.push(...)
app.history.push(HistoryCell::User)
app.dispatch_started_at = Some(now)
app.last_send_at = Some(now)
```

**Event-driven 更新**：engine 通過 `tx_event` 發送 `Event`，UI 在 `event_loop` 中處理：
- `Event::TurnStarted` (engine.rs:908)
- `Event::MessageStarted` / `MessageDelta` / `MessageComplete`
- `Event::ToolCallStarted` / `ToolCallComplete`
- `Event::TurnComplete { usage, status, error }` (engine.rs:1128)

---

## 任務 C：Snapshot Rollback 完整生命週期

### 概述

Snapshot 使用 side git repo，位置 `~/.deepseek/snapshots/<project_hash>/<worktree_hash>/.git`
完全隔離於用戶自己的 git 倉庫。每個 snapshot 是一個 git commit，label 格式 `pre-turn:N`、`post-turn:N`、`tool:<call_id>`。

### Pre-turn Snapshot 觸發

```
handle_send_message — engine.rs:918–926

if self.config.snapshots_enabled:
  [ASYNC] tokio::task::spawn_blocking(||
    pre_turn_snapshot(&workspace, turn_seq, cap_bytes))
    → crates/tui/src/core/turn.rs:139
      → snapshot_with_label(workspace, "pre-turn:{seq}", cap_bytes)  — turn.rs:161
        → SnapshotRepo::open_or_init_with_cap(workspace, cap_bytes)  — snapshot/repo.rs:205
          [首次 OR 後續打開]
          → git init --quiet <parent>               (首次)
          → git config user.name "deepseek-snapshots"
          → git config user.email "snapshots@deepseek-tui.local"
          → git config gc.auto 0
          → write_builtin_excludes(&git_dir)       (寫入 .git/info/exclude)
          → cleanup_stale_pack_temps(&git_dir)     (清理中斷的 pack 操作)
        → repo.snapshot("pre-turn:{seq}")  — snapshot/repo.rs:304
          [容量檢查] 如果 git_dir > MAX_SNAPSHOT_SIZE_MB (500MB) → 激進 prune
          → git add -A
          → git write-tree
          → git commit-tree -m "pre-turn:{seq}" [-p HEAD]
          → git update-ref HEAD <sha>
          → repo.prune_keep_last_n(MAX_SNAPSHOTS=50)  — 刪除最舊的
        → return SnapshotId(sha)
  .await
```

**錯誤路徑**：
- `spawn_blocking` 失敗 → 非致命，log WARN（`# ensure non-fatal`）
- `SnapshotRepo::open_or_init_with_cap` 失敗（git 缺失、workspace 過大）→ log WARN, return None
- `snapshot()` 中任一步驟失敗 → log WARN, return None
- `prune_keep_last_n` 失敗 → log WARN（非致命）

### Post-turn Snapshot 觸發

```
handle_send_message 末尾 — engine.rs:1141–1148

if self.config.snapshots_enabled:
  → spawn_blocking_supervised("post-turn-snapshot", move || {
      post_turn_snapshot(&post_workspace, post_seq, post_cap)
    })
    → turn.rs:157 → snapshot_with_label(workspace, "post-turn:{seq}", cap_bytes)
      （同 pre-turn 流程，label 不同，fire-and-forget）
```

**關鍵差異**：post-turn 使用 `spawn_blocking_supervised`（fire-and-forget），不 `await` 結果。TurnComplete 已經發出，UI 已可操作。

### Pre-tool Snapshot（surgical undo）

```
turn_loop.rs:1662–1678 — 在執行 file-modifying tool 之前：

if result_override.is_none()
  && matches!(tool_name, "write_file" | "edit_file" | "apply_patch"):
  [ASYNC] spawn_blocking(||
    pre_tool_snapshot(&ws, &tool_id, cap_bytes))
    → turn.rs:151
      → snapshot_with_label(workspace, "tool:{call_id}", cap_bytes)
```

### Restore 完整路徑

```
用戶輸入 /restore N
  → crates/tui/src/commands/restore.rs:17  — restore(app, arg)
    → SnapshotRepo::open_or_init(&workspace)  — restore.rs:19
    → repo.list(LIST_LIMIT=10)  — restore.rs:29  → snapshot/repo.rs:505
      → git log --oneline -n <limit> --format="%H|%s|%ct"
    → 解析 args，檢查信任模式 (trust_mode || yolo)
    → repo.restore(&snapshots[n-1].id)  — restore.rs:73  → snapshot/repo.rs:418
      → tree_paths("HEAD") — 當前工作樹文件列表
      → tree_paths(target_sha) — 目標 snapshot 文件列表
      → git checkout <sha> -- :/
      → remove_paths_missing_from_target(current, target)  — snapshot/repo.rs:469
        （刪除 snapshot 中不存在的文件）
    → 返回 CommandResult::message("Restored snapshot #N ...")
```

**錯誤路徑**：
- `SnapshotRepo::open_or_init` 失敗 → `CommandResult::error("Snapshot repo unavailable...")`
- `repo.list` 失敗 → `CommandResult::error("Failed to list snapshots...")`
- `repo.restore` 失敗 → `CommandResult::error("Restore failed: {e}")`
- 非信任模式 → `CommandResult::message("Refusing to restore...")`
- N 超出範圍 → `CommandResult::error("Only X snapshot(s) available...")`
- 無 snapshot → `CommandResult::message("No snapshots yet...")`

### Cap/Prune

**觸發條件與邏輯**：

1. **Per-snapshot pruning** (turn.rs:172)：
   ```
   每次 snapshot 後：
   → repo.prune_keep_last_n(DEFAULT_MAX_SNAPSHOTS=50)  — snapshot/repo.rs:636
     → git log --format="%H" 取得所有 commits
     → 若 > max_count，刪除最舊的
     → git update-ref -d refs/heads/master <old_sha>
     → prune_unreachable_objects() — git gc --prune=now
   ```

2. **Size-based pruning** (snapshot/repo.rs:307)：
   ```
   snapshot() 內部：
   if dir_size_mb(git_dir) > MAX_SNAPSHOT_SIZE_MB (500MB):
     → 逐步縮小 retention age (1s → 0.1s step)
     → 直到 dir_size ≤ PRUNE_TARGET_MB (400MB)
     → 如果仍超標 → 清空所有 refs
   ```

3. **Boot-time pruning** (prune.rs:21)：
   ```
   每次 session 啟動：
   → prune_older_than(workspace, DEFAULT_MAX_AGE=7 days)  — snapshot/prune.rs:21
     → repo.prune_older_than(max_age)  — snapshot/repo.rs:549
     → repo.prune_unreachable_objects() — git gc --prune=now
   ```

**錯誤路徑（全部非致命）**：
- `prune_keep_last_n` 失敗 → `tracing::warn`
- `prune_older_than` 失敗 → 返回 `Ok(0)`（無 snapshot repo）
- `prune_unreachable_objects` 失敗 → 忽略

---

## 驗證命令

以下 grep 命令驗證報告中引用的關鍵函數存在於對應文件中：

### Crate 依賴圖驗證（5 個關鍵點）

```bash
# CLI binary 入口
grep -n "fn main" crates/cli/src/main.rs

# TUI binary 入口
grep -n "async fn main" crates/tui/src/main.rs

# TUI engine 入口
grep -n "fn run\b" crates/tui/src/core/engine.rs

# leaf crate 驗證：無內部依賴
grep -c "^deepseek-" crates/protocol/Cargo.toml
grep -c "^deepseek-" crates/secrets/Cargo.toml
grep -c "^deepseek-" crates/mcp/Cargo.toml

# app-server 依賴 core
grep "deepseek-core" crates/app-server/Cargo.toml
```

### 單輪 Turn 驗證（5 個關鍵點）

```bash
# dispatch_user_message (TUI → engine)
grep -n "fn dispatch_user_message" crates/tui/src/tui/ui.rs

# engine 主 event loop 接收 Op::SendMessage
grep -n "Op::SendMessage" crates/tui/src/core/engine.rs

# handle_deepseek_turn
grep -n "fn handle_deepseek_turn" crates/tui/src/core/engine/turn_loop.rs

# create_message_stream → HTTP 請求
grep -n "fn create_message_stream" crates/tui/src/client.rs

# auto_router 入口
grep -n "fn resolve_auto_model_selection" crates/tui/src/tui/auto_router.rs

# await_tool_approval (權限判斷)
grep -n "fn await_tool_approval" crates/tui/src/core/engine/approval.rs

# execute_tool_with_lock (工具執行)
grep -n "fn execute_tool_with_lock" crates/tui/src/core/engine/tool_execution.rs

# flush_pending_lsp_diagnostics
grep -n "fn flush_pending_lsp_diagnostics" crates/tui/src/core/engine/lsp_hooks.rs

# subagent completion handoff
grep -n "fn subagent_completion_runtime_message" crates/tui/src/core/engine/turn_loop.rs

# persistence actor spawn
grep -n "fn spawn_persistence_actor" crates/tui/src/tui/persistence_actor.rs
```

### Snapshot 驗證（5 個關鍵點）

```bash
# pre_turn_snapshot
grep -n "fn pre_turn_snapshot" crates/tui/src/core/turn.rs

# post_turn_snapshot
grep -n "fn post_turn_snapshot" crates/tui/src/core/turn.rs

# SnapshotRepo::snapshot (git operations)
grep -n "fn snapshot\b" crates/tui/src/snapshot/repo.rs

# SnapshotRepo::restore
grep -n "fn restore" crates/tui/src/snapshot/repo.rs

# restore command entry
grep -n "fn restore\b" crates/tui/src/commands/restore.rs

# prune_keep_last_n
grep -n "fn prune_keep_last_n" crates/tui/src/snapshot/repo.rs

# boot-time prune
grep -n "fn prune_older_than" crates/tui/src/snapshot/prune.rs
```
