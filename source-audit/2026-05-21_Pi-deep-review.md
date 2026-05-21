# Pi — 深層架構審計報告

**日期：** 2026-05-21
**版本：** 0.75.4
**Repo：** `/Users/lesprivilege/Projects/pi/`

---

## 任務 A：項目結構全景

### A.1 Package 一覽

| Package | npm name | 角色 | 是否 leaf |
|---------|----------|------|-----------|
| `packages/tui` | `@earendil-works/pi-tui` | Terminal UI library，differential rendering | 是 |
| `packages/ai` | `@earendil-works/pi-ai` | Unified multi-provider LLM API，模型發現與 provider 配置 | 是 |
| `packages/agent` | `@earendil-works/pi-agent-core` | Agent runtime — loop、tool call、state management、session tree、compaction、harness | 否（依賴 pi-ai） |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | 互動式 coding agent CLI，整合所有下層 package，提供 extension 系統、mode 層、TUI | 否（依賴 agent、ai、tui） |

### A.2 依賴關係

```
pi-tui (leaf)
pi-ai  (leaf)
  ^
  |
pi-agent-core  ──── 依賴 pi-ai
  ^
  |
pi-coding-agent  ── 依賴 pi-agent-core、pi-ai、pi-tui
```

### A.3 各 Package 核心文件與職責

#### packages/tui/
| 文件路徑 | 職責 |
|---------|------|
| `src/tui.ts` | TUI 核心：Container、Component 樹、differential rendering、overlay |
| `src/terminal.ts` | Terminal 抽象層（ProcessTerminal） |
| `src/keys.ts` | 按鍵解析（kitty protocol） |
| `src/keybindings.ts` | Keybindings 管理 |
| `src/editor-component.ts` | Editor 元件介面 |
| `src/components/` | 各 UI 元件：box、input、editor、markdown、select-list、settings-list、loader、image、text |
| `src/stdin-buffer.ts` | Stdin 緩衝區，批次分割 |
| `src/terminal-image.ts` | Kitty/iTerm2 圖片協定支援 |

#### packages/ai/
| 文件路徑 | 職責 |
|---------|------|
| `src/types.ts` | 核心型別：Model、Message、Context、Tool、Provider、StreamOptions、AssistantMessageEvent |
| `src/api-registry.ts` | API provider 註冊表 — `registerApiProvider()`、`getApiProvider()` |
| `src/stream.ts` | `stream()`、`streamSimple()`、`complete()`、`completeSimple()` — 對外 API 調用入口 |
| `src/models.ts` | Model 定義（自動生成的 provider/model 列表） |
| `src/models.generated.ts` | 自動生成的所有已知 model 資料 |
| `src/env-api-keys.ts` | 環境變數 API key 讀取 |
| `src/session-resources.ts` | 跨 session 的 provider resource 管理 |
| `src/oauth.ts` | OAuth 登入流程 |
| `src/cli.ts` | pi-ai CLI entry |
| `src/providers/register-builtins.ts` | 註冊所有內建 API provider — lazy-load 包裝 |
| `src/providers/anthropic.ts` | Anthropic Messages API provider |
| `src/providers/openai-completions.ts` | OpenAI Chat Completions API provider |
| `src/providers/openai-responses.ts` | OpenAI Responses API provider |
| `src/providers/openai-codex-responses.ts` | OpenAI Codex Responses API provider |
| `src/providers/amazon-bedrock.ts` | AWS Bedrock provider |
| `src/providers/google.ts` | Google Generative AI provider |
| `src/providers/google-vertex.ts` | Google Vertex AI provider |
| `src/providers/mistral.ts` | Mistral provider |
| `src/providers/cloudflare.ts` | Cloudflare Workers AI / AI Gateway provider |
| `src/providers/github-copilot-headers.ts` | GitHub Copilot 認證 header |
| `src/providers/azure-openai-responses.ts` | Azure OpenAI Responses provider |
| `src/providers/simple-options.ts` | SimpleStreamOptions 標準化 |
| `src/providers/transform-messages.ts` | 跨 provider 的 message 轉換邏輯 |
| `src/providers/openai-prompt-cache.ts` | OpenAI 系 prompt cache 標記 |
| `src/providers/faux.ts` | Faux provider（測試用） |
| `src/providers/images/` | 圖片生成 provider（openrouter） |
| `src/utils/` | 工具函數：event-stream、oauth、diagnostics、hash、json-parse、overflow 等 |

#### packages/agent/
| 文件路徑 | 職責 |
|---------|------|
| `src/agent.ts` | **Agent class** — stateful wrapper，lifecycle events，queue 管理，streaming 狀態 |
| `src/agent-loop.ts` | **Runtime loop 核心** — `runLoop()`、`streamAssistantResponse()`、`executeToolCalls()` |
| `src/types.ts` | Agent 核心型別：AgentTool、AgentState、AgentEvent、AgentLoopConfig、StreamFn |
| `src/harness/agent-harness.ts` | **AgentHarness** — higher-level harness，整合 session、turn state、hook system |
| `src/harness/types.ts` | Harness 型別：Session、ExecutionEnv、Skill、PromptTemplate、AgentHarnessOptions |
| `src/harness/messages.ts` | 自訂訊息型別（bashExecution、custom、branchSummary、compactionSummary）與 `convertToLlm()` |
| `src/harness/system-prompt.ts` | `formatSkillsForSystemPrompt()` — XML skills 區塊生成 |
| `src/harness/skills.ts` | Skill 載入引擎（SKILL.md 發現、ignore 規則、frontmatter 解析） |
| `src/harness/prompt-templates.ts` | Prompt template 格式化 |
| `src/harness/session/session.ts` | **Session class** — session tree 的 entry append、buildContext、moveTo（branch） |
| `src/harness/session/jsonl-storage.ts` | JSONL 儲存實作 |
| `src/harness/session/jsonl-repo.ts` | JSONL session repo（create、open、fork、list） |
| `src/harness/session/memory-repo.ts` | In-memory session repo（測試用） |
| `src/harness/compaction/compaction.ts` | Compaction engine — `prepareCompaction()`、`compact()`、`generateSummary()` |
| `src/harness/compaction/branch-summarization.ts` | Branch summarization engine |
| `src/harness/env/nodejs.ts` | Node.js ExecutionEnv 實作 |
| `src/proxy.ts` | Proxy 設定 |

