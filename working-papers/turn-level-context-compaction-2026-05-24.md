> Working Paper · Draft · 2026-05-24
> Proposes Turn-Level Context Compaction as third alternative to full compaction and subagent dispatch for coding agent context management.

---

# Turn-Level Context Compaction：Coding Agent 的第三条路

## 问题

当前主流 coding agent 的 context 管理只有两种策略：

**策略一：全量 compact。** 对话历史膨胀到阈值后，用 LLM 将整段历史压缩为摘要，重新开始。Claude Code、DeepSeek-TUI 均采用此方案。代价是决策链断裂——agent 在 compact 后丢失了"为什么做这个选择"的细节，后续编辑常需要重新 `view` 文件、重新理解上下文。

**策略二：Subagent 分发。** 主 agent 将具体编码任务分发给隔离的子 agent，子 agent 在独立 context 中执行，返回摘要。OpenAI Codex CLI、Hermes Agent 的 `delegate_tool` 均属此类。代价是子 agent 天生缺乏决策链——它不知道主 agent 为什么选择这个方案、之前尝试过什么、项目的整体约束是什么。

两种策略看似对立，实则共享同一个结构性缺陷：**它们都在 context 容量和决策连续性之间做二选一。**

本文提出第三种策略：**Turn-Level Context Compaction**——在同一 session 内，对特定 assistant turn 的代码产出进行选择性替换，保留 prefix cache 命中的同时释放 context 空间。

---

## 第一性原理：Prefix Cache 的工作方式

理解这个方案为什么可行，需要先理解 prompt caching 的核心机制。

主流 LLM API（Anthropic、DeepSeek、OpenAI）的 prompt caching 都是 **exact prefix match**：服务器将 context 的 token 序列从头开始与缓存比对，在第一个不匹配的 token 处截断，之后的部分重新计算。

这意味着一个关键性质：**修改 context 尾部不影响前面的 cache 命中。**

一个典型的 coding agent 对话：

```
[system prompt]              ← cached prefix ✓
[turn 1: user ask]           ← cached prefix ✓
[turn 2: agent plan]         ← cached prefix ✓
[turn 3: user confirm]       ← cached prefix ✓
[turn 4: agent writes 200 lines of code]  ← 这里占了大量 token
[turn 5: tool result: success]
[turn 6: new user request]   ← 新输入
```

Turn 4 的 200 行代码已经写入磁盘。它留在 context 里的唯一作用是让 agent "记住"自己写了什么。但 agent 随时可以通过 `view` 命令重新读取文件——它不需要在 context 里保留逐行代码。

**如果我们在发送下一次 API 请求前，将 turn 4 从完整代码替换为结构化摘要：**

```
[system prompt]              ← cached prefix ✓ (不变)
[turn 1-3]                   ← cached prefix ✓ (不变)
[turn 4: "wrote src/parser.ts: TokenStream class with peek/advance/expect, 
          handles nested brackets, error recovery via panic mode"]
[turn 5: tool result: success]
[turn 6: new user request]
```

Turn 1-3 的 prefix cache 完全命中。Turn 4 变短了，token 序列不同，从 turn 4 开始重新计算——但这段很短（摘要 vs. 200 行代码），计算开销微乎其微。

**净效果：prefix cache 基本保全，context 窗口释放了几百甚至上千 token。**

---

## 实证：独立 L1 推理引擎已经验证了这个思路

这不是纯理论推导。ds4（DwarfStar 4）是 llama.cpp 作者 Georgi Gerganov 开发的独立推理引擎，兼容 DeepSeek 模型格式，实现了完整的三协议服务端（OpenAI / Anthropic / Responses）。审计其源码后，我发现它在 L1 层已经实现了完全相同的 turn-level replacement 思路——不是 DeepSeek 官方的服务端实现，但证明了原理层面的可行性。

ds4 的 `tool_memory` 机制（`ds4_server.c:8024-8114`）做的事情是：

1. Agent 生成 tool call 时，ds4 用 `tool_memory_remember()` 记录 **exact DSML byte sequence**——即模型实际采样出的 token 序列
2. 下一次请求到达时，`tool_memory_attach_to_messages()` 用记录的 exact DSML **替换** 上层 harness 传来的 canonical rendering
3. 因为 token 序列与上次完全一致，KV cache 实现 byte-level prefix match

这本质上就是 turn-level replacement：ds4 不信任上层 harness 的序列化结果（因为 JSON 序列化的 key ordering、whitespace 可能与原始采样不同），所以自己维护了一份"原始产出 → 精确回放"的映射。

