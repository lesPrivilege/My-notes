# my-notes

個人筆記與工作論文的公開倉庫。覆蓋 AI 工程架構、認知哲學、組織資本、學習方法論。

## 結構

```
my-notes/
├── CURATION.md           ← 精選原子筆記索引
├── archive/              ← 原子筆記（按分類歸檔）
│   ├── tech.md           ← 技術與系統（179 entries）
│   ├── mind.md           ← 認知、哲學與心智（91 entries）
│   ├── life.md           ← 生活、決策與心智工具（35 entries）
│   ├── power.md          ← 組織、資本與權力（65 entries）
│   └── method.md         ← 數學、學習與方法論（38 entries）
├── reading-notes/        ← arxiv 論文與 repo 的閱讀筆記（67 篇）
└── working-papers/       ← 工作論文（組裝完成的連續論證，12 篇）
```

## 更新

由 `~/Scripts/sync-my-notes.sh` 增量同步：

- **archive/** — 從 Obsidian 歸檔同步，全量推送
- **CURATION.md** — 精選清單，手動維護後推送
- **reading-notes/** — 直接在本 repo 新增，全量推送
- **working-papers/** — 從 Obsidian 歸檔同步，增量推送

## working-papers

- [compile-economy-2026-05-20](working-papers/compile-economy-2026-05-20.md) — 編譯經濟學：智能進步的分層治理框架
- [externalization-ceiling-2026-04-30](working-papers/externalization-ceiling-2026-04-30.md) — Harness 層投資的結構性貶值
- [discard-capability-2026-05-11](working-papers/discard-capability-2026-05-11.md) — 丟棄函數：信息篩選作為智能系統的統一上限
- [00-cover-letter](working-papers/00-cover-letter-Harness-Engineering-observations.txt) — Harness Engineering 觀察總論
- [01-Claude-Code-review](working-papers/01-Claude-Code-Harness-Engineering-review.txt) — Claude Code Harness 架構源碼回顧
- [02-Mnemos-review](working-papers/02-Mnemos-architecture-review.txt) — Mnemos 架構回顧
- [03-Harness-investment](working-papers/03-Harness-investment-and-upstream-advantage.txt) — Harness 投資與上游優勢
- [04-DeepSeek-ecosystem](working-papers/04-DeepSeek-open-source-harness-ecosystem-review.txt) — DeepSeek 開源 Harness 生態考察
- [audit-deepseek-tui](working-papers/audit-deepseek-tui.md) — DeepSeek-TUI 源碼審計 (v0.8.39)
- [audit-ds4](working-papers/audit-ds4.md) — ds4 源碼審計
- [audit-reasonix](working-papers/audit-reasonix.md) — Reasonix 源碼審計
