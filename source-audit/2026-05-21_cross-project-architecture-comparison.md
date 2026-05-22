# 四專案橫向架構比較表

**日期：** 2026-05-21
**用途：** DeepSeek Harness PM 面試即時調用素材
**覆蓋專案：** DeepSeek-TUI (Rust, L2) · Reasonix (TS, L2) · Pi (TS, L2) · ds4 (C, L1)

---

## 1. Runtime Loop

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **語言** | Rust | TypeScript | TypeScript | C |
| **Loop 結構** | Channel-driven：UI → `EngineHandle.send(Op)` → `Engine::run` event loop（`rx_op.recv().await`） | Generator-based：`CacheFirstLoop.step()` 是 `async *` generator，`for (;;)` 無上限 iter | 雙層 while-loop：外層 follow-up queue、內層 tool-steering（`runLoop()` at agent-loop.ts:155） | 單線程 worker：`worker_main()` dequeue → `generate_job()`，無 agent loop（純推理引擎） |
| **Turn 入口** | `Engine::handle_send_message` (engine.rs:593) | `step()` (loop.ts:539) | `Agent.prompt()` → `runAgentLoop()` → `runLoop()` | `generate_job()` (ds4_server.c:10505) |
| **Iter 終止** | Turn-level：stream 結束 + tool 全部執行完 = TurnComplete event | 8 條終止路徑：end_turn / budget / context-guard / storm-stuck / user-abort / API-error / API-abort / escalation-loop | 6 條終止路徑：正常（無 tool calls）/ error / aborted / external abort / shouldStopAfterTurn hook / batch terminate | 單次 request-response，無 iter |
| **Follow-up 機制** | 無明確 follow-up queue；sub-agent 用 `rx_subagent_completion` channel sentinel 注入 | 無 follow-up queue；每次 `step()` 只處理一輪 user input | 雙層設計：外層 `followUpQueue`、內層 `steeringQueue`（`pendingMessages`） | N/A |

**PM 洞察：** TUI 的 channel-driven 設計讓 UI 和 engine 完全解耦，適合做 approval flow 等阻塞操作。Reasonix 的 generator 模式讓調用方可以 `for await` 逐事件消費，最適合做 TUI streaming。Pi 的雙層 queue 最靈活（extension 可注入 steering message），但複雜度最高。ds4 不是 agent，是推理引擎——沒有 loop，只有 request → inference → response。

---

## 2. Context Assembly

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **System Prompt** | `system_prompt_for_mode_with_context_skills_session_and_approval(mode, ...)` — mode-aware + skills + session context + approval mode | `ImmutablePrefix.toMessages()` — `{system, fewShots}`，不可變 | 兩層：底層 `AgentHarness`（string 或 callback）→ 高層 `buildSystemPrompt()` 組裝 tools + guidelines + context files + skills + date/CWD | `render_chat_prompt_text()` — 把 messages + tool schemas 渲染為 DSML 標記的 token 序列 |
| **History 管理** | `session.messages`（最多 500 條持久化）+ compaction summary | 三區分治：ImmutablePrefix / AppendOnlyLog / VolatileScratch | `Agent.state.messages` + Session tree（JSONL parentId chain → `buildSessionContext()` 線性化） | 無 history 管理——每次 request 的 messages 由調用方（L2 harness）提供 |
| **動態 Context** | LSP diagnostics injection（`flush_pending_lsp_diagnostics`）、layered context checkpoint、auto-compaction | Skill pin（`<skill-pin>` 區塊在 compaction 中保留）、pending user message 注入 | Extension hooks：`transformContext` → 修改 messages、`before_provider_request` → 修改 payload、context files、skills | tool_memory：exact DSML replay（`tool_memory_attach_to_messages`）、7 層 cache lookup |
| **Healing/Repair** | Markdown-based fallback tool parsing（`tool_parser::parse_tool_calls`）、transparent stream retry | `healActiveLogBeforeSend()`：`shrinkOversizedToolResults` + `fixToolCallPairing` → 修復後 `compactInPlace` + `rewriteSession` | `convertToLlm()`：custom message types → standard roles、unknown role → filter out | 無——L1 層不做 message repair |

**PM 洞察：** Reasonix 的 healing pipeline 最獨特——它在發送前主動修復 broken tool call pairing，這直接保護 prefix cache 一致性。TUI 的 LSP injection 是差異化功能（edit 後自動注入 diagnostics）。ds4 的 tool_memory 是最底層的 cache 保護機制——byte-level exact replay。

---

