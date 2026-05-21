# Reasonix 執行路徑架構審計

> 日期：2026-05-21
> 目標 repo：`/Users/lesprivilege/Projects/DeepSeek-Reasonix`
> 範圍：src/loop.ts 主循環 → context 組裝 → API 調用 → repair pipeline → tool 執行 → compaction → cost control

---

## 任務 A：CacheFirstLoop.step() 一輪的完整邏輯

### A.1 進入時的初始狀態

```
CacheFirstLoop.step() (src/loop.ts:539)
```

- `_steerConsumed = false`
- `_turn++`（從 0 開始，每次 step 遞增）
- `scratch.reset()` — 清空 reasoning / planState / notes
- `repair.resetStorm()` — 清除 StormBreaker 的 sliding window
- `_turnSelfCorrected = false`
- `_escalateThisTurn = false`
- 如果 `_proArmedForNextTurn` 為 true → `_escalateThisTurn = true`；清除 armed 旗標
- 新 `AbortController` 給 `_turnAbort`（如有 carry-over abort 則 forward 到新 controller）
- `this.appendAndPersist({ role: "user", content: userInput })` — 在首次 API 調用前就寫入 log + session file
- `pendingUser = null`

**出錯路徑**：如果 budget 門檻觸發（`budgetUsd != null && spent >= budgetUsd`），yield `{ role: "error" }` 並 return，不修改 log。80% 警告只 yield `{ role: "warning" }`，繼續流程。

### A.2 Context 組裝順序

```
buildMessages() (src/loop.ts:488)
  ├── healActiveLogBeforeSend() (src/loop.ts:495)
  │     ├── 取得 log.toMessages() → current（AppendOnlyLog 的淺拷貝）
  │     ├── healLoadedMessages(current, DEFAULT_MAX_RESULT_CHARS) (src/loop/healing.ts:61)
  │     │     ├── shrinkOversizedToolResults() (src/loop/shrink.ts:17) — 裁切 tool result 的 content 字元數
  │     │     └── fixToolCallPairing() (src/loop/healing.ts:13) — 刪除未配對的 assistant.tool_calls 或 stray tool messages
  │     └── 如果有修復 → log.compactInPlace(healed.messages) + rewriteSession()
  └── 合併三層：
        [this.prefix.toMessages(), ...healedMessages, (pendingUser)]
```

**ImmutablePrefix.toMessages()** (src/memory/runtime.ts:37)：
- 輸出 `[{ role: "system", content: system }, ...fewShots]`

**AppendOnlyLog.toMessages()** (src/memory/runtime.ts:114)：
- 輸出 `_entries` 的淺拷貝（每輪累積的 user/assistant/tool 訊息）

**VolatileScratch**（src/memory/runtime.ts:123）：
- 存儲 `reasoning`、`planState`、`notes`
- **不直接進入 messages array** — reasoning 經由 `buildAssistantMessage()` 嵌入 assistant message 的 `reasoning_content` 欄位
- 每輪起始在 step() 第 574 行調用 `scratch.reset()`

**最終 messages array** = `system` + `fewShots` + (healed) all prior turns + 可選 `pendingUser`

**出錯路徑**：
- `healLoadedMessages` 不會拋出 — `shrinkOversizedToolResults` 純 map，`fixToolCallPairing` 純遍歷
- `healActiveLogBeforeSend` 中的 `rewriteSession()` 拋出 → catch 吞掉，in-memory heal 依然生效

### A.3 API 調用

**transport 層**：SSE streaming（預設 `stream: true`）或 raw JSON（`stream: false`）

**streaming 路徑**（step() 第 767 行 `if (this.stream)`）：

