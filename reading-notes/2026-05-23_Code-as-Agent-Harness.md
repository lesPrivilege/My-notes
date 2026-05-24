# Code as Agent Harness — Toward Executable, Verifiable, and Stateful Agent Systems

- **來源**: https://arxiv.org/abs/2605.18747
- **日期**: 18 May 2026
- **作者/機構**: Xuying Ning et al. (42 authors), UIUC / Meta / Stanford
- **类型**: paper (survey)
- **置信度**: high
- **模式**: deep-review

## 核心問題

Code 在 agentic system 中的角色正在從「target output」（LLM 要生成的產物）擴展為 operational substrate——agent 用它來推理、行動、建模環境、驗證進度。現有 survey 通常只把 code 當作 LLM 的輸出產品，缺乏統一視角來理解 code 如何作為 agent 整個執行基礎設施的組織中心。

## 核心方案

提出 **Code as Agent Harness** 統一框架，圍繞三層組織：

1. **Harness Interface** — Code 作為 agent 推理（program-delegated reasoning / 形式驗證 / iterative code-grounded reasoning）、行動（skill selection / programmatic policy / lifelong code-based agents）、環境建模（structured world representations / execution-trace modeling / code-grounded evaluation）的介面。

2. **Harness Mechanisms** — 長 horizon 執行的內部機制：planning（linear decomposition / structure-grounded / search-based / orchestration）、memory & context engineering（working / semantic / experiential / long-term / multi-agent / compaction）、tool use（function-oriented / environment-interaction / verification-driven / workflow-orchestration）、control loop（Plan-Execute-Verify with sandboxed execution + deterministic sensors）、adaptive optimization（deep telemetry → evolution agent → governed harness mutation）。

3. **Scaling the Harness** — 單 agent → 多 agent：共享 code artifacts（repository / execution trace / blackboard）作為多 agent coordination 的基底；functional role specialization（synthesis / understanding / verification / execution / planning agents）；shared-harness synchronization（parallel branches + merge / hierarchical memory / agent pool scaling）。

論文還提出了關鍵的 **The Central Gap**：現有系統在 shared harness representation 上存在 gap——從 implicit/file-only 到 repository-based 到 execution-based 到 blackboard/shared-state，缺乏統一的 formal shared-state abstraction。

## 證據

survey paper，無新實驗。核心貢獻在於 taxonomy + 統一框架 + open challenges 的系統化整理。附帶 awesome list（GitHub: YennNing/Awesome-Code-as-Agent-Harness-Papers）。

Coverage 範圍：
- Code assistants（從 inline completion → autonomous SWE agents → multi-agent code assistance）
- GUI/OS automation（program world as partially observable, code as bridge）
- Embodied agents（code as control boundary, layered harness）
- Scientific discovery（self-driving labs as executable feedback loops）
- Personalization & recommendation（preference state as editable artifact）

## 風險與弱點

- ⚠️ **Survey 的固有局限** — 框架是描述性而非規範性的。taxonomy 覆蓋廣泛，但未能給出在不同場景下應該選擇哪種 harness architecture 的 principled guidance。
- ⚠️ **The Central Gap 只被識別未被解決** — 論文正確指出了 shared-state abstraction 的缺失，但沒有提出具體 architecture 來填補這個 gap。
- ⚠️ **Self-evolving harness 的 regression 問題被列為 open challenge 但討論不足** — §5.2.3 提到的 regression-free harness improvement 是實務上最頭痛的問題，論文只點到為止。
- ⚠️ **42 位作者的協調成本反映在 depth 不均** — harness interface 和 mechanism 部分非常詳細（各小節都有具體方法 mapping），但 multi-agent scaling 部分相對抽象，尤其是 shared-harness synchronization 的具體機制描述不足。

## 待驗證問題

- 這個 framework 與 ActiveGraph（arXiv:2605.21997）的 event-sourced log-as-agent 設計之間是什麼關係？ActiveGraph 的 append-only log + projection 恰好是 The Central Gap（shared-state abstraction）的一個具體實現——兩篇論文在同一天 arXiv 上出現，是巧合還是同一個 emergent insight？
- Harness engineering 是否正在成為一個獨立子領域（open problem §5.2.7 "Toward a Science of Harness Engineering"）？如果 harness 的可靠性才是 agent 自主性的真正瓶頸而不是模型能力，那目前的評測標準（task success rate）是否從根本上就不對？
- "Agent-initiated code artifacts" 與 system-provided harness 之間的邊界在哪裡？當 agent 自己生成一個 tool 並在後續步驟中使用，這個 tool 應該被視為 harness 還是 agent 的產出？這個邊界模糊性可能是 harness engineering 的核心 tension。
