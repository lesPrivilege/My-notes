# Anthropic 銷售主管如何用 Claude Cowork 管理 4000 客戶

- **來源**: https://claude.com/blog/how-an-anthropic-sales-leader-uses-claude-cowork-to-run-a-4-000-account-book
- **日期**: 2026-05-20
- **作者/機構**: Travis Bryant, Head of US Mid-Market GTM at Anthropic
- **類型**: article
- **置信度**: high（官方 blog，第一人稱敘述）
- **模式**: deep-review

## 核心問題

銷售主管的工作本質是判斷：時間投在哪、數字怎麼看。但圍繞判斷的數據工作（從 4 個系統拼資料、每次刷新重新 baseline）佔據了一週大部分時間，壓縮了真正有價值的客戶對話和策略決策。

## 核心方案

1. **Claude Cowork（非 Claude Code）** — 關鍵差異：Travis 試過 Claude Code 但不習慣 terminal。Cowork 提供 GUI 界面 + 自然語言 prompt，讓非技術背景的銷售主管也能用。核心模式是：「用英文描述要做什麼，Cowork 去執行」。

2. **排程化 Skill** — 每日自動檢查 Google Calendar 訂會議室、每次外部會議前自動拉 BigQuery + Salesforce 數據生成客戶簡報。排程器是關鍵：「一旦 prep 不再是 slash command 而是自動運行，我就不會忘記」。

3. **分三層自動化** — 每日（微優化，共 ~90 分鐘）、週五（forecast rollup，省 ~3 小時）、季度（大策略專案，如 4000 客戶打分）。

4. **4000 客戶 propensity scoring** — 定義兩套五維評分 rubric（tech / industries），Claude Cowork 每夜逐個打分（deep web research + Salesforce + BigQuery），產出數字 + 文字 rationale，最後自動生成 interactive dashboard。

5. **精準 prompt 模式** — 非技術 prompt：告訴 Claude 用什麼維度打分 → 跑一個 test territory → 檢查輸出 → 調整權重 → 跑下一個 territory。

## 證據

- 每日微優化節省 ~90 分鐘（分散在多次排程任務）
- 週五 forecast 從手工裝配到單頁 web report，省 ~3 小時/週
- 4000 帳戶 scoring 以前跨 RevOps/FP&A/Marketing 數百工時，Travis 一晚上做完
- Anthropic 官方的 Sales plugin 已提供 baseline skills（call prep 等）

## 風險與弱點

- ⚠️ **這是 Anthropic 員工用自家產品** — 內部 API 權限、數據接入（BigQuery/Salesforce）的順暢度不必然反映外部客戶體驗
- ⚠️ **4000 帳戶 scoring 的 AI 幻覺風險** — 對每個帳戶做 deep web research + 自動評分，評分 rationale 可能包含幻覺資訊，AE 如果盲目信任會誤判 territory
- ⚠️ **缺少失敗案例** — 文章只有成功經驗，沒有 Claude Cowork 搞錯 forecast、排錯會議、給錯評分的反例
- ⚠️ **vendor lock-in 隱憂** — 整個工作流深度綁定 Anthropic 生態（Cowork + BigQuery + Salesforce + internal tools），遷移成本極高
- ⚠️ **規模化依賴安全權限設計** — Cowork 能讀寫 Salesforce、BigQuery、Google Calendar、內部文檔，安全邊界設計是成敗關鍵，但文章完全沒提

## 待驗證問題

- Cowork 的排程任務如何處理下游系統變更（如 Salesforce schema 改動、BigQuery table rename）而不需要 Travis 介入修復？
- 帳戶 scoring 的 rubric 權重調整是手動試錯還是 Cowork 可以從 AE 後續行為（close rate）自我校正？
- 文中提到的「internal documents」是怎麼 feed 給 Cowork 的？需要手動 maintain knowledge base 嗎？