```
this.client.stream({model, messages, tools, signal, thinking, reasoningEffort})
  (src/client.ts:250)
  │
  ├── waitForChatRateLimit(signal) (src/client.ts:134) — RPM 門控
  ├── fetchWithRetry() (src/retry.ts) — POST ${baseUrl}/chat/completions
  │     ├── body: JSON.stringify(buildPayload(opts, true)) (src/client.ts:153)
  │     │     └── { model, messages, tools, stream:true, extra_body:{thinking:{type}}, reasoning_effort }
  │     └── retryable statuses: [408, 429, 500, 502, 503, 504]
  │
  ├── eventSourceParser (eventsource-parser) 解析 SSE frame
  │     └── 每個 event → queue.push(StreamChunk)
  │           ├── contentDelta / reasoningDelta / toolCallDelta / usage
  │
  └── AsyncGenerator 逐 chunk yield
```

**non-streaming 路徑**（step() 第 893 行 `else`）：

```
this.client.chat({model, messages, tools, signal, thinking, reasoningEffort})
  (src/client.ts:212)
  │
  ├── waitForChatRateLimit()
  ├── fetchWithRetry() — POST, stream:false
  └── 回傳 ChatResponse { content, reasoningContent, toolCalls, usage }
```

**時間限制**：`timeoutMs = 660000`（11 分鐘 — 考慮 DeepSeek load-balancer 的 10 分鐘 keep-alive）

**retry 策略**：只有 initial HTTP fetch 會被 retry。一旦 streaming body 開始傳送，不再 retry（避免重複計費和 context 失同步）。

**出錯路徑**（step() 第 907 行 `catch (err)`）：
- `signal.aborted` → yield `{ role: "done" }`，return（乾淨的 early exit）
- `is5xxError(err) && probeDeepSeekReachable()` → 嘗試 getBalance 判斷伺服器是否可達
- 其他錯誤 → `formatLoopError(err, probe)` (src/loop/errors.ts:8)，針對 401/402/422/400/5xx 分別格式化，yield `{ role: "error" }`，return

### A.4 Tool call 解析

**解析的輸入**：**raw streaming output**，未經 repair pipeline 修改。

- streaming 路徑：累積 `callBuf: Map<number, ToolCall>`，從每個 `chunk.toolCallDelta` 的 `{ index, id, name, argumentsDelta }` 逐步組裝。最終 `toolCalls = [...callBuf.values()]`
- non-streaming 路徑：直接取 `resp.toolCalls`

**解析後**，toolCalls 才進入 repair pipeline（step() 第 985 行）：

```
const { calls: repairedCalls, report } = this.repair.process(
  toolCalls,
  reasoningContent || null,
  assistantContent || null,
);
```

### A.5 Repair pipeline 介入點

```
ToolCallRepair.process() (src/repair/index.ts:53)
```

**Pass 1: Scavenge**（scavenge.ts:20）
- **謂詞**：`reasoningContent || content` 非空
- **輸入**：reasoningContent + content（以 `\n` 合併）
- **輸出**：從兩個來源掃出的額外 ToolCall 陣列
- **Pattern A**：DSML invoke blocks（`<｜DSML｜invoke name="...">`）
- **Pattern B**：raw JSON objects（三種 shape：`{name, arguments}` / OpenAI-style / `{tool_name, tool_args}`）
- **去重**：與 declaredCalls 比對 signature(`name::args`)
- **限制**：`maxCalls = 4`，`MAX_SCAVENGE_INPUT = 100KB`
- **新增**：不重複的 scavenged calls 被 merged 到 declaredCalls 之後

**Pass 2: Truncation repair**（truncation.ts:11）
- **謂詞**：每筆 call 的 `function.arguments` 經 `JSON.parse` 失敗
- **輸入**：原始 arguments string
- **流程**：balancing braces → 補逗號 → 補 dangling key → 關閉 unterminated string → pop remaining open structures
- **輸出**：修復後的 JSON string 或 `{}`（fallback — `fallback: true` 表示不可回復）
- **副作用**：`truncationsFixed++`，notes 記錄修復詳情

**Pass 3: Storm breaker**（storm.ts:33）
- **謂詞**（`StormBreaker.inspect()`）：
  - `isStormExempt(call) === true` → 直接通過（不計入重複檢測）
  - `isMutating(call) === true` → 清除 window 中所有 `readOnly` 的 prior entries（post-edit 的 verify-read 不會被誤判為重複）；mutator entries 仍保留
  - 在 sliding window（`windowSize=6`，可設 `REASONIX_STORM_WINDOW`）內，同 `(name, args)` 達到 `threshold=3`（可設 `REASONIX_STORM_THRESHOLD`）→ `{ suppress: true }`
