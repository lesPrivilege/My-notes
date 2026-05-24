# Understand Anything — Interactive Codebase Knowledge Graph

- **來源**: https://github.com/Lum1104/Understand-Anything
- **日期**: 2026-03-15（首次發布），活躍中
- **作者/機構**: Lum1104
- **类型**: repo
- **置信度**: high
- **模式**: deep-review

## 核心問題

進入一個陌生的大型 codebase（20 萬行起跳）時，沒有結構化的理解入口。傳統方式只能逐文件閱讀、仰賴文件或問同事。

## 核心方案

Claude Code plugin / CLI tool，用 multi-agent pipeline 分析專案，產出互動式 knowledge graph + dashboard：

**Multi-Agent Pipeline（6+1 agents）：**
1. `project-scanner` — 發現檔案、檢測語言與 framework
2. `file-analyzer` — 提取 function、class、import，產出 graph nodes/edges（平行執行，最多 5 concurrent，20-30 files/batch）
3. `architecture-analyzer` — 識別架構層（API / Service / Data / UI / Utility）
4. `tour-builder` — 按依賴順序生成引導式學習路線
5. `graph-reviewer` — 驗證 graph 完整性
6. `domain-analyzer` — 提取業務域、流程、步驟（`/understand-domain`）
7. `article-analyzer` — 從 wiki 文章中提取 entity、claim、隱含關係（`/understand-knowledge`）

**雙層 graph 設計：** 結構層（import、function call 等靜態分析 → 確定性）與語義層（LLM 產生意圖描述）。支援 incremental 更新（只 re-analyze 變更檔案）。

**Dashboard 功能：** 互動式 knowledge graph、模糊與語意搜索、diff impact analysis（變更波及範圍）、persona-adaptive UI（junior dev / PM / power user 顯示不同細節）、layer visualization、12 種語言概念（generics/closures/decorators 等在出現位置 inline 解釋）。

## 證據

22.4k stars，2026-03 發布後兩個月內爆發式增長。支援 Claude Code、Codex、Cursor、Copilot、Gemini CLI、OpenCode 等所有主流 coding agent。有 live demo 與 Discord 社區。

## 風險與弱點

- ⚠️ **LLM-dependent semantic layer 的穩定性** — 語義節點依賴 LLM 生成描述，不同模型／溫度可能產生不一致的 graph 輸出。確定性只保證在結構層（import graph），語義層的 reproduability 未說明。
- ⚠️ **大專案的實際 performance** — 百萬行級 monorepo 的 5-concurrent parallel scan 在實務上需要多久？README 未給 benchmark 數據。
- ⚠️ **CI/CD 整合深度不足** — post-commit hook 有提到，但在 PR review 流程中如何讓 graph 隨 codebase 自動演化、如何讓 review 者直接透過 diff impact graph 做 review，還是開放問題。
- ⚠️ **知識庫（wiki/articles）分析模式尚淺** — 宣稱支援 Karpathy-pattern LLM wiki，但 entity/claim extraction 的品質和 recall 未報告。

## 待驗證問題

- 與 repo 內現有文件生成工具（如 Typedoc、JSDoc、obvious 的 architecture.md）的關係是取代還是補充？如果 graph 是 live 的，傳統文件是否就沒有存在的必要了？
- 這個工具的反向問題更有趣：如果 agent 在 coding 時已經透過 knowledge graph 理解 codebase，那 graph 本身是否可以作為 agent 的 persistent memory 來用？這跟 ActiveGraph 的 event-sourced graph 思路形成一個有意思的對稱：ActiveGraph 把 log 變 graph，這裡把 codebase 變 graph。