区别在于 ds4 是为了 **保护** cache alignment（确保 token 序列精确一致），而本文提出的方案是为了 **释放空间**（用更短的摘要替换冗长产出，接受尾部 cache 重算的小成本）。两者是同一原理的两个方向。

---

## 横向审计：现有框架的改造可行性

我对 6 个开源 coding agent 框架进行了源码级审计，定位了每个框架的 message 组装入口和可改造点。

### Reasonix（DeepSeek 生态 L2 harness）

Reasonix 的三层 context 模型天然适合这个方案：

- **ImmutablePrefix**：system prompt + few-shots，不可变
- **AppendOnlyLog**：对话历史，逐条追加
- **VolatileScratch**：每轮重置的临时区

改造点在 `buildMessages()`（`src/loop.ts:476`）。当前发送路径是 `healActiveLogBeforeSend()` → 合并 `prefix.toMessages()` 与 healed log。只需在 heal 之后、合并之前，对 AppendOnlyLog 中的代码生产型 assistant tool-call 参数做 selective replacement：

```
healActiveLogBeforeSend()
→ [NEW] compactCodeProductions(healedMessages)  // 替换大代码载荷为摘要
→ 合并 [prefix, compactedMessages]  // 当前 user input 已在 step() 开头 append 进 log
```

Reasonix 已经有 `shrinkOversizedToolResults()`（`src/loop/shrink.ts:17`）裁切过长的 tool result。本方案是它的自然延伸——不只裁切 tool result，也裁切 assistant 的代码产出。

当前源码还已有 `shrinkOversizedToolCallArgsByTokens()`（`src/loop/shrink.ts:65`）的雏形和测试；附录中的 `compactCodeProductions()` 可以视为它的语义化版本：只针对已经成功落盘的代码生产型 tool call，保留 tool-call 配对结构，替换大段参数载荷。

额外优势：Reasonix 的 `healActiveLogBeforeSend()` 已经会在 `healLoadedMessages()` / `fixToolCallPairing()` 改写历史 messages 后执行 `compactInPlace` + `rewriteSession`。基础设施是现成的。

### Pi（通用 harness）

Pi 的 session tree 架构（JSONL，linked-list via `parentId`）提供了更精细的控制。每个 entry 是独立节点，可以被单独替换而不影响树结构。

改造点在 `Session.buildSessionContext()`（线性化 session tree 为 message array 的过程中）或 `convertToLlm()`（custom message types → standard roles 的转换层）。Pi 已有 `compactionSummary` 和 `branchSummary` 两种自定义 entry type，增加一个 `codeProductionSummary` type 是架构内的自然扩展。

### DeepSeek-TUI（Rust harness）

TUI 的情况最说明 compact 策略不足的问题。根因分析显示，TUI 的 session 只增不减（`session.rs:173`），每次 API 请求发送 `self.session.messages.clone()`（`turn_loop.rs:1937`），payload 随轮数线性增长。

TUI 有手动 `/compact` 命令，但它是全量 LLM 摘要——丢失所有细节。Turn-level replacement 可以作为 `/compact` 之前的 **渐进式** context 管理：每轮自动替换上一轮的代码产出，延缓触达 context 上限的速度，减少需要全量 compact 的频率。

改造点在 `messages_with_turn_metadata()`（`turn_loop.rs:1937`），在 clone 之后、发送之前做 turn 级别的替换。

---

## 方案规格

### 替换策略

不是所有 assistant turn 都应该被替换。规则：

1. **只替换包含代码产出的 turn**——即执行了 `write`/`create` 类 tool 且 tool result 为 success 的 turn
2. **保留最近 N 轮**（建议 N=2）——agent 可能需要回看刚写的代码做微调
3. **保留包含决策推理的 turn**——如果 assistant message 包含 reasoning/thinking 内容，保留推理部分、只替换代码块

### 摘要格式

摘要不是自然语言总结（那需要额外 LLM 调用）。它是 **结构化元数据**，由 agent 在写代码时同步生成：

```
[Code Production Summary]
file: src/parser.ts
action: created
exports: TokenStream (class), TokenKind (enum)  
key decisions: recursive descent over parser combinator (simpler error recovery)
edge cases handled: nested brackets, unclosed strings
lines: 187
```

这个摘要的生成不需要额外 API 调用——它在 tool execution 阶段就可以从代码内容中提取（AST 分析或 regex），或者要求 agent 在 tool call 参数中附带 `summary` 字段。