- **輸入**：merged ToolCall[]（declared + scavenged）
- **輸出**：filtered ToolCall[]（被 suppress 的 calls 被移除）
- **self-correction**（step() 第 1015 行）：如果 `allSuppressed`（全部 tool calls 被 storm breaker 擋掉）且 `!_turnSelfCorrected`，把原始 tool_calls 寫回 assistant message + 插入 stub tool responses → 給 model 一次 self-correct 機會。再次全擋就進入 `forceSummaryAfterIterLimit({ reason: "stuck" })`

**錯誤路徑**：
- Scavenge 在輸入 > 100KB 時直接返回空結果並記錄 note
- Truncation repair 在所有修復手段失敗時返回 `"{}"`（fallback），note 標明不可回復
- Storm breaker 返回 `suppress: true` + reason string

### A.6 Tool 執行

#### (a) parallelSafe 判斷

```
ToolRegistry.isParallelSafe() (src/tools.ts:136)
```

- 讀取 `this._tools.get(name)?.parallelSafe`
- `undefined` / 未設定的 tool 預設為 `false`
- 由 tool 註冊時的 `ToolDefinition` 決定（tools.ts:21）

`CacheFirstLoop` 在第 1128 行讀取：
```
this.tools.isParallelSafe(repairedCalls[callIdx]?.function?.name ?? "")
```

#### (b) 分組策略

分組邏輯在 step() 第 1120-1135 行：

```
while (callIdx < repairedCalls.length) {
  // 收集連續的 parallel-safe calls（最多 parallelMax=3 個）
  if (!dispatchSerial) {
    while (callIdx < repairedCalls.length && chunk.length < parallelMax
           && tools.isParallelSafe(name)) {
      chunk.push(repairedCalls[callIdx++]);
    }
  }
  if (chunk.length === 0) {
    chunk.push(repairedCalls[callIdx++]); // non-parallel-safe 或 serial mode 走這條
  }
  // 並行執行 chunk
  const settled = await Promise.allSettled(chunk.map(c => runOneToolCall(c, signal)));
  // 按原始順序 yield 結果
}
```

**關鍵行為**：
- `parallelMax` 預設 `3`（env `REASONIX_PARALLEL_MAX` 可調，上限 16）
- `env REASONIX_TOOL_DISPATCH=serial` 強制串行
- 依賴關係判斷：**無**。不分組分析 data dependency，只按 parallelSafe flag 連續分組。unsafe call 打斷 chunk 並單獨執行
- 結果順序：`Promise.allSettled` 後按 `chunk` 原始索引迭代，保證 append 順序與 model 宣告順序一致

**出錯路徑**（step() 第 1170-1176 行）：
- `Promise.allSettled` — rejected 的 call 捕獲為 `{ error: "${name}: ${message}" }` JSON string，不終止 chunk

### A.7 結果回注

第 1182-1197 行：

```
this.appendAndPersist({
  role: "tool",
  tool_call_id: call.id ?? "",
  name,
  content: result,       // 可能是實際結果或 { error: ... }
});
```

`appendAndPersist` (loop.ts:269)：
1. `this.log.append(message)` — AppendOnlyLog.push()
2. 如果 `this.sessionName` 存在 → `appendSessionMessage(this.sessionName, message)` — JSONL append

### A.8 VolatileScratch.reset() 的調用位置

```
src/loop.ts:574  -- step() 內，每輪開頭（_turn++ 之後，for loop 之前）
src/loop.ts:310  -- clearLog() 內
src/loop.ts:339  -- switchWorkspace() 內
```

### A.9 循環終止條件

所有終止路徑，均在 step() 的 `for (let iter = 0; ; iter++)` 內：