#### packages/coding-agent/
| 文件路徑 | 職責 |
|---------|------|
| `src/cli.ts` | **CLI 入口** — shebang、process title、`main()` 調用 |
| `src/main.ts` | **主入口** — 參數解析、session 建立、mode dispatch（interactive/print/rpc） |
| `src/core/agent-session.ts` | **AgentSession** — 核心 abstraction：事件訂閱、persistence、auto-retry、compaction、model 管理 |
| `src/core/agent-session-runtime.ts` | **Runtime** — create/wire AgentSession + Agent + extensions + tools |
| `src/core/agent-session-services.ts` | **Services factory** — SettingsManager、ModelRegistry、ResourceLoader 初始化 |
| `src/core/sdk.ts` | **SDK** — `createAgentSession()`、`createAgentSessionFromServices()` + tool factories |
| `src/core/session-manager.ts` | **SessionManager** — session 檔案 CRUD、tree 遍歷、migration、fork |
| `src/core/system-prompt.ts` | `buildSystemPrompt()` — 完整 system prompt 生成 |
| `src/core/skills.ts` | Skill 載入（coding-agent 版，含 frontmatter、ignore、source tracking） |
| `src/core/extensions/types.ts` | **Extension 系統型別** — ExtensionAPI、ToolDefinition、extension events |
| `src/core/extensions/loader.ts` | **Extension loader** — jiti-based TS/JS 模組載入、discovery、virtual modules |
| `src/core/extensions/runner.ts` | **Extension runner** — event dispatch、handler chain、keybinding conflict |
| `src/core/extensions/wrapper.ts` | Tool wrapper — registered tool → AgentTool 轉換 |
| `src/core/tools/` | **7 個內建工具** — read、bash、edit、write、grep、find、ls |
| `src/core/model-registry.ts` | **ModelRegistry** — provider/model CRUD、API key 解析 |
| `src/core/model-resolver.ts` | **Model resolver** — CLI model 解析、scoped models 解析 |
| `src/core/resource-loader.ts` | **ResourceLoader** — 載入 skills、prompts、themes、context files、extensions |
| `src/core/settings-manager.ts` | **SettingsManager** — JSON/YAML 設定管理 |
| `src/core/event-bus.ts` | 跨 extension 的 EventBus |
| `src/core/bash-executor.ts` | Bash 執行器 |
| `src/core/compaction/` | Compaction（coding-agent 層，包裝 agent 層的功能） |
| `src/core/auth-storage.ts` | API key / OAuth 憑證儲存 |
| `src/modes/interactive/interactive-mode.ts` | **InteractiveMode** — TUI rendering、use r input、slash commands |
| `src/modes/print-mode.ts` | **Print mode** — 一次性 prompt → stdout |
| `src/modes/rpc/` | **RPC mode** — JSON-RPC over stdin/stdout |

### A.4 關鍵位置標示

| 元件 | 位置 |
|------|------|
| CLI 入口 | `packages/coding-agent/src/cli.ts:1` |
| 主入口 | `packages/coding-agent/src/main.ts:425` `main()` |
| Runtime loop 主文件 | `packages/agent/src/agent-loop.ts:155` `runLoop()` |
| Agent class | `packages/agent/src/agent.ts:166` `Agent` |
| Context/memory 管理 | `packages/agent/src/harness/session/session.ts:78` `Session` |
| Tool dispatch | `packages/agent/src/agent-loop.ts:373` `executeToolCalls()` |
| Provider 抽象層 | `packages/ai/src/api-registry.ts` `ApiProvider` + `packages/ai/src/stream.ts` `streamSimple()` |
| Session 管理 | `packages/coding-agent/src/core/session-manager.ts` `SessionManager` |
| System prompt 構建 | `packages/coding-agent/src/core/system-prompt.ts:28` `buildSystemPrompt()` |
| Extension 系統主文件 | `packages/coding-agent/src/core/extensions/` |

---

## 任務 B：Runtime Loop 完整追蹤

### B.1 Loop 入口

Pi 的 agent loop 是 **imperative while-loop**（非 recursive、非 event-driven core loop），以雙層巢狀結構實作：

```
main() (coding-agent/src/main.ts:425)
  └── interactiveMode.run() (coding-agent/src/modes/interactive/interactive-mode.ts)
       or runPrintMode() / runRpcMode()
       └── AgentSession.prompt() (coding-agent/src/core/agent-session.ts:961)
            └── Agent.prompt() (agent/src/agent.ts:327)
                 └── runAgentLoop() (agent/src/agent-loop.ts:95)
                      └── runLoop() (agent/src/agent-loop.ts:155)
```

**`runLoop()`** (`agent/src/agent-loop.ts:155-269`) 是唯一的低階 loop 實作。結構：

```
while (true)  // 外層：處理 follow-up messages
  while (hasMoreToolCalls || pendingMessages.length > 0)  // 內層：處理 tool calls + steering
    1. 注入 pending messages
    2. streamAssistantResponse() → LLM 調用
    3. 檢查 stopReason === "error" | "aborted" → return
    4. 提取 tool calls
    5. executeToolCalls() → 非同步執行
    6. prepareNextTurn hook → context/model/thinking 更新
    7. shouldStopAfterTurn hook → 提早退出
    8. 從 steering queue 取出 pending messages
  // 外層續：從 follow-up queue 取出 messages，繼續外層 while
```

**錯誤路徑：** `streamAssistantResponse()` 返回 AssistantMessage 的 stopReason 為 "error" 或 "aborted" 時，loop 立即 emit `turn_end` + `agent_end` 並 return。錯誤不會 throw；而是編碼在回傳的 message 中。

### B.2 Context 組裝

#### System Prompt

有兩個層級的 system prompt 構建：