### Cache 影响量化

以 DeepSeek V4 Flash 定价为参考（`inputCacheHit: $0.0028/M`，`inputCacheMiss: $0.14/M`）：

- 一段 200 行代码 ≈ 800 tokens
- 替换为摘要 ≈ 80 tokens
- 释放 720 tokens context 空间
- Cache 重算成本：80 tokens × $0.14/M ≈ 可忽略
- 如果这 720 tokens 延迟了一次全量 compact（需要 LLM summary 调用 + 全部 prefix cache 失效），净节省显著

---

## 与现有方案的对比

|  | 全量 Compact | Subagent | Claude Code Microcompact | Turn-Level Compact |
|---|---|---|---|---|
| 决策链保持 | ✗ 全部丢失 | ✗ 子 agent 无决策链 | ✓ 保留 | ✓ 前序 turn 完整保留 |
| Prefix cache | ✗ 全部失效 | ✓ 主 agent 不变 | ✓ `cache_edits` 服务端配合 | ✓ 替换点之前完全命中 |
| Context 释放 | ✓ 大量释放 | ✓ 隔离 context | △ 渐进释放 | △ 渐进释放 |
| 额外 LLM 调用 | 需要（摘要） | 需要（子 agent） | 不需要 | 不需要 |
| 实现复杂度 | 低 | 高（跨 agent 通信） | 中（需服务端 API 支持） | 中（纯 client-side） |
| 适用时机 | context 接近上限 | 独立子任务 | 每 turn 自动 | 每轮自动 |

需要说明的是，Claude Code 已经在生产中实现了类似思路——其分级 compaction 的第一层（microcompaction）会每 turn 自动将老的 tool result 替换为 disk 引用。但它依赖 Anthropic 私有的 `cache_edits` API 在服务端外科手术式编辑已缓存的 prompt，这是一个 provider-specific 的能力。本文提出的方案是 **纯 client-side** 的：不依赖任何服务端特殊 API，只在 message 组装层做结构化替换，可以直接移植到任何使用标准 OpenAI/Anthropic chat completions 接口的 harness。对于 DeepSeek 生态（Reasonix、TUI）以及其他非 Anthropic 的 agent 框架，这是目前唯一不需要服务端配合的渐进式 context 管理路径。

Turn-level compact 不是要取代前述方案，而是填补它们之间的空白：**在 context 尚未饱和时，渐进式地释放空间，延缓全量 compact 的触发，同时保持决策连续性。**

四者可以组合使用：turn-level compact 作为常态策略，subagent 用于真正独立的子任务，全量 compact 作为最后的 fallback。如果 API 端未来支持 segment-based caching，client-side 方案可以进一步与服务端协同。

---

## 开放问题

**1. 摘要质量的下限保证。** Agent 后续要修改之前写的代码时，摘要是否足以让它定位到正确的文件和位置？如果摘要遗漏了关键的边界条件处理，agent 可能在修改时引入 regression。这需要在摘要格式中编码足够的结构信息——不是"写了什么"，而是"做了哪些决策、处理了哪些边界"。

**2. API 端的协同优化空间。** 当前方案在 client 端即可实现，不需要服务端改动。但如果 API 端支持 content-addressable KV cache（而非 exact prefix match），client 可以告诉服务器"这段摘要对应之前那段完整代码的 KV state"，直接复用已有的 KV cache 而不重算。这需要 API 层面的协议扩展——从 prefix-based caching 到 segment-based caching。

**3. 与 thinking/reasoning 的交互。** 带 extended thinking 的模型（DeepSeek R1/V4 Pro、Claude with thinking）在 assistant turn 中包含大量 reasoning token。这些 reasoning 是否也应该被 compact？它们对后续决策的价值可能高于代码本身。

---

## 结论

Turn-level context compaction 是一个从第一性原理可推导的方案：prefix cache 是 exact prefix match，修改尾部不影响前面的 cache，代码产出写入磁盘后在 context 中的边际价值递减。

ds4（独立 L1 推理引擎）的 tool_memory 机制在原理层面验证了 turn-level replacement 的可行性。Reasonix、Pi、DeepSeek-TUI 的现有架构都提供了自然的改造入口。实现不需要 API 端改动，工程量集中在 message 组装层的一个 hook 函数。

这不是一个需要重新设计架构的方案，而是现有 compact 策略的精细化——从"对话级"粒度下沉到"turn 级"粒度。如果 API 端配合支持 segment-based caching，收益可以进一步放大。