1. **model 返回 end_turn（無 tool calls）**（第 1054-1061 行）：
   - `repairedCalls.length === 0`（且 `allSuppressed === false`）→ yield `{ role: "done" }` → return

2. **max_turns 限制**：**代碼中無 max_turns 限制**。理論上無限，僅受 context 和 budget 約束

3. **context-guard 觸發 force summary**（第 1094-1109 行）：
   - `decideAfterUsage().kind === "exit-with-summary"` → `forceSummaryAfterIterLimit(summaryContext, { reason: "context-guard" })` → return

4. **storm breaker 全部 suppressed 且已 self-correct**（第 1055-1058 行）：
   - `allSuppressed && this._turnSelfCorrected` → `forceSummaryAfterIterLimit({ reason: "stuck" })` → return

5. **用戶中斷（Esc）**（第 627 行）：
   - `signal.aborted` → yield synthetic assistant message + `{ role: "done" }` → return。重置 `_turnAbort` 為新 controller

6. **API call 中 abort**（第 918 行）：
   - catch block 內 `signal.aborted` → yield `{ role: "done" }` → return。重置 `_turnAbort`

7. **API call 錯誤**（第 930 行）：
   - 非 abort 的其他錯誤 → yield `{ role: "error" }` → return

8. **budget 耗盡**（第 549 行）：
   - step() 入口處檢查 → yield `{ role: "error" }` → return。**注意**：此處 return 後 `_turn` 已遞增但 user message 已透過 `appendAndPersist` 寫入 log（第 622 行），與一般錯誤不同

9. **flash→pro escalation 後續調用**（第 948-973 行）：
   - 檢測到 `<<<NEEDS_PRO>>>` marker → 設定 `_escalateThisTurn = true` → `iter--` → `continue` → 以 pro model 重做本 iter

---

## 任務 B：Compaction 的完整路徑

### B.1 觸發條件

Compaction 由 `ContextManager.decideAfterUsage()` (src/context-manager.ts:107) 決定，在每次 tool call iter 結束後（loop.ts:1065）。

**所有觸發路徑**：

| 條件 | ratio (promptTokens / ctxMax) | Decision kind |
|------|-------------------------------|---------------|
| `ratio > 0.8` | > 80% | `exit-with-summary` |
| `0.7 < ratio ≤ 0.8` (aggressive, 0.8 優先) | 70-80% | `fold` with `tailBudget=ctxMax*0.1` |
| `0.5 < ratio ≤ 0.7` | 50-70% | `fold` with `tailBudget=ctxMax*0.2` |
| `ratio ≤ 0.5` 或 `usage == null` 或 `alreadyFoldedThisTurn` | — | `none` |

**手動觸發**：
- `compactHistory()` (loop.ts:260) 公開方法，可被外部調用
- `mechanicalTruncate()` (context-manager.ts:214) 在 preflight 檢查時調用（`decidePreflight > 0.95` 時）

### B.2 Compaction 如何修改 AppendOnlyLog

`ContextManager.fold()` (src/context-manager.ts:155)：

1. `deps.log.toMessages()` → 全量複製
2. 從尾端向前算 token budget（`keepRecentTokens`，預設 `ctxMax * 0.2` 或 `ctxMax * 0.1`（aggressive））
3. 計算 boundary：累積 token <= tailBudget，且以 user message 為邊界
4. `head = all.slice(0, boundary)` → 送給 `summarizeForFold()` (context-manager.ts:289)
5. 呼叫 `deepseek-v4-flash` 做 non-streaming chat，15 秒 timeout，hardcoded system prompt
6. 提取 head 中的 `<skill-pin>` 區塊（`extractPinnedSkills()`），用 `[skill "name" memo]` 替換原文
7. `summaryMsg = buildAssistantMessage(HISTORY_FOLD_MARKER + summary.content + memoTail, [], model, summary.reasoningContent)`
8. `deps.log.compactInPlace([summaryMsg, ...tail])` → **這是 AppendOnlyLog 的唯一例外**（`compactInPlace` 第 106 行：`this._entries = [...replacement]`）
9. `persistRewrite(replacement)` → `rewriteSession()` 重寫 session 檔案