1. **底層 AgentHarness** (`agent/src/harness/agent-harness.ts:321-333`)：
   - `systemPrompt` 可以是 string 或 callback `(context) => string`
   - callback 接收 env、session、model、thinkingLevel、activeTools、resources
   - 預設值是 "You are a helpful assistant."

2. **高層 AgentSession** (`coding-agent/src/core/agent-session.ts:877-911` `_rebuildSystemPrompt()`)：
   - 呼叫 `buildSystemPrompt()` (`coding-agent/src/core/system-prompt.ts:28`)
   - 組裝：custom prompt → tools list → guidelines → 專案 context files → skills → 日期/CWD
   - toolPromptSnippets 為每個 active tool 生成一行摘要
   - context files 從 `ResourceLoader.getAgentsFiles()` 取得
   - skills 從 `ResourceLoader.getSkills()` 取得
   - 結果透過 `agent.state.systemPrompt` 設定

#### History/Messages 管理

Messages 儲存在三個地方：

1. **Agent.state.messages** (`agent/src/agent.ts:82-87`) — 目前 run 的 transcript，Array of AgentMessage
2. **Harness turn state** (`agent/src/harness/agent-harness.ts:152-162`) — `AgentHarnessTurnState.messages`
3. **Session tree** (`coding-agent/src/core/session-manager.ts`) — JSONL 檔案，每行一個 entry，含 parentId/parentId 形成 linked-list tree

##### buildSessionContext() (`coding-agent/src/core/session-manager.ts:314`)
- 從 leaf 走 parentId chain 到 root
- 遍歷 path，收集 message / custom_message / branch_summary / compaction 型別 entry
- 如果是 compaction entry：先插入 compaction summary message，再跳過 compacted 部分
- 回傳 flat AgentMessage[] 陣列

#### Tool Specs 注入

- `AgentLoopConfig.model` 帶有 tool 定義 (`Tool[]`)
- Tool 注入發生在 `agent-loop.ts:292-296`，建立 `llmContext: Context` 時傳入 `context.tools`
- `context.tools` 來自 `AgentContext.tools` / `AgentHarnessTurnState.activeTools`

#### 動態 Context（memory, skills, MCP）

- **Skills**：透過 `formatSkillsForPrompt()` 加入 system prompt 尾部的 `<available_skills>` XML 區塊
- **Context files**：由 `ResourceLoader.loadProjectContextFiles()` 載入，加入 system prompt 的 `<project_context>` 區塊
- **Extension hooks**：`config.transformContext` + `config.beforeToolCall` + `config.afterToolCall` — extensions 可在 context 被發送到 LLM 前修改 messages
- **MCP**：無原生 MCP 支援 — MCP servers 必須寫成 extension（調用 MCP SDK）

#### 最終發送到 API 的格式

`agent-loop.ts:275-368` `streamAssistantResponse()`：
1. `config.transformContext(messages)` → AgentMessage[]（選用）
2. `config.convertToLlm(messages)` → Message[]（AgentMessage → {user, assistant, toolResult}）
3. 建立 `llmContext: { systemPrompt, messages, tools }`
4. 呼叫 `streamFn(model, llmContext, options)`

**`convertToLlm()`** (`agent/src/harness/messages.ts:120-164`) 負責自訂訊息型別的轉換：
- bashExecution → user（文字化 + 指令/輸出/exit code）
- custom → user
- branchSummary → user（包在 `<summary>` 標籤中）
- compactionSummary → user（包在 `<summary>` 標籤中）
- user/assistant/toolResult → 直接保留
- 未知 role → 過濾掉（return undefined）

### B.3 API 調用

#### Provider 抽象層

`packages/ai/src` 的設計：

```
api-registry.ts:  Map<api, { stream, streamSimple }>
stream.ts:         streamSimple(model, context, options) → AssistantMessageEventStream
```

- **`registerApiProvider()`** (`ai/src/api-registry.ts:66`) 註冊 provider，key 為 API 類型字串
- **`streamSimple()`** (`ai/src/stream.ts:43`) 查找 provider 並呼叫 `provider.streamSimple()`
- 每個 provider module 實作自己的 `stream()` 和 `streamSimple()`
- Provider modules 是 **lazy-loaded**（`register-builtins.ts` 用 `createLazyStream` 包裝）

#### 支援的 API 類型（已知）

| API 名稱 | Provider 文件 |
|----------|--------------|
| `anthropic-messages` | `providers/anthropic.ts` |
| `openai-completions` | `providers/openai-completions.ts` |
| `openai-responses` | `providers/openai-responses.ts` |
| `azure-openai-responses` | `providers/azure-openai-responses.ts` |
| `openai-codex-responses` | `providers/openai-codex-responses.ts` |
| `google-generative-ai` | `providers/google.ts` |
| `google-vertex` | `providers/google-vertex.ts` |
| `bedrock-converse-stream` | `providers/amazon-bedrock.ts` |
| `mistral-conversations` | `providers/mistral.ts` |

#### Request 構建

`agent/src/agent-loop.ts:275-368` `streamAssistantResponse()`：
1. transformContext → convertToLlm → 組裝 Context
2. 解析 API key：`getApiKey(config.model.provider) || config.apiKey`
3. 調用 `streamFn(model, llmContext, { ...config, apiKey, signal })`
4. 回傳 `AssistantMessageEventStream`

#### Streaming Response Parsing

事件流 protocol（`ai/src/types.ts:347-359` `AssistantMessageEvent`）：
```
start → text_start / text_delta / text_end
       → toolcall_start / toolcall_delta / toolcall_end
       → thinking_start / thinking_delta / thinking_end
       → done (正常終止) | error (錯誤終止)
```

`streamAssistantResponse()` (`agent-loop.ts:313-367`) 遍歷事件流，根據 event type 更新 `partialMessage` 並 emit 對應的 `AgentEvent`（`message_start` / `message_update` / `message_end`）。

### B.4 Tool Call 處理

#### 從 Response 提取 Tool Calls

`agent-loop.ts:203`：
```typescript
const toolCalls = message.content.filter((c) => c.type === "toolCall");
```

#### Tool Dispatch