更根本地说，当前 agent 社区对 context 管理的优化目标集中在 cache hit rate——这是一个可观测、可量化的指标，但不是最重要的指标。真正影响 agent 任务完成质量的是 **决策连续性**：agent 在第 15 轮修改第 3 轮写的代码时，它是否还记得当初为什么那样写。全量 compact 之后这个信息丢失，agent 只能从摘要猜测，猜错就引入 regression。Claude Code 用 microcompaction 保护决策链，正是基于同样的认识。对于真正追求 agent 任务完成率的团队，优化的重心应该从"cache hit rate"转向"decision continuity"——前者是手段，后者是目的。

---

*本文基于对 ds4、Reasonix、Pi、DeepSeek-TUI、Bub、Hermes Agent 六个开源项目的源码级审计。审计报告涉及的具体函数名、行号均经 grep 交叉验证。*

---

## 附录：Reasonix 的 `compactCodeProductions()` diff sketch

下面的 sketch 基于当前 Reasonix 源码校准：`buildMessages()` 位于 `src/loop.ts:476`，发送前路径是 `healActiveLogBeforeSend()` -> `prefix.toMessages()` + healed log。注意这里不建议把历史 assistant turn 整条替换成普通摘要，因为 Reasonix/DeepSeek 的历史消息必须保持 `assistant.tool_calls` 后面紧跟对应 `tool` message；更稳妥的做法是保留 tool-call 外壳和 `id`，只把已经成功落盘的 `write_file` / `edit_file` / `multi_edit` 参数压缩成结构化摘要。

```diff
diff --git a/src/loop.ts b/src/loop.ts
@@
-import {
-  looksLikeCompleteJson,
-  shrinkOversizedToolCallArgsByTokens,
-  shrinkOversizedToolResults,
-  shrinkOversizedToolResultsByTokens,
-} from "./loop/shrink.js";
+import {
+  compactCodeProductions,
+  looksLikeCompleteJson,
+  shrinkOversizedToolCallArgsByTokens,
+  shrinkOversizedToolResults,
+  shrinkOversizedToolResultsByTokens,
+} from "./loop/shrink.js";
@@
   private buildMessages(): ChatMessage[] {
     const healedMessages = this.healActiveLogBeforeSend();
-    return [...this.prefix.toMessages(), ...healedMessages];
+    const compactedMessages = compactCodeProductions(healedMessages, {
+      keepRecentAssistantTurns: 2,
+      minArgTokens: 800,
+    }).messages;
+    return [...this.prefix.toMessages(), ...compactedMessages];
   }
```

