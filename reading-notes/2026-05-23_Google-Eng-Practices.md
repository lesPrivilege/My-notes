# Google Engineering Practices Documentation

- **来源**: https://github.com/google/eng-practices
- **日期**: 2019-09-04（首次發布），持續更新
- **作者/機構**: Google
- **类型**: repo
- **置信度**: high
- **模式**: quick-summary

## 核心問題

程式碼審查（code review）在大多數團隊中缺乏系統性標準：審查者不知道該看什麼、變更作者不知道怎麼寫好的 CL、審查流程耗時且品質不穩定。

## 核心方案

兩份互相搭配的指南：

1. **The Code Reviewer's Guide** — 審查者的責任：檢查設計正確性、功能性、複雜度、測試覆蓋、命名與註解、程式碼風格（這些應由 linter 自動化而非人工）；知道何時說 LGTM 以及何時 nit（小問題可放行，不阻擋合併）。
2. **The Change Author's Guide** — 開發者的責任：寫好的 CL description（說明 *why* 而非 *what*）、保持 CL 小且聚焦（原則上一份 PR 只做一件事或一個邏輯變更）、收到審查意見時理性討論而非防禦性回應。

## 證據

21k+ stars，Google 內部長期運行的 code review 文化的外部化文件。內容源於業界最有影響力的 code review 實踐之一，無實證 benchmark — 其權威來自 Google 的內部採用歷史。

## 風險與弱點

- ⚠️ 假設組織有充沛的審查人力資源（Google 的 full-time engineer 文化）——小團隊或 startup 可能難以完整套用，尤其「每個 CL 都需要多輪審查」。
- ⚠️ 對「CL 要多小」的指導偏形式化（"smaller is better"），但分散式系統中的跨服務變更往往無法拆小。
- ⚠️ 未討論 code review 的工具生態（GitHub PR vs Gerrit vs Phabricator）對流程的具體影響。

## 待驗證問題

- 對於 AI-assisted 程式碼生成（Copilot、Claude Code 等大量生成 code 的場景），這些 guideline 是否需要調整？Google 內部是否正在更新這些文件來回應 LLM 生成的程式碼審查挑戰？