`executeToolCalls()` (`agent-loop.ts:373-388`)：
1. 檢查是否有 tool 的 `executionMode === "sequential"`
2. 如果 config.toolExecution === "sequential" 或存在 sequential tool → `executeToolCallsSequential()`
3. 否則 → `executeToolCallsParallel()`

**執行模式：**
- `"sequential"` (`agent-loop.ts:395-449`)：逐一 prepare → validate → execute → finalize
- `"parallel"` (`agent-loop.ts:451-516`)：sequentially prepare，then `Promise.all()` 並行執行；結果依原始順序回傳

#### Prepare 階段

`prepareToolCall()` (`agent-loop.ts:562-626`)：
1. 查找 tool by name（在 `currentContext.tools` 中）
2. 如果找不到 → 回傳 `createErrorToolResult("Tool ${name} not found")`
3. `prepareToolCallArguments()` — 選用的 argument shim
4. `validateToolArguments()` — schema 驗證（來自 pi-ai）
5. `beforeToolCall` hook — 可返回 `{ block: true }` 阻止執行
6. 錯誤 → 回傳 ImmediateToolCallOutcome（不執行）

#### Tool Execute

`executePreparedToolCall()` (`agent-loop.ts:628-663`)：
1. 呼叫 `prepared.tool.execute(toolCallId, args, signal, onUpdate)`
2. `onUpdate` callback 發出 `tool_execution_update` 事件
3. 錯誤 → 捕獲並回傳 error ToolResult

#### Finalize 階段

`finalizeExecutedToolCall()` (`agent-loop.ts:665-708`)：
1. `afterToolCall` hook — 可 override content、details、isError、terminate
2. 回傳 `FinalizedToolCallOutcome`

#### 結果回注 Context

1. `createToolResultMessage()` (`agent-loop.ts:727-737`) — 從 finalized 建立 `ToolResultMessage`
2. 加入 `currentContext.messages` 與 `newMessages`
3. Emit `message_start` / `message_end` 事件

### B.5 Context Compaction

#### 觸發條件

兩處觸發：

1. **`AgentHarness.compact()`** (`agent/src/harness/agent-harness.ts:683-735`) — 手動調用
2. **`AgentSession` auto-compaction** (`coding-agent/src/core/agent-session.ts`) — 在 `_handlePostAgentRun()` 中檢查

`shouldCompact()` (`agent/src/harness/compaction/compaction.ts:196-199`)：
```typescript
如果 !settings.enabled → false
如果 contextTokens > contextWindow - reserveTokens → true
// 預設：enabled=true, reserveTokens=16384, keepRecentTokens=20000
```

#### 壓縮策略

`compact()` (`agent/src/harness/compaction/compaction.ts:626-705`)：

1. **`prepareCompaction()`** (`compaction.ts:541-606`)：
   - 找到上次 compaction 的邊界
   - 用 `findCutPoint()` 決定 cut point（保留約 keepRecentTokens 的近期 token）
   - 處理 turn splitting（cut 落在 turn 中間時 split summarization）
   - 提取 file operations（read/written/edited files）

2. **`generateSummary()`** (`compaction.ts:455-518`)：
   - 如果 `previousSummary` 存在 → 使用 `UPDATE_SUMMARIZATION_PROMPT`（增量更新）
   - 否則 → 使用 `SUMMARIZATION_PROMPT`（全新摘要）
   - 調用 `completeSimple()` 進行 LLM 摘要生成
   - 支援 thinking/reasoning（如果 model 支援）

3. **Split turn handling** (`compaction.ts:652-678`)：
   - 同時進行 history summarization 和 turn prefix summarization（`Promise.all`）
   - 合併為單一 summary

4. **File operations 附錄** (`compaction.ts:696-697`)：
   - 提取 `readFiles` 和 `modifiedFiles` 加入 summary 末尾

5. **結果儲存**：寫入 session 為 `compaction` type entry

#### 壓縮後對 Cache 的影響

Compaction 後，context 被替換為 summarization message（role: `compactionSummary`）加上保留的近期 messages。下次 API 調用會使用新的 context，provider cache 可能失效（因為 prompt 已改變）。`sessionId` 持續傳遞給 provider 以優化 session-aware caching。

### B.6 Session Tree

#### 數據結構

Session 是 **tree，以 JSONL 檔案儲存**。每個 entry 有：
- `id`（uuidv7 或 8-char hash）
- `parentId` | `null`
- `type` + 型別特定欄位

tree 實際上是以 linked list 結構表達的 tree（parentId 指向 parent node）。`leafId` 記錄目前位置。

**Entry types**（`coding-agent/src/core/session-manager.ts:137-146`）：
- `message` — LLM user/assistant/toolResult message
- `thinking_level_change` — thinking level 變更記錄
- `model_change` — model 變更記錄
- `compaction` — compaction entry（含 summary）
- `branch_summary` — branch 摘要
- `custom` — extension 專用資料（不進 context）
- `custom_message` — extension injected message（進 context）
- `label` — 使用者標記
- `session_info` — session 名稱等 metadata

#### Branching 邏輯

- **自動分支**：每個新 entry 的 parentId 設為目前 leaf，形成線性擴展
- **分支切換**：`Session.moveTo(entryId)` → 將 leaf 設為目標 entryId，並可選擇性寫入 branch_summary entry
- **Fork**：`SessionRepo.fork()` → 複製 session 檔案到新路徑
- **`navigateTree()`** (`agent/src/harness/agent-harness.ts:737-835`)：
  1. 收集從目前 leaf 到目標 entry 的 entries
  2. 找到 common ancestor
  3. 可選：用 LLM 生成 branch summary
  4. 呼叫 `session.moveTo()`

#### 持久化方式

JSONL 格式每行一個 JSON entry：
```
{"type":"session","id":"...","timestamp":"...","cwd":"...","version":3}
{"type":"message","id":"...","parentId":"...","timestamp":"...","message":{...}}
{"type":"compaction","id":"...","parentId":"...","summary":"...","firstKeptEntryId":"...","tokensBefore":1234}
```