```diff
diff --git a/src/loop/shrink.ts b/src/loop/shrink.ts
@@
+const CODE_PRODUCTION_TOOLS = new Set(["write_file", "edit_file", "multi_edit"]);
+
+export interface CodeProductionCompactionOptions {
+  keepRecentAssistantTurns?: number;
+  minArgTokens?: number;
+}
+
+export function compactCodeProductions(
+  messages: ChatMessage[],
+  opts: CodeProductionCompactionOptions = {},
+): { messages: ChatMessage[]; compactedCount: number; tokensSaved: number } {
+  const keepRecent = opts.keepRecentAssistantTurns ?? 2;
+  const minArgTokens = opts.minArgTokens ?? 800;
+  const assistantTurns = indexAssistantTurns(messages);
+  const protectedAssistants = new Set(assistantTurns.slice(-keepRecent));
+  const successfulToolResults = collectSuccessfulToolResults(messages);
+  let compactedCount = 0;
+  let tokensSaved = 0;
+
+  const out = messages.map((msg, idx) => {
+    if (msg.role !== "assistant") return msg;
+    if (protectedAssistants.has(idx)) return msg;
+    if (!Array.isArray(msg.tool_calls) || msg.tool_calls.length === 0) return msg;
+
+    let changed = false;
+    const tool_calls = msg.tool_calls.map((call) => {
+      const toolName = call.function?.name ?? "";
+      if (!CODE_PRODUCTION_TOOLS.has(toolName)) return call;
+      if (!call.id || !successfulToolResults.has(call.id)) return call;
+
+      const originalArgs = call.function?.arguments ?? "";
+      const beforeTokens = countTokensBounded(originalArgs);
+      if (beforeTokens < minArgTokens) return call;
+
+      const summary = summarizeCodeProduction(toolName, originalArgs, successfulToolResults.get(call.id)!);
+      if (!summary) return call;
+
+      const compactArgs = JSON.stringify(summary);
+      const afterTokens = countTokensBounded(compactArgs);
+      if (afterTokens >= beforeTokens) return call;
+
+      changed = true;
+      compactedCount += 1;
+      tokensSaved += beforeTokens - afterTokens;
+      return { ...call, function: { ...call.function, arguments: compactArgs } };
+    });
+
+    return changed ? { ...msg, tool_calls } : msg;
+  });
+
+  return { messages: out, compactedCount, tokensSaved };
+}
+
+function indexAssistantTurns(messages: ChatMessage[]): number[] {
+  const out: number[] = [];
+  for (let i = 0; i < messages.length; i++) {
+    if (messages[i]?.role === "assistant") out.push(i);
+  }
+  return out;
+}
+
+function collectSuccessfulToolResults(messages: ChatMessage[]): Map<string, string> {
+  const out = new Map<string, string>();
+  for (const msg of messages) {
+    if (msg.role !== "tool" || !msg.tool_call_id) continue;
+    const content = typeof msg.content === "string" ? msg.content : "";
+    if (looksSuccessfulCodeWriteResult(content)) out.set(msg.tool_call_id, content);
+  }
+  return out;
+}
+
+function looksSuccessfulCodeWriteResult(content: string): boolean {
+  if (!content.trim()) return false;
+  try {
+    const parsed = JSON.parse(content) as { error?: unknown; rejectedReason?: unknown };
+    if (parsed && typeof parsed === "object" && (parsed.error || parsed.rejectedReason)) return false;
+  } catch {
+    /* Plain-text success results are normal for write_file/edit_file/multi_edit. */
+  }
+  return /^(wrote|edited|multi_edit: applied)\b/.test(content);
+}
+
+function summarizeCodeProduction(
+  toolName: string,
+  rawArgs: string,
+  toolResult: string,
+): Record<string, unknown> | null {
+  let args: Record<string, unknown>;
+  try {
+    args = JSON.parse(rawArgs) as Record<string, unknown>;
+  } catch {
+    return null;
+  }
+
+  if (toolName === "write_file") {
+    const content = typeof args.content === "string" ? args.content : "";
+    return {
+      compacted: "code-production",
+      tool: "write_file",
+      path: args.path,
+      contentSummary: summarizeTextPayload(content),
+      result: toolResult,
+    };
+  }
+
+  if (toolName === "edit_file") {
+    return {
+      compacted: "code-production",
+      tool: "edit_file",
+      path: args.path,
+      searchSummary: summarizeTextPayload(String(args.search ?? "")),
+      replaceSummary: summarizeTextPayload(String(args.replace ?? "")),
+      result: firstLine(toolResult),
+    };
+  }
+
+  if (toolName === "multi_edit") {
+    const edits = Array.isArray(args.edits) ? args.edits : [];
+    return {
+      compacted: "code-production",
+      tool: "multi_edit",
+      editCount: edits.length,
+      paths: [...new Set(edits.map((e) => e?.path).filter(Boolean))],
+      edits: edits.map((e) => ({
+        path: e?.path,
+        searchSummary: summarizeTextPayload(String(e?.search ?? "")),
+        replaceSummary: summarizeTextPayload(String(e?.replace ?? "")),
+      })),
+      result: firstLine(toolResult),
+    };
+  }
+
+  return null;
+}
+
+function summarizeTextPayload(text: string): Record<string, unknown> {
+  const lines = text.length ? text.split(/\r?\n/) : [];
+  return {
+    chars: text.length,
+    lines: lines.length,
+    head: lines.slice(0, 3).join("\n"),
+    tail: lines.length > 6 ? lines.slice(-3).join("\n") : undefined,
+  };
+}
+
+function firstLine(text: string): string {
+  return text.split(/\r?\n/, 1)[0] ?? "";
+}
```

这个 hook 的性质是 send-time compaction：它不必立刻重写 `AppendOnlyLog`，只要输出是确定性的，第二次及之后发送同一段历史时就能稳定命中 compact 后的 prefix。如果希望 session 文件也变小，可以沿用 `healActiveLogBeforeSend()` 的模式，在 `compactedCount > 0` 时执行 `this.log.compactInPlace(compactedMessages)` + `rewriteSession()`；但这会把“发送前视图压缩”升级为“日志本体压缩”，需要同步更新 `/retry`、session replay、read-before-edit tracker 的语义测试。