**關鍵細節**：
- fold summary 使用 **flash** model（成本低），非當前 turn 的 model
- 如果 summary 調用失敗（timeout/abort/error）→ 返回 `{ content: "" }` → fold 不執行
- 如果 head token 佔比 < `HISTORY_FOLD_MIN_SAVINGS_FRACTION` (0.3) → 跳過 fold

### B.3 Compaction 後對 prefix cache 的影響

**代碼中無明確的「cache miss 成本 vs 壓縮節省」的決策邏輯**。沒有權衡計算來判斷是否值得承受 cache miss。

設計取捨的側面證據：
- fold 使用 **flash** model 做 summary，單次調用成本極低（~0.14/M input tokens）
- `mechanicalTruncate`（preflight emergency）純粹丟棄舊訊息，不做 API call
- Context manager 的決策是 **one-sided**：只要 ratio 超標就壓縮，不考慮 cache 成本
- fold 後的 summary message 被插入到 tail 之前，改變了 token 序列 → prefix cache 必然 miss（cache key 包含完整 prompt prefix + messages 的指紋）
- ImmutablePrefix 的 `fingerprint` (runtime.ts:63) 跟蹤 system/tools/fewShots 的變化，但 compaction 不影響 prefix（prefix 是 system + fewShots），只影響 messages 部分 → 只有 `prompt_tokens_cache_miss` 會增加

**實務上的 trade-off**：
- 壓縮節省了大量 context token（可能數百 K tokens）
- 代價是下一輪的 prefix cache miss（需要重新計算）
- 由於 DeepSeek flash 的 1M context window 和極低的 cache miss 定價（$0.14/M vs output $0.28/M），compaction 在大多數場景下淨收益為正
- 代碼層沒有動態判斷「多少 token 值得 cache miss」

**錯誤路徑**：
- `summarizeForFold` 中的 `Promise.race([chat, abortPromise, timeoutPromise])` — abort 或 timeout 都 return `{ content: "", reasoningContent: "" }`
- `persistRewrite` 拋出 → catch 吞掉（in-memory 的 compactInPlace 已生效）

---

## 任務 C：Cost Control 決策路徑

### C.1 flash-first 預設

**配置位置**：

```
CacheFirstLoop 建構子 (src/loop.ts:179)
  this.model = opts.model ?? "deepseek-v4-flash";

CacheFirstLoopOptions 介面 (src/loop.ts:74)
  model?: string;        // default "deepseek-v4-flash"
  autoEscalate?: boolean; // default true

DEEPSEEK_PRICING (src/telemetry/stats.ts:5-13)
  deepseek-v4-flash: { inputCacheHit: 0.0028, inputCacheMiss: 0.14, output: 0.28 }
  deepseek-v4-pro:   { inputCacheHit: 0.003625, inputCacheMiss: 0.435, output: 0.87 }
```

**生效流程**：
1. CacheFirstLoop 沒有指定 model → `"deepseek-v4-flash"`
2. 每次 API 調用使用 `this.modelForCurrentCall()` (loop.ts:390) 決定實際 model
3. 未被 escalation 覆蓋時，返回 `this.model`（即 flash）

### C.2 failure-signal auto-escalation

**被視為 failure signal 的條件及 predicate 位置**：

| 條件 | Predicate | 代碼位置 |
|------|-----------|----------|
| Model 輸出 `<<<NEEDS_PRO>>>` 或 `<<<NEEDS_PRO: reason>>>` marker | `isEscalationRequest(assistantContent)` + `autoEscalate` + `modelForCurrentCall() !== ESCALATION_MODEL` | step() 第 948-952 行 |
| streaming 階段發現 marker | `isEscalationRequest(escalationBuf)` | step() 第 813 行 break |

**不是 failure signal 的項目**（不觸發 escalation）：
- Timeout → 拋錯，回 error event，不自動 escalation
- HTTP 5xx → 拋錯 + probe reachability，不自動 escalation
- Tool call 失敗（`Promise.allSettled` 中的 rejected）→ 作為 `{ error: ... }` 回注，不自動 escalation
- Empty result / model hallucination → 不自動 escalation