支援版本遷移（v1 → v2 → v3，`migrateToCurrentVersion()` / `session-manager.ts:265-275`）。

### B.7 循環終止（所有終止路徑）

1. **正常終止**：內層 loop 結束（無 tool calls + 無 steering messages）且外層檢查 follow-up queue 為空 → emit `agent_end` → `break`
2. **錯誤終止**：`streamAssistantResponse()` 返回 stopReason "error" → emit `turn_end` + `agent_end` → `return`
3. **中斷終止**：stopReason "aborted"（AbortSignal 觸發）→ 同上
4. **外部 abort**：`Agent.abort()` / `AgentHarness.abort()` 觸發 AbortController
5. **shouldStopAfterTurn hook**：回傳 true → emit `agent_end` → `return`
6. **批次終止條件**：`shouldTerminateToolBatch()` (`agent-loop.ts:544-546`) — 當某批次中所有工具結果的 `terminate` 都為 true 時，該次 turn 不接續

---

## 任務 C：Extension System 深度分析

### C.1 Extension 加載

**入口：** `discoverAndLoadExtensions()` (`coding-agent/src/core/extensions/loader.ts:575-621`)

**載入順序：**
1. 專案本地：`${cwd}/.pi/extensions/`
2. 全域：`${agentDir}/extensions/`
3. 明確配置的 paths（CLI `--extensions` / settings）

**發現邏輯**（`discoverExtensionsInDir()` / `loader.ts:538-570`）：
1. 直接檔案：`*.ts` 或 `*.js` → 直接載入
2. 子目錄 + index.ts/js → 載入 index
3. 子目錄 + package.json 含 `pi.extensions` field → 載入聲明的路徑
4. 不遞迴超過一層

**載入機制**（`loadExtensionModule()` / `loader.ts:356-368`）：
- 使用 `jiti`（Node.js）或 virtualModules（Bun binary）載入 TypeScript/JavaScript
- 每個 extension 是一個 factory function：`(pi: ExtensionAPI) => void | Promise<void>`
- 共享 runtime（`ExtensionRuntime`）用 `assertActive()` + `invalidate()` 防止 stale context 使用

### C.2 Extension 介面（完整 API Surface）

`ExtensionAPI` (`types.ts:1084-1311`)：

| 方法 | 功能 |
|------|------|
| `on(event, handler)` | 訂閱 26+ 種 extension event |
| `registerTool(tool)` | 註冊 LLM-callable tool（ToolDefinition 格式） |
| `registerCommand(name, options)` | 註冊 slash command |
| `registerShortcut(shortcut, options)` | 註冊鍵盤快捷鍵 |
| `registerFlag(name, options)` | 註冊 CLI flag |
| `registerMessageRenderer(customType, renderer)` | 註冊自訂訊息渲染器 |
| `getFlag(name)` | 取得 CLI flag 值 |
| `sendMessage(message, options?)` | 發送 custom message 到 session |
| `sendUserMessage(content, options?)` | 發送 user message |
| `appendEntry(customType, data?)` | 寫入自訂 session entry |
| `setSessionName(name)` | 設定 session 顯示名稱 |
| `getSessionName()` | 取得 session 名稱 |
| `setLabel(entryId, label)` | 標記 entry |
| `exec(command, args, options?)` | 執行 shell 指令 |
| `getActiveTools()` | 取得 active tool 名稱列表 |
| `getAllTools()` | 取得所有工具資訊 |
| `setActiveTools(toolNames)` | 設定 active tools |
| `getCommands()` | 取得可用 slash commands |
| `setModel(model)` | 切換 model |
| `getThinkingLevel()` / `setThinkingLevel()` | Thinking level 管理 |
| `registerProvider(name, config)` | 註冊/覆寫 model provider |
| `unregisterProvider(name)` | 移除 provider |
| `events` | 跨 extension EventBus |

**Extension event 類型**（26+ events）：

| Event | 觸發時機 | 可回傳 |
|-------|----------|--------|
| `resources_discover` | 啟動/重載時 | 額外 resource paths |
| `session_start` | session 啟動 | — |
| `session_before_switch` | 切換 session 前 | `cancel` |
| `session_before_fork` | fork 前 | `cancel`, `skipConversationRestore` |
| `session_before_compact` | compaction 前 | `cancel`, `compaction` |
| `session_compact` | compaction 後 | — |
| `session_shutdown` | extension runtime 關閉 | — |
| `session_before_tree` | 樹導航前 | `cancel`, `summary` |
| `session_tree` | 樹導航後 | — |
| `context` | LLM 調用前 | 修改 messages |
| `before_provider_request` | 發送請求前 | 替換 payload |
| `after_provider_response` | 收到 response 後 | — |
| `before_agent_start` | agent loop 啟動前 | 附加 message、替換 system prompt |
| `agent_start` | agent loop 啟動 | — |
| `agent_end` | agent loop 結束 | — |
| `turn_start` | 每個 turn 開始 | — |
| `turn_end` | 每個 turn 結束 | — |
| `message_start` | message 開始 | — |
| `message_update` | streaming update | — |
| `message_end` | message 結束 | 替換 message |
| `tool_execution_start` | tool 執行開始 | — |
| `tool_execution_update` | tool 進度更新 | — |
| `tool_execution_end` | tool 執行結束 | — |
| `model_select` | model 變更 | — |
| `thinking_level_select` | thinking level 變更 | — |
| `tool_call` | tool 調用前 | `block`, `reason` |
| `tool_result` | tool 結果後 | 修改 content/details/isError |
| `user_bash` | 使用者 !/!! 指令 | 自訂 bash operations |
| `input` | 使用者 input | `continue`, `transform`, `handled` |

### C.3 Extension 與 Runtime 的邊界

Extension **不能直接修改 runtime loop 本身**。可以做的事：