## 3. Tool Dispatch

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **並行策略** | 批次規劃：`plan_tool_execution_batches()` → Parallel(read-only) / Serial(write/approval)。使用 `RwLock`：read tools 共享鎖、write tools 排他鎖 | 連續分組：parallelSafe flag 連續的 calls 分組（max 3），`Promise.allSettled`。無 data dependency 分析 | 兩模式可選：`executeToolCallsParallel()`（`Promise.all`）/ `executeToolCallsSequential()`。由 tool 的 `executionMode` 或 config 決定 | N/A（ds4 不執行 tool，只生成 tool call 指令） |
| **權限模型** | 三值 Allow/Deny/Ask：mode-based（Plan 封鎖 destructive）+ per-tool `approval_requirement` + loop guard（重複檢測） | 無權限系統 | 無明確權限：`beforeToolCall` hook 可返回 `{block: true}` + `allowedToolNames` allowlist + active tool 過濾 | N/A |
| **錯誤處理** | Tool 失敗 → `ToolResult { success: false }` → 記入 session，不中斷其他並行工具 | `Promise.allSettled` → rejected 包裝為 `{error: "name: message"}` JSON → 回注 log | 捕獲 → error ToolResult → 回注 messages → LLM 看到錯誤，自行決策 | Malformed tool call → 作為 assistant text 返回而非 error |
| **Repair pipeline** | Markdown fallback parsing（4 種格式）+ transparent stream retry（max 2 次） | 三階段：Scavenge（DSML/JSON pattern 掃出遺漏 calls）→ Truncation repair（修復截斷 JSON）→ Storm breaker（抑制重複 calls，window=6, threshold=3） | 無 repair pipeline | DSML decode tracker + `observe_tool_markers()` 在 decode 階段即時追蹤 |

**PM 洞察：** Reasonix 的三階段 repair pipeline 是最精密的——它承認 LLM 的 tool call 輸出不可靠，系統性地修復。TUI 的 RwLock 並行模型最工程化（讀寫鎖語義）。Storm breaker 的 self-correction 機制（全部被擋 → 給 model 一次自我修正機會）是 PM 可以展開的 UX 決策點。

---

## 4. Session Persistence

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **儲存格式** | JSON（`SavedSession` struct → `serde_json::to_string_pretty`）+ git snapshot | JSONL（`appendSessionMessage` 逐行追加）| JSONL tree（每行一個 entry，含 `parentId`、`type`）| `.kv` 二進制檔案（SHA1 header + text + KV payload + tool map）|
| **數據結構** | Flat messages array（max 500）| Flat messages array（AppendOnlyLog 的持久化鏡像） | Tree：linked-list via parentId → branching / fork / compaction entries | KV cache entries（token prefix → GPU state snapshot）|
| **寫入時機** | Checkpoint（每次 dispatch 前）+ SessionSnapshot（TurnComplete 後）。Persistence actor 用 latest-wins coalescing | 每條 message 即時 append（`appendAndPersist`）| 每條 entry 即時 append 到 JSONL | 4 種時機：cold（prefill 前）/ continued（每 10K tokens）/ evict（cache miss 前）/ shutdown |
| **原子性** | `write_atomic`（NamedTempFile + fsync + rename） | 無原子性保證（直接 append）| 無原子性保證（JSONL append） | 原子寫入（`.tmp` → `rename` to `.kv`）|
| **Branching** | 無 branching（linear history + snapshot rollback）| 無 branching | 完整 tree：`Session.moveTo(entryId)` + branch_summary + fork | N/A |
| **容量控制** | MAX_SESSIONS=50、MAX_PERSISTED_MESSAGES=500 | 無明確限制（由 compaction 控制 memory 量） | 無明確限制（由 compaction 控制） | cache eviction：score = `(hits+1) × tokens / size`，LRU with 6h half-life decay |

**PM 洞察：** TUI 的 snapshot 機制（side git repo，pre/post-turn + pre-tool，500MB cap，50 snapshots，7-day retention）是最完整的 rollback 方案——這是 UX 護城河。Pi 的 session tree 最靈活（branching + fork），是 "Neovim for agents" 哲學的體現。ds4 的 KV cache persistence 是最底層的——它保存的不是對話，而是 GPU 計算狀態。

---