**Escalation 後的模型**：`ESCALATION_MODEL = "deepseek-v4-pro"`（loop.ts:56）

**錯誤路徑**：
- `parseEscalationMarker` (escalation.ts:8) 用 regex `^<<<NEEDS_PRO(?::\s*([^>]*))?>>>` 做 anchored match。mid-text 的 `<<<NEEDS_PRO>>>` 不是 escalation request（正常對話內容）
- 如果在 pro 上運作時 model 仍輸出 marker（本不該發生）→ `modelForCurrentCall() === ESCALATION_MODEL` 阻止遞迴 escalation

### C.3 `/pro` 單 turn 提權

**完整路徑**：

1. **用戶觸發**：UI 調用 `loop.armProForNextTurn()` (loop.ts:369)
   - 設定 `_proArmedForNextTurn = true`

2. **生效**：下次 `step()` 入口（loop.ts:586-591）
   ```
   if (this._proArmedForNextTurn) {
     this._escalateThisTurn = true;
     this._proArmedForNextTurn = false;  // 一次性，清除 flag
   }
   ```

3. **費用歸屬**：step() 第 977-980 行
   ```
   const turnStats = this.stats.record(this._turn, this.modelForCurrentCall(), usage);
   ```
   `modelForCurrentCall()` 由於 `_escalateThisTurn` 為 true → 返回 `"deepseek-v4-pro"`

4. **回到 flash**：下次 step() 入口自動重置
   ```
   this._escalateThisTurn = false;  // (loop.ts:584)
   ```
   除非用戶再次 `armProForNextTurn()` 或 model 觸發 auto-escalation

**取消**：`loop.disarmPro()` (loop.ts:374) 在 step() 入口前清除

### C.4 turn-end auto-compaction

觸發條件與任務 B 完全相同（`decideAfterUsage`），關係如下：

- **同一決策點**：auto-compaction 和 turn-end cost control 是同一個 `decideAfterUsage()` 調用
- **不重疊**：`alreadyFoldedThisTurn` guard (context-manager.ts:119) 防止同一 turn 內多次 fold（但 `exit-with-summary` 不受此限）
- **兩階段**：
  1. `kind === "fold"` → 執行 `compactHistory()`（在 turn 內，iter 未結束）
  2. `kind === "exit-with-summary"` → `forceSummaryAfterIterLimit()`（結束 turn）

**成本影響**：
- Compact 的 summary 調用是 flash model，non-streaming，計入 stats（`summarizeForFold` 第 337 行 `this.deps.stats.record(...)`）
- Force summary 也是 flash model，同樣計入 stats（`forceSummaryAfterIterLimit` 第 56 行）

---

## 任務 D：核心文件依賴圖

### src/loop.ts 及其 1-hop imports

```
loop.ts
 ├── client.ts
 ├── core/pause-gate.ts
 ├── hooks.ts
 ├── mcp/registry.ts
 ├── context-manager.ts
 ├── core/inflight.ts
 ├── i18n/index.ts
 ├── loop/errors.ts
 ├── loop/escalation.ts
 ├── loop/force-summary.ts
 ├── loop/healing.ts
 ├── loop/hook-events.ts
 ├── loop/messages.ts
 ├── loop/shrink.ts
 ├── loop/thinking.ts
 ├── loop/types.ts
 ├── memory/runtime.ts
 ├── memory/session.ts
 ├── repair/index.ts
 ├── telemetry/stats.ts
 ├── tools.ts
 └── types.ts
```

### src/context-manager.ts 及其被 import 者

```
context-manager.ts
 ├── client.ts
 ├── client.ts (Usage)
 ├── loop.js (re-export 群)
 │    ├── loop/healing.ts
 │    ├── loop/thinking.ts
 │    └── loop/messages.ts
 ├── mcp/registry.ts
 ├── memory/runtime.ts (AppendOnlyLog — type only)
 ├── memory/session.ts (rewriteSession)
 ├── telemetry/stats.ts
 ├── tokenizer.ts
 └── types.ts (ChatMessage)
```