- **Hook into loop lifecycle**（26+ event hooks）
- **修改 context** 在送出前（`context` event → `transformContext` callback）
- **修改 payload** 在發送前（`before_provider_request` event）
- **修改 tool result** 在回注前（`afterToolCall` hook）
- **注冊 tool**（`registerTool()`）
- **注入 message**（`sendMessage()` / `sendUserMessage()`）
- **影響 loop 控制流**（`beforeToolCall` → `{ block: true }`、`session_before_switch` → `{ cancel: true }`）
- **註冊 provider**（`registerProvider()` — 可自訂 streamSimple handler）

Extension 無法修改的事：
- 不能直接改寫 `runLoop()` 邏輯
- 不能新增 event types
- 不能改變 tool dispatch 的 parallel/sequential 策略

### C.4 內建 vs 外部 Extension

**目前所有功能都是內建**（built-in tools、system prompt、compaction、session management）。Extension system 是**擴充機制**，不是核心功能的實現層。

外部 extension 能做的是：
1. 註冊自訂 LLM-callable tools
2. 註冊 slash commands（`/mycommand`）
3. 註冊鍵盤快捷鍵
4. 註冊 CLI flags
5. 註冊客製化 provider（自訂 baseUrl、model list、stream handler）
6. 註冊自訂訊息渲染器
7. 監聽並影響所有 session/agent/tool lifecycle events
8. 跨 extension 通訊（EventBus）

### C.5 MCP 整合

Pi **沒有原生 MCP 支援**。MCP servers 必須以 extension 的形式整合（extension 內部使用 MCP SDK 作為 client）。這與 Reasonix 和 Claude Code 不同—後兩者都有原生 MCP 協定支援。

---

## 任務 D：Tool 系統完整圖譜

### D.1 所有內建 Tools

| Tool | 文件位置 | 功能 |
|------|---------|------|
| read | `coding-agent/src/core/tools/read.ts` | 讀取檔案（支援行數限制、offset） |
| bash | `coding-agent/src/core/tools/bash.ts` | 執行 shell 指令（支援 timeout、cwd、streaming output） |
| edit | `coding-agent/src/core/tools/edit.ts` | 編輯檔案（diff-based，支援多種編輯策略） |
| write | `coding-agent/src/core/tools/write.ts` | 寫入/覆寫檔案（支援大檔案） |
| grep | `coding-agent/src/core/tools/grep.ts` | 全文搜尋（目錄/pattern） |
| find | `coding-agent/src/core/tools/find.ts` | 檔案搜尋（模式/類型） |
| ls | `coding-agent/src/core/tools/ls.ts` | 列出目錄內容 |
| truncate | `coding-agent/src/core/tools/truncate.ts` | Token-based 截斷工具（輔助用，非 LLM-callable） |
| file-mutation-queue | `coding-agent/src/core/tools/file-mutation-queue.ts` | 檔案變更佇列（edit/write 防衝突） |

### D.2 Tool 定義格式

Pi 有兩種 tool 定義格式：

#### 底層：AgentTool (`agent/src/types.ts:361-384`)
```typescript
interface AgentTool<TParameters extends TSchema, TDetails> extends Tool<TParameters> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;
  execute: (toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;
  executionMode?: ToolExecutionMode;
}
```

#### 高層：ToolDefinition (`coding-agent/src/core/extensions/types.ts:426-473`)
```typescript
interface ToolDefinition<TParams, TDetails, TState> {
  name: string;
  label: string;
  description: string;
  promptSnippet?: string;         // system prompt 中的一行摘要
  promptGuidelines?: string[];    // system prompt guidelines 附加
  parameters: TParams;            // TypeBox schema
  renderShell?: "default" | "self";
  prepareArguments?: (args) => Static<TParams>;
  executionMode?: ToolExecutionMode;
  execute: (toolCallId, params, signal, onUpdate, ctx) => Promise<AgentToolResult<TDetails>>;
  renderCall?: (args, theme, context) => Component;
  renderResult?: (result, options, theme, context) => Component;
}
```

主要差異：`ToolDefinition` 多了 UI 相關欄位（`promptSnippet`、`renderShell`、`renderCall`/`renderResult`）以及執行時期的 `ExtensionContext`。

轉換：`wrapRegisteredTool()` (`coding-agent/src/core/extensions/wrapper.ts`) 將 `ToolDefinition` 轉換為 `AgentTool`。

### D.3 Tool 權限模型

Pi **沒有明確的 allow/deny/ask 權限系統**。採用的是：

1. **`beforeToolCall` hook** (`agent/src/agent-loop.ts:581-605`)：可以返回 `{ block: true, reason: "..." }` 阻止 tool 執行
2. **Active tool 過濾**：`AgentSession.setActiveToolsByName()` 控制哪些 tool 在 context 中
3. **`allowedToolNames`** (`coding-agent/src/core/agent-session.ts:171`)：選用的 allowlist
4. **`noTools` / `noBuiltinTools`** 啟動選項

**錯誤路徑：** Tool 找不到時，`prepareToolCall()` 回傳 `{ kind: "immediate", result: errorToolResult }`，不會 throw。Tool 執行中拋錯被 `executePreparedToolCall()` 捕獲並編碼為 error tool result。

### D.4 Tool 結果處理

1. Tool 返回 `AgentToolResult<T>`（`content: (TextContent|ImageContent)[]` + `details: T`）
2. `finalizeExecutedToolCall()` 允許 `afterToolCall` hook 修改結果
3. `createToolResultMessage()` (`agent-loop.ts:727-737`) 轉為 `ToolResultMessage`
4. 加入 `currentContext.messages`，emit `message_start` + `message_end` 事件
5. 在下次 LLM 調用時，`convertToLlm()` 將 ToolResultMessage 原樣保留

**Terminate 機制**：如果批次中所有 tool result 的 `terminate` 皆為 true，該 turn 結束後不再接續 LLM 調用。

---

## 任務 E：Provider 抽象層

### E.1 支援的 Provider

透過 `packages/ai/src/providers/register-builtins.ts:345-399` `registerBuiltInApiProviders()` 註冊：

