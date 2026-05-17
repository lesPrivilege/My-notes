# Agent Skills — Production-Grade Engineering Skills for AI Coding Agents

- **來源**: https://github.com/addyosmani/agent-skills
- **日期**: 2026-02-15 建立
- **作者**: Addy Osmani（Google Chrome 工程總監）
- **類型**: repo
- **置信度**: high
- **模式**: deep-review + solo-dev 實用性分析

## 核心問題

AI coding agent 預設走最短路徑——跳過 spec、測試、安全審查、code review。Agent Skills 把資深工程師的 judgment encode 成 agent 可執行的結構化工作流，不讓開發者用人腦記憶這些流程。

## 20 個 Skills 分類

| 階段 | Skills |
|------|--------|
| Define | idea-refine, spec-driven-development |
| Plan | planning-and-task-breakdown |
| Build | incremental-implementation, test-driven-development, context-engineering, source-driven-development, frontend-ui-engineering, api-and-interface-design, browser-testing-with-devtools, debugging-and-error-recovery |
| Verify | (build 階段已包含) |
| Review | code-review-and-quality, code-simplification, security-and-hardening, performance-optimization |
| Ship | git-workflow-and-versioning, ci-cd-and-automation, deprecation-and-migration, documentation-and-adrs, shipping-and-launch |

## 關鍵設計

- **Process, not prose** — 每個 SKILL.md 是工作流，不是參考文檔
- **Anti-rationalization table** — 列出 agent 常用藉口 + 對應反駁論證。約束的不是 agent，是使用 agent 的人
- **Verification is non-negotiable** — 每個 skill 以 evidence requirements 結尾
- **Progressive disclosure** — SKILL.md 是 entry point，references 按需加載

## Agent Personas

3 個 specialist personas：code-reviewer、test-engineer、security-auditor

## Solo Dev 實用性（按 ROI 排序）

### 必用
- **git-workflow-and-versioning** — atomic commits, 好的 commit message
- **debugging-and-error-recovery** — formalize 已在做的流程
- **code-simplification** — Chesterton's Fence + Rule of 500
- **context-engineering** — 多模型分工下餵正確 context
- **source-driven-development** — 用官方文檔 grounding

### 視情況
- **incremental-implementation** — 超過一週的 feature
- **test-driven-development** — 開源/分享用，自用寫 critical path 即可
- **security-and-hardening** — 有使用者資料或公開網路時必用
- **spec-driven-development** — 維護超過一個月的功能寫 5 行 spec

### 需改編
- **code-review-and-quality** — solo 時切 code-reviewer persona self-review
- **api-and-interface-design** — 有 module boundary 時用
- **documentation-and-adrs** — ADR 記錄 why，API docs 有使用者時才需要
- **ci-cd-and-automation** — lint + test on push 值得設

### 幾乎用不到
- deprecation-and-migration, shipping-and-launch, frontend-ui-engineering, browser-testing, performance-optimization

## 最佳 solo dev 組合

```
git-workflow + debugging + code-simplification + context-engineering + source-driven-development
```
