# Harness Engineering · 源码审计 · 合成论文

围绕 AI 工程架构、Harness Engineering、agent runtime 与分布式智能系统的长期判断与源码验证。
材料来自个人 Obsidian 长期笔记 archive，论证置于 working papers，证据追溯至 source audits。

本仓库同时承载当前本地使用的 skills：`skills/` 以本机 `~/.pi/agent/skills` 的实际工作树为准。笔记仍保留在仓库根目录，避免在本地建立第二个 project；`sync-my-notes.sh` 只同步笔记目录，不会自动暂存或覆盖 skills。

---

## 先读这里

首次进入，从这三篇判断 + 一份架构地图开始：

- [**Harness 投资与上游优势**](working-papers/deliverables/03-Harness-investment-and-upstream-advantage.txt) — 外化天花板、harness 折旧、模型方做 harness 的结构性收益。理解 Harness Engineering 的核心不对称。
- [**Claude Code Harness 源码复盘**](working-papers/deliverables/01-Claude-Code-Harness-Engineering-review.txt) — Query Loop 心跳、Context Governance 分层、工具权限与恢复机制的源码级追踪。
- [**DeepSeek 开源 Harness 生态考察**](working-papers/deliverables/04-DeepSeek-open-source-harness-ecosystem-review.txt) — DeepSeek-TUI、Reasonix、ds4 三个项目的架构定位与工程取舍。
- [**四项目横向架构比较**](source-audit/2026-05-21_cross-project-architecture-comparison.md) — TUI + Reasonix + Pi + ds4 在 Runtime Loop、Context 传递、工具编排上的系统对照。

## 证据地图

| 层 | 内容 | 目录 |
|---|---|---|
| **Working Papers** | 每篇论证一个独立命题的长期判断 | [`working-papers/`](working-papers/) |
| **Source Audits** | 逐文件逐路径的源码审计与执行路径追踪 | [`source-audit/`](source-audit/) |
| **Archive** | 按主题归并的原子笔记，commit 可追溯时间戳 | [`archive/`](archive/) |

判断写在 working papers 里，判断的依据在 source audits 里，判断的原料在 archive 里。

---

以下为各分类的完整清单。

## archive

| 分類 | 條數 | 文件 |
|------|------|------|
| 技術與系統 | 179 | [`archive/tech.md`](archive/tech.md) |
| 認知、哲學與心智 | 91 | [`archive/mind.md`](archive/mind.md) |
| 組織、資本與權力 | 65 | [`archive/power.md`](archive/power.md) |
| 數學、學習與方法論 | 38 | [`archive/method.md`](archive/method.md) |
| 生活、決策與心智工具 | 35 | [`archive/life.md`](archive/life.md) |

## source-audit

| 報告 | 專案 | 規模 | 類型 |
|------|------|------|------|
| [`reasonix`](source-audit/2026-05-21_reasonix-execution-path.md) | Reasonix — TypeScript coding agent | 294 files | 執行路徑追蹤 |
| [`ds4`](source-audit/2026-05-21_ds4-execution-path.md) | DwarfStar 4 — C inference engine | 15,581 lines | 執行路徑追蹤 |
| [`deepseek-tui`](source-audit/2026-05-21_deepseek-tui-execution-path.md) | DeepSeek-TUI — Rust coding agent | 14 crates | 執行路徑追蹤 |
| [`deepseek-tui-root-cause`](source-audit/2026-05-22_deepseek-tui-root-cause-analysis.md) | DeepSeek-TUI 口碑根因 | 209K lines | 源碼根因分析 |
| [`pi`](source-audit/2026-05-21_Pi-deep-review.md) | Pi — TypeScript agent harness | 598 files, 4 packages | 深層架構審計 |
| [`bub`](source-audit/2026-05-22_bub-execution-path.md) | Bub — Python agent runtime | ~3,300 lines | 執行路徑追蹤 |
| [`hermes`](source-audit/2026-05-21_Hermes-Agent-Source-Audit.md) | Hermes Agent — Python agent | v0.14.0 | 源碼審計 |
| [`openclaw`](source-audit/2026-05-21_OpenClaw-Source-Audit.md) | OpenClaw — TS agent platform | 18,019 files | 源碼審計 |
| [`cross-project`](source-audit/2026-05-21_cross-project-architecture-comparison.md) | TUI + Reasonix + Pi + ds4 | 4 repos | 架構比較 |
| [`reasonix-claim`](source-audit/2026-05-18_Reasonix-Audit.md) | Reasonix — TypeScript | v0.43.0 | 聲稱驗證 |
| [`ds4-claim`](source-audit/2026-05-18_DS4-Audit.md) | DwarfStar 4 — C | alpha | 聲稱驗證 |
| [`deepseek-tui-claim`](source-audit/2026-05-18_DeepSeek-TUI-Audit.md) | DeepSeek-TUI — Rust | v0.8.39 | 聲稱驗證 |
| [`cli-prompts`](source-audit/cli-review-prompts.md) | Audit prompt templates | 4 prompts | 技術文件 |

## working-papers

### composed（合成論文）

- [compile-economy-2026-05-20](working-papers/composed/compile-economy-2026-05-20.md) — 編譯經濟學：智能進步的分層治理框架
- [cognitive-domain-boundary-2026-04-30](working-papers/composed/cognitive-domain-boundary-2026-04-30.md) — 認知領域邊界
- [discard-capability-2026-05-11](working-papers/composed/discard-capability-2026-05-11.md) — 丟棄函數：信息篩選作為智能系統的統一上限
- [externalization-ceiling-2026-04-30](working-papers/composed/externalization-ceiling-2026-04-30.md) — Harness 層投資的結構性貶值
- [turn-level-context-compaction-2026-05-24](working-papers/composed/turn-level-context-compaction-2026-05-24.md) — Turn-Level Context Compaction：Coding Agent 的第三條路

### deliverables（交付物）

- [00-cover-letter](working-papers/deliverables/00-cover-letter-Harness-Engineering-observations.txt) — Harness Engineering 觀察總論
- [01-Claude-Code-review](working-papers/deliverables/01-Claude-Code-Harness-Engineering-review.txt) — Claude Code Harness 架構源碼回顧
- [02-Mnemos-review](working-papers/deliverables/02-Mnemos-architecture-review.txt) — Mnemos 架構回顧
- [03-Harness-investment](working-papers/deliverables/03-Harness-investment-and-upstream-advantage.txt) — Harness 投資與上游優勢
- [04-DeepSeek-ecosystem](working-papers/deliverables/04-DeepSeek-open-source-harness-ecosystem-review.txt) — DeepSeek 開源 Harness 生態考察
- [audit-deepseek-tui](working-papers/deliverables/audit-deepseek-tui.md) — DeepSeek-TUI 源碼審計 (v0.8.39)
- [audit-ds4](working-papers/deliverables/audit-ds4.md) — ds4 源碼審計
- [audit-reasonix](working-papers/deliverables/audit-reasonix.md) — Reasonix 源碼審計

## 更新

由 `~/Scripts/sync-my-notes.sh` 增量同步：

- **archive/** — 從 Obsidian 歸檔同步，全量推送
- **reading-notes/** — 直接在本 repo 新增，全量推送
- **working-papers/composed/** — 從 Obsidian 歸檔同步，增量推送
- **working-papers/deliverables/** — 從 delivery 項目拷貝，手動推