| API 類型 | Provider 名稱 | 實作文件 |
|----------|--------------|---------|
| `anthropic-messages` | anthropic、deepseek、xai、groq 等 | `providers/anthropic.ts` |
| `openai-completions` | openai、deepseek、xai、groq、cerebras、openrouter 等 | `providers/openai-completions.ts` |
| `openai-responses` | openai、openai-codex | `providers/openai-responses.ts` |
| `azure-openai-responses` | azure-openai | `providers/azure-openai-responses.ts` |
| `openai-codex-responses` | openai-codex | `providers/openai-codex-responses.ts` |
| `google-generative-ai` | google | `providers/google.ts` |
| `google-vertex` | google-vertex | `providers/google-vertex.ts` |
| `bedrock-converse-stream` | amazon-bedrock | `providers/amazon-bedrock.ts` |
| `mistral-conversations` | mistral | `providers/mistral.ts` |

已知 provider 名稱（`ai/src/types.ts:23-55` `KnownProvider`）：amazon-bedrock、anthropic、google、google-vertex、openai、azure-openai-responses、openai-codex、deepseek、github-copilot、xai、groq、cerebras、openrouter、vercel-ai-gateway、zai、mistral、minimax、moonshotai、huggingface、fireworks、together、opencode、kimi-coding、cloudflare-workers-ai、cloudflare-ai-gateway、xiaomi 等。

### E.2 Provider 介面

`api-registry.ts:23-27`：
```typescript
interface ApiProvider<TApi, TOptions> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

抽象層定義的功能：
- `stream()` — 低階 streaming（provider-specific options）
- `streamSimple()` — 統一 SimpleStreamOptions 的 streaming
- 透過 `complete()` / `completeSimple()` 提供非 streaming completion
- **無 embedding、無 images generation（另有 images API registry）**

每種 API 類型在註冊表中有一個 entry。`streamSimple()` 為所有 provider 提供一致介面。

### E.3 Provider 切換

**配置方式：**
1. **Model selection**：`Model.id` + `Model.provider` + `Model.api` 字段
2. **ModelRegistry** (`coding-agent/src/core/model-registry.ts`)：提供 model 查找、API key 解析
3. **CLI**：`--model provider/pattern`、`--provider`、`--models`（scoped models）
4. **Settings**：`settings.json` 中的 `defaultProvider` / `defaultModel`
5. **UI**：Ctrl+P cycling、model selector

切換時，`AgentSession.setModel()` → `agent.state.model` 更新 → 下次 LLM 調用使用新 model。`AgentLoopConfig.model` 在每次 `prepareNextTurn` 時刷新。

### E.4 Provider-Specific 行為

| Provider | 優化 |
|----------|------|
| **Anthropic** | `cache_control` markers（system prompt、last tool def、last user/assistant text）、`sessionId` 傳遞、`x-session-affinity` header、thinking budget、eager input streaming |
| **OpenAI (Completions)** | `reasoning_effort`、`max_completion_tokens`、`store` field、`stream_options.include_usage`、OpenRouter routing、prompt cache retention |
| **OpenAI (Responses)** | `session_id` header、`previous_response_id`、`prompt_cache_retention` |
| **DeepSeek** | 透過 `openai-completions` adapter 支援 `thinking: { type }` format |
| **Google** | `thoughtSignature` for multi-turn thinking、context caching |
| **Bedrock** | AWS SDK-based invocation |
| **GitHub Copilot** | OAuth token 認證、動態 token refresh |
| **OpenRouter** | Provider routing preferences、`HTTP-Referer`/`X-OpenRouter-Title` headers |

---

## 任務 F：Prompt / Skill / Template 系統

### F.1 System Prompt

**構建流程**（`coding-agent/src/core/system-prompt.ts:28-175` `buildSystemPrompt()`）：

1. 如果 `customPrompt` 存在：
   - 使用自訂 prompt 作為基底
   - 附加 appendSystemPrompt
   - 附加 `<project_context>`（context files）
   - 附加 skills（如果 `read` tool active）
   - 附加日期 + CWD

2. 否則（預設）：
   - 開頭：「You are an expert coding assistant operating inside pi, a coding agent harness.」
   - `Available tools:` — 只有提供 `promptSnippet` 的 tool 會出現
   - `Guidelines:` — 動態生成（根據 active tools）
   - Pi 文件路徑說明
   - 附加 appendSystemPrompt
   - `<project_context>`
   - Skills
   - 日期 + CWD

3. 預設 system prompt 中嵌入的 Pi 文件路徑是**寫死的**（README path、docs path、examples path）。

**系統 prompt 支援模板化：**
- 可透過 `--system-prompt` CLI flag 提供完整取代
- 可透過 `--append-system-prompt` 附加 supplement
- Extension 可透過 `before_agent_start` event 修改系統 prompt

### F.2 Skills

**發現與載入：**

有兩個層級的 skill 系統：

1. **底層** (`agent/src/harness/skills.ts`)：
   - 載入 `SKILL.md` 檔案
   - 支援 ignore 檔案（.gitignore、.ignore、.fdignore）
   - YAML frontmatter 解析
   - `formatSkillInvocation()` 生成 XML skill block

2. **高層** (`coding-agent/src/core/skills.ts`)：
   - 相同的載入邏輯，加上 source tracking（`SourceInfo`）
   - `formatSkillsForPrompt()` 生成 `<available_skills>` XML block

**載入來源**（`coding-agent/src/core/resource-loader.ts`）：
- 專案本地：`${cwd}/.pi/skills/`
- 全域：`${agentDir}/skills/`
- CLI `--skills` path
- Extension `resources_discover` event 附加路徑

### F.3 Prompt Templates

Prompt templates 是 `.md` 檔案，有 YAML frontmatter，放在 `${cwd}/.pi/prompts/` 或 `${agentDir}/prompts/`。

格式：
```markdown
---
name: explain
description: Explain a concept
---
Explain {{arg0}} in simple terms, with examples.
```

使用：`/explain functional programming` 展開為對應 prompt。支援 `{{arg0}}`、`{{arg1}}` 等變數替換。

### F.4 Extension 的自訂 Provider 註冊

Extension 可透過 `pi.registerProvider("name", config)` 註冊自訂 provider：
- 指定 `baseUrl` 覆蓋現有 provider 的端點
- 提供完整 model list 取代該 provider
- 提供 `oauth` 配置支援 OAuth 登入
- 提供自訂 `streamSimple` handler 處理自訂 API

---

## 任務 G：與 Reasonix / Claude Code 的結構對比

| 維度 | Pi | Reasonix | Claude Code |
|------|-----|-----------------|-------------------|
| **Loop 結構** | 雙層 while-loop（外層 follow-up / 內層 tool-steering），imperative | 已知 | 單層 while + break，callback-based |
| **Context 管理** | AgentMessage[] array，Session tree 線性化，多 custom role 型別 | 已知 | SessionMemory + autocompact，message 陣列 |
| **Tool dispatch** | parallel（`Promise.all`）/ sequential 可選，preflight validation | 已知 | concurrency safety 分批，sequential by default |
| **Extension/Plugin** | 26+ event hooks + `registerTool()` + `registerCommand()` + `registerProvider()`，jiti 動態載入 | 已知 | Skills + MCP + hooks，無 provider 註冊 |
| **Session** | JSONL 檔案，linked-list tree，compaction/branch_summary/custom entries，支援 fork | 已知 | MEMORY.md + SessionMemory，無 tree |
| **Compaction** | agent-core 層 `compact()`，turn-end auto-check，LLM 摘要（增量/全新），split-turn support，file operation tracking | 已知 | staged collapse + reactive，摘要 + 截斷混合 |
| **Provider** | 9 API types，20+ known providers，lazy-load provider modules，自訂 provider 可動態註冊 | 已知（DeepSeek-only） | 已知（Anthropic-native） |
| **Philosophy** | harness-first：分層（Agent → AgentHarness → AgentSession → InteractiveMode），extensible by design | 已知（cache-first） | 已知（continuity-first） |
| **Tool 系統** | 7 built-in tools（read/bash/edit/write/grep/find/ls），ToolDefinition + AgentTool 雙層定義 | 已知 | read/bash/edit/write 4 tools |
| **Extension 權限** | Tool/command/keybinding/provider 註冊，event 監聽，**不能**修改 loop 結構 | 已知 | Skills + MCP + hooks |

---

## 驗證命令

以下 grep 命令驗證報告中引用的每個關鍵函數名：

```bash
# 任務 A - Project Structure
grep -n "^export.*function\|^export.*class\|^export interface" /Users/lesprivilege/Projects/pi/packages/agent/src/index.ts
grep -n "^export.*function\|^export.*class\|^export interface" /Users/lesprivilege/Projects/pi/packages/ai/src/index.ts
grep -n "^export.*function\|^export.*class\|^export interface" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/index.ts

