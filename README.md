# my-notes

個人筆記與工作論文的公開倉庫。覆蓋 AI 工程架構、認知哲學、組織資本、學習方法論。

## 結構

```
my-notes/
├── archive/              ← 原子筆記（按分類歸檔）
├── reading-notes/        ← arxiv 論文與 repo 的閱讀筆記（67 篇）
├── source-audit/         ← CLI 執行路徑追蹤審計（4 篇）
└── working-papers/
    ├── composed/          ← 合成論文（論證鏈完整，4 篇）
    ├── deliverables/      ← 交付物（源碼審計、事實報告，8 篇）
    └── usage-index.md     ← 消費追蹤
```

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
| [`pi`](source-audit/2026-05-21_Pi-deep-review.md) | Pi — TypeScript agent harness | 598 files, 4 packages | 深層架構審計 |

## working-papers

### composed（合成論文）

- [compile-economy-2026-05-20](working-papers/composed/compile-economy-2026-05-20.md) — 編譯經濟學：智能進步的分層治理框架
- [cognitive-domain-boundary-2026-04-30](working-papers/composed/cognitive-domain-boundary-2026-04-30.md) — 認知領域邊界
- [discard-capability-2026-05-11](working-papers/composed/discard-capability-2026-05-11.md) — 丟棄函數：信息篩選作為智能系統的統一上限
- [externalization-ceiling-2026-04-30](working-papers/composed/externalization-ceiling-2026-04-30.md) — Harness 層投資的結構性貶值

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