### src/repair/*.ts 及其被 import 者

```
repair/index.ts
 ├── types.ts (ToolCall)
 ├── repair/scavenge.ts
 ├── repair/storm.ts
 ├── repair/truncation.ts
 └── (exports: flatten.ts 全量)

repair/scavenge.ts
 └── types.ts (ToolCall)

repair/storm.ts
 └── types.ts (ToolCall)

repair/truncation.ts
 └── (無外部 import)

repair/flatten.ts
 └── types.ts (JSONSchema)
```

**誰 import repair/*.ts**：
| Importing file | Imported from |
|---|---|
| src/loop.ts | `repair/index.js` |
| src/tools.ts | `repair/flatten.js` |
| src/index.ts (re-export) | `repair/index.js` |

### src/tools.ts 的調用者

| Importing file | Line |
|---|---|
| src/loop.ts | `import { ToolRegistry } from "./tools.js"` (line 53) |
| src/index.ts | `export { ToolRegistry } from "./tools.js"` (line 82) |
| src/code/setup.ts (推測) | `registerFilesystemTools` 等透過 index.ts 暴露 |

工具實作檔案（`src/tools/*.ts`）各自註冊到 ToolRegistry，由 loop 在建構時建立 registry：`this.tools = opts.tools ?? new ToolRegistry()`。loop.ts 僅使用 `ToolRegistry` 類別和 `isParallelSafe()` 方法。

---

## 驗證命令

以下 grep 命令驗證報告中引用的每個關鍵函數名與行號：

### 任務 A

```bash
grep -n 'async \*step' src/loop.ts           # 539
grep -n 'class CacheFirstLoop' src/loop.ts   # 104
grep -n 'async \*stream' src/client.ts       # 250
grep -n 'reset()' src/memory/runtime.ts      # 128
grep -n 'compactInPlace' src/memory/runtime.ts # 106
grep -n 'buildMessages' src/loop.ts          # 488
grep -n 'isParallelSafe' src/tools.ts        # 136
grep -n 'class StormBreaker' src/repair/storm.ts # 14
grep -n 'inspect(' src/repair/storm.ts       # 33
grep -n 'process(' src/repair/index.ts       # 53
```

### 任務 B

```bash
grep -n 'decideAfterUsage' src/context-manager.ts   # 107
grep -n 'async fold' src/context-manager.ts          # 155
grep -n 'HISTORY_FOLD_THRESHOLD' src/context-manager.ts         # 23 (0.5)
grep -n 'FORCE_SUMMARY_THRESHOLD' src/context-manager.ts       # 33 (0.8)
grep -n 'HISTORY_FOLD_MIN_SAVINGS_FRACTION' src/context-manager.ts # 31 (0.3)
grep -n 'summarizeForFold' src/context-manager.ts   # 289
grep -n 'async compactHistory' src/loop.ts          # 260
```

### 任務 C

```bash
grep -n 'armProForNextTurn' src/loop.ts        # 369
grep -n 'modelForCurrentCall' src/loop.ts      # 390
grep -n 'isEscalationRequest' src/loop/escalation.ts # 16
grep -n 'budgetUsd' src/loop.ts                # 83 (介面), 119 (屬性), 547 (入口檢查)
grep -n 'DEEPSEEK_PRICING' src/telemetry/stats.ts # 5
grep -n 'DEEPSEEK_CONTEXT_TOKENS' src/telemetry/stats.ts # 37
```

### 任務 D

```bash
grep -n 'import.*from.*loop\|from.*context-manager\|from.*repair\|from.*tools' src/loop.ts | head -15
grep -n 'import' src/context-manager.ts | head -10
grep -n 'import' src/repair/index.ts
grep -rn 'from.*tools\.js\|from.*tools/.*\.js' src/loop.ts src/context-manager.ts src/repair/ src/index.ts
```
