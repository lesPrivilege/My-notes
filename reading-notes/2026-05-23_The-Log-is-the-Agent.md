# The Log is the Agent: Event-Sourced Reactive Graphs for Auditable, Forkable Agentic Systems

- **来源**: https://arxiv.org/abs/2605.21997
- **日期**: 21 May 2026
- **作者/机构**: Yohei Nakajima, Untapped Capital / activegraph.ai
- **类型**: paper
- **置信度**: high
- **模式**: deep-review

## 核心问题

主流 Agent 框架围绕 LLM 构建：对话循环优先，工具、规则、日志层层叠加，"memory" 层作为检索式附加组件丢进去。这种架构下状态不可重建、不可重放、不可分叉——log 是 exhaust（产物），不是 substrate（基底）。ActiveGraph 反轉這個關係：log 才是 agent 本身。

## 核心方案

1. **Event Sourcing 基底** — 所有状态变化是 append-only log 中的事件：goal 创建、object 产生、behavior 启动/完成、LLM 调用、tool 调用、关系建立等。没有"内存"，没有可变全局状态。graph 是 log 的 deterministic projection（fold）。

2. **Behavior as Reaction** — 没有编排脚本 / workflow。Behavior 声明 subscription（事件类型 + 图结构 pattern），当 graph 变化时 runtime 触发 body，body 可以调 LLM / tool / 创建对象 → 新事件 → 可能触发更多 behavior。控制流是 emergent 的。

3. **Deterministic Replay** — LLM 调用不防呆，但首次运行时记录 response（content-addressed cache，按 prompt hash 索引）；重放时命中 cache，不调模型。两种模式：permissive（event-by-event 重放，miss 的 hash 重新调用）和 strict（逐事件对比，divergence 报错）。strict 通过 = 该 run 可复现的证明。

4. **Forking + Structural Diff** — 在任意 event 处分支：共享前缀从 cache 回放（免费），只在新路径执行模型调用。两个 fork 之间可以做结构化 diff（哪些 object / relation / patch 不同）。附赠轻量级 in-run primitive `frame` 用于短生命周期子上下文。

5. **Relation-Behavior** — typed edge 本身可以携带逻辑。不仅仅是 object 响应变化，typing 关联这一动作也可以触发计算。

## 证据

论文以 diligence pack（尽职调查 pack）作为可复现的 demo：

- 3 家公司（Northwind Robotics, Stellar Logistics, Pinecone Bio），全部用 recorded fixtures 离线执行
- 671 events → 93 objects（3 companies, 24 questions, 9 documents, 25 claims, 25 evidence, 1 contradiction, 3 risks, 3 memos）+ 76 relations
- 103 次 LLM 调用 + 48 次 tool 调用，无编排代码
- `pip install activegraph && activegraph quickstart` 即可复现，byte-identical
- 每个 claim 携带 provenance block（created_by behavior + caused_by_event），通过 typed relation 追溯到 question / document / evidence

论文明确声明不做 task performance 评测（"a systems paper, not empirical claims about task performance"）。

## 风险与弱点

- ⚠️ **Determinism Contract 无静态保证** — behavior 作者需要遵守规则（不读随机数、不读 wall clock、不做 framework 外的 I/O），违反者首次运行正常、重放时才暴露（strict replay divergence error）。动态执行成本转移到了 debug 阶段。
- ⚠️ **Replay 成本随 log 线性增长** — 百万级 event 的 run 目前需要完整重放。无 checkpoint / compaction 策略。
- ⚠️ **无并发 / 分布式写者模型** — 单 run 的 append-only log 顺序良好定义，但多 agent 争用共享 graph、分布式 writer 的 ordering 问题未解决。
- ⚠️ **外部工具副作用** — tool 的 response recording 只保证重放时确定，首次执行的真实外部副作用仍然存在。这是 recording 的固有局限。
- ⚠️ **Schema Evolution 是实际运维负担** — event / object type 变更后，老 shape 的事件在 disk 上，需要 migration tooling。
- ⚠️ **Loop / Divergence 防护粗糙** — per-run budget（events、behavior calls、model calls、recursion depth、time 上限）是 blunt instrument，非静态终止保证。
- ⚠️ **无大规模实证** — 论文诚实地标注了这一点。architecture 的认知优越性在 paper 层面是 plausible 的，但能否在真实长尾 agent 任务中转化为 measurable advantage 未经验证。

## 待验证问题

- deterministic cache 在 fork 改变 prompt 后需要重执行 affected calls，这个"affected"范围如何界定？是仅 re-run hash-miss 的 LLM 调用，还是整个 cascade？
- relation-behavior 在图的规模增长后（如千级 node/edge 时）的 subscription 匹配性能如何？Cypher subset 的实现是否存在 N+1 问题？
- Self-improving agent（§7）论点是这篇文章最有趣的 implication，但 paper 明确标注未实现。fork-and-diff 作为 improvement primitive 的落地路径——proposal → fork → eval → structural diff——在实践中如何防止 evaluation 循环不崩（agent 改自己 prompt 然后改坏了检测器）？
- BabyAGI lineage 中的 prior graph-memory research 具体指什么？§8.1 提到"layered memory architectures repeatedly ran up against the absence of a single authoritative history to project from"——这个 claim 是否有对应实证或某篇具体工作的分析？