# 任務 B - Runtime Loop
grep -n "^export async function runLoop\|^export function runAgentLoop\|^async function runLoop\|^async function streamAssistantResponse\|^async function executeToolCalls\|^async function prepareToolCall\|^async function executePreparedToolCall\|^async function finalizeExecutedToolCall" /Users/lesprivilege/Projects/pi/packages/agent/src/agent-loop.ts

# 任務 B - Compaction
grep -n "^export function shouldCompact\|^export function prepareCompaction\|^export async function compact\|^export async function generateSummary\|^export function estimateContextTokens\|^export function findCutPoint" /Users/lesprivilege/Projects/pi/packages/agent/src/harness/compaction/compaction.ts
grep -n "^export function convertToLlm" /Users/lesprivilege/Projects/pi/packages/agent/src/harness/messages.ts

# 任務 B - Session
grep -n "^export function buildSessionContext\|^export class Session\b" /Users/lesprivilege/Projects/pi/packages/agent/src/harness/session/session.ts
grep -n "^export class SessionManager\b\|^export function buildSessionContext\b\|^export function migrateSessionEntries\|^export function parseSessionEntries" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/session-manager.ts

# 任務 B - Agent
grep -n "^export class Agent\b\|^class Agent\b\|^\s*async prompt\|^\s*async continue\|^\s*private async runPromptMessages\|^\s*private async runWithLifecycle\|^\s*private async processEvents" /Users/lesprivilege/Projects/pi/packages/agent/src/agent.ts

# 任務 C - Extension
grep -n "^export.*function\|^export.*class\|^export interface" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/extensions/types.ts | head -40
grep -n "^export async function discoverAndLoadExtensions\|^export async function loadExtensions\|^export function createExtensionRuntime\|^function createExtensionAPI\|^async function loadExtensionModule\|^function discoverExtensionsInDir" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/extensions/loader.ts

# 任務 D - Tools
grep -n "^export function createAllTools\|^export function createToolDefinition\|^export function createCodingTools\|^export function createReadOnlyTools" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/tools/index.ts

# 任務 E - Provider
grep -n "^export function registerApiProvider\|^export function getApiProvider\|^export function clearApiProviders\|^export function streamSimple\|^export function stream\b\|^export function completeSimple" /Users/lesprivilege/Projects/pi/packages/ai/src/stream.ts
grep -n "^export function registerBuiltInApiProviders\|^export function setBedrockProviderModule\|^export function resetApiProviders" /Users/lesprivilege/Projects/pi/packages/ai/src/providers/register-builtins.ts

# 任務 F - System Prompt / Skills
grep -n "^export function buildSystemPrompt\|^export function formatSkillsForPrompt" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/system-prompt.ts
grep -n "^export function loadSkills\b\|^export async function loadSourcedSkills\|^export function formatSkillInvocation\|^export function formatSkillsForSystemPrompt" /Users/lesprivilege/Projects/pi/packages/agent/src/harness/skills.ts
grep -n "^export function loadSkills\b\|^export function formatSkillsForPrompt\|^export function loadSkillsFromDir" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/skills.ts

# AgentSession
grep -n "class AgentSession\b\|^\s*async prompt\|^\s*private _handleAgentEvent\|^\s*private _rebuildSystemPrompt\|^\s*private async _runAgentPrompt\|^\s*private async _handlePostAgentRun\|^\s*private async _emitExtensionEvent" /Users/lesprivilege/Projects/pi/packages/coding-agent/src/core/agent-session.ts
