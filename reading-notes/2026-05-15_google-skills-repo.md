# google/skills — Agent Skills for Google Products

- **来源**: github.com/google/skills
- **日期**: 2026-03-31（created），活躍維護中
- **作者/机构**: Google
- **类型**: repo
- **置信度**: high

## 核心問題

Agent 開發者需要領域特定的知識與操作指南才能有效使用 Google 產品。此 repo 以標準化 SKILL.md 格式封裝這些知識，讓 agent 可以直接 consume 並執行。

## 核心方案

1. **Standardized skill format** — 每個 skill 一個目錄：`SKILL.md`（frontmatter: name/description/compatibility → 核心指令）+ `references/` 子目錄（CLI、client library、MCP、IaC、IAM 等參考文件）。
2. **`npx skills add google/skills`** 安裝 — 透過 skills.sh / Agent Skills 生態系分發。
3. **Recipe 模式** — 除基礎 skills 外有跨產品 recipe（onboarding、auth、networking observability）。

## Repo Stats

- ⭐ 8,246 stars | 633 forks | 21 open issues
- License: Apache 2.0
- 13 個 skills：Gemini API、AlloyDB、BigQuery、Cloud Run、Cloud SQL、Firebase、GKE、3 個 Recipe、3 個 WAF

## 設計觀察

- SKILL.md + references/ 分離：separation of concerns 比 monolithic 格式好。
- Gemini API skill 顯著更詳細（SDK 初始化/認證指引佔大量篇幅）。
- GKE skill 附實際 YAML assets（`default-deny-netpol.yaml` 等）。
- 明確標記 "Vertex AI → Agent Platform" rebranding，要求使用新 Gen AI SDK。

## 風險與弱點

- ⚠️ Google 產品 rebranding 頻繁，skills 需持續維護。
- ⚠️ Recipe skills 偏 thin（無完整 references/）。
- ⚠️ 依賴 agentskills.io 生態系，platform risk。
- ⚠️ 僅涵蓋 GCP 產品，無 Workspace/Android/Chrome 等。

## 待驗證問題

- SKILL.md 在 agent workflow 中的實際 routing 機制？
- Skills 的版本管理與 outdate detection？
- 與 Anthropic/OAI 的 skills 生態的 interoperability？