## 5. Compaction / Context Management

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **觸發條件** | `should_compact()` at turn start（token 比例閾值）| `decideAfterUsage()`：>80% → exit-with-summary、50-80% → fold、<50% → none | `shouldCompact()`：`contextTokens > contextWindow - reserveTokens`（reserve=16384） | 無 compaction（L1 層，由 cache eviction 替代） |
| **摘要模型** | LLM（`compact_messages_safe`）| flash model（`deepseek-v4-flash`，15s timeout，non-streaming）| LLM（`completeSimple()`，支援增量更新 `UPDATE_SUMMARIZATION_PROMPT`） | N/A |
| **保留策略** | Compaction summary + recent messages | tail budget（ctxMax × 0.1 或 0.2）+ skill pin 保留 | `keepRecentTokens`（20000）+ file operations tracking（read/written files）| N/A |
| **Emergency 策略** | 無明確 emergency path | `mechanicalTruncate()`：preflight 檢查 ratio > 0.95 時純丟棄（無 API call，與 decideAfterUsage 是不同決策點） | 無明確 emergency path | `kv_cache_evict()`：LRU with decay score |
| **Cache 影響** | 摘要替換舊 messages → prefix cache miss | fold 改變 token 序列 → prefix cache 必然 miss。**無動態 cost-benefit 權衡** | compaction summary 替換 → cache 可能失效。`sessionId` 傳遞以優化 session-aware caching | 7 層 cache hierarchy：live token/text/visible prefix → disk `.kv` load |

**PM 洞察：** Reasonix 的兩階段設計（fold at 50-70% + force exit at 80%）最精細，且用 flash model 做 summary 控制成本。Pi 的增量 summary（UPDATE_SUMMARIZATION_PROMPT）避免資訊丟失。ds4 的 7 層 cache lookup 是面試必講的深度點——它展示了 L1 引擎如何用 cache hierarchy 替代 L2 的 compaction。

---

## 6. Provider / API Abstraction

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **Provider 數量** | 10+ providers（`deepseek-agent` crate：DeepSeek、OpenAI、Anthropic、Google 等）| 1（DeepSeek-only：`DeepSeekClient`，hardcoded `baseUrl`）| 20+ known providers（9 API types：anthropic-messages、openai-completions、openai-responses 等），lazy-load modules | 1（自身即 provider，暴露三協定端點）|
| **切換機制** | Auto-router：heuristic fast-path → flash router slow-path（4s timeout）→ fallback | flash-first + `<<<NEEDS_PRO>>>` auto-escalation + `/pro` 單 turn 提權 | `ModelRegistry` + CLI `--model` + UI Ctrl+P cycling + Extension `registerProvider()` | 模型內建於引擎，無切換 |
| **API 格式** | OpenAI Chat Completions（SSE streaming）| OpenAI Chat Completions（SSE streaming，660s timeout）| 每種 API type 有專門 provider module：Anthropic Messages、OpenAI Completions/Responses、Google、Bedrock 等 | 三協定入口：`/v1/chat/completions`、`/v1/messages`、`/v1/responses` |
| **Provider 特化** | Auto model selection（heuristic + flash router）、transparent stream retry | DeepSeek-specific：thinking mode（`<<<NEEDS_PRO>>>` 標記）、prefix cache pricing awareness、RPM rate limiting | Per-provider 優化：Anthropic `cache_control` markers + `sessionId`、OpenAI `reasoning_effort`、Google `thoughtSignature`、Copilot OAuth refresh | 三協定各自的 live state 管理：Responses（visible text + call IDs）、Anthropic（call IDs only）、Chat（無 live state） |

**PM 洞察：** 這裡的分層最清晰——ds4 是三協定 **server**（暴露 API），TUI/Reasonix 是 **client**（消費 API），Pi 是 **multi-client**（消費多種 API）。Reasonix 的 DeepSeek-first 策略讓它可以做 flash→pro escalation 和精確的 cache pricing 計算，但也限制了生態擴展。Pi 的 extension-based provider 註冊最開放。

---

## 7. Async Architecture

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **Runtime** | Tokio（multi-threaded） | Node.js single-thread + async generators | Node.js single-thread + async/await | pthreads（single worker thread + detached client threads）|
| **關鍵 Async 邊界** | UI → Engine：`EngineHandle.send(Op)` channel。Engine → Tools：`RwLock` + `FuturesUnordered`。Engine → Persistence：`unbounded_channel` fire-and-forget。Engine → SubAgent：`rx_subagent_completion` channel | Loop → Client：`this.client.stream()` async generator yield。Loop → Tools：`Promise.allSettled` | Agent → API：`streamFn()` async iterator。Agent → Tools：`Promise.all` / sequential | Client → Worker：`pthread_cond_wait/signal`（job queue）。Accept → Client：detached pthread per connection |
| **Cancellation** | `cancel_token`（CancellationToken）+ approval channel close | `AbortController` per turn（`_turnAbort`）+ carry-over abort forwarding | `AbortSignal` propagation through `runLoop` → tools | `g_stop_requested` flag（SIGINT/SIGTERM）|
| **Backpressure** | Persistence actor：latest-wins coalescing（同類型多請求只寫最後一個）| 無明確 backpressure | 無明確 backpressure | Job queue：linked list + `pthread_cond_wait`（worker 忙時 client 阻塞）|

**PM 洞察：** TUI 的 async 架構最成熟——channel-based decoupling 讓每個子系統獨立演進。Persistence actor 的 latest-wins coalescing 是生產級設計。ds4 的 single worker thread 是有意為之——GPU inference 本質上是序列化的，多線程只增加鎖開銷。

---

## 8. Error Philosophy

| 維度 | DeepSeek-TUI | Reasonix | Pi | ds4 |
|------|-------------|----------|-----|-----|
| **核心理念** | 錯誤不中斷 session：tool 失敗 → 記錄為 `{success: false}` → 繼續。Snapshot 操作全部 non-fatal | 錯誤編碼為 event yield：`{role: "error"}` → 返回給調用方決策。Healing 吞錯（in-memory 已修復） | 錯誤編碼在返回值：stopReason "error" → loop 結束、tool 錯誤 → error ToolResult 回注 messages | 錯誤映射為 HTTP status：400/409/500/503。Malformed tool call → 降級為 text |
| **Self-correction** | Loop guard：重複 tool call 檢測 → block | Storm breaker：全部被擋 → 插入 stub responses → 給 model 一次自我修正。第二次全擋 → force summary exit | `shouldStopAfterTurn` hook 可由 extension 實現自定義停止邏輯 | DSML decode tracker：未終止的 tool call → `finish="error"` |
| **Retry 策略** | Transparent stream retry（stream 中斷且無內容 → 重發，max 2 次）| 只 retry initial HTTP fetch（retryable: 408/429/5xx）。Streaming body 開始後不 retry | 無內建 retry（delegated to provider modules） | N/A（server 端，不做 client retry）|

---

## 9. 面試快速調用索引

### 「請比較 X 和 Y 的 Z」速查

| 話題 | 最佳對比對 | 關鍵差異 |
|------|-----------|---------|
| Loop 架構 | TUI vs Pi | Channel-driven（解耦）vs 雙層 while-loop（靈活）|
| Cache 策略 | Reasonix vs ds4 | L2 prefix cache 保護（healing + 三區分治）vs L1 七層 cache hierarchy |
| Tool 安全 | TUI vs Pi | 三值權限 + loop guard vs hook-based blocking + allowlist |
| 可擴展性 | Pi vs Reasonix | 26+ event hooks + provider registry vs DeepSeek-only 深度優化 |
| 錯誤恢復 | TUI vs Reasonix | Snapshot rollback（git-based）vs Storm breaker + self-correction |
| Provider 策略 | Pi vs TUI | Multi-provider（20+ known）vs Auto-router（heuristic + flash probe）|
| Session 模型 | Pi vs TUI | Tree（branching + fork）vs Linear + git snapshot |
| 成本控制 | Reasonix | flash-first + auto-escalation + budget guard + fold 用 flash model |

### 跨層架構論述模板

> "ds4 用七層 cache hierarchy 在 L1 保護 KV state continuity；Reasonix 在 L2 用三區分治 + healing pipeline 保護 prefix cache alignment；TUI 在 L2 用 snapshot 保護 application state continuity。三個層次各自解決不同粒度的『狀態延續』問題——這就是為什麼 upstream 做 harness 有結構性優勢：cache contract 可以跨層設計。"

### 可展開的技術深度點

1. **ds4 exact DSML replay**：tool_memory_remember → tool_memory_attach_to_messages → render 時優先 raw_dsml → byte-level KV alignment。若 ID miss → canonical rendering → canonicalize_tool_checkpoint rebuild。面試時可畫 flowchart。

2. **Reasonix repair pipeline**：Scavenge（從 reasoning/content 掃出遺漏 tool calls）→ Truncation（修復截斷 JSON）→ Storm（抑制重複，window=6）。三階段串聯，每階段都有 fallback。

3. **TUI approval flow**：`plan_tool_execution_batches()` → 批次中 approval_required = true → `Event::ApprovalRequired` → `await_tool_approval()` 阻塞 → UI 側 Approved/Denied/RetryWithPolicy 三選一。Channel-based 設計讓 UI 響應不阻塞 engine。

4. **Pi extension boundary**：26+ event hooks 但不能修改 loop 結構、不能新增 event types、不能改 dispatch 策略。這是刻意的——runtime loop 是不可替換的核心，extension 只能 hook 和 modify。

5. **ds4 KV cache eviction**：`score = (effective_hits + 1.0) × tokens / size`，hits 以 6h half-life 指數衰減。Just-written file 得 DBL_MAX 防止立即驅逐。Budget-driven 逐步刪除最低分。
