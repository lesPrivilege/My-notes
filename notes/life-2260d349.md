---
title: "API key 泄露是操作流程缺脫敏步驟的問題"
date: 2026-04-30
source: life
---

- **API key 泄露是操作流程缺脫敏步驟的問題**：Agent 的思考過程會把環境變量裡的敏感信息寫進輸出（debug API 連接時自然會讀 env 並打印）。風險不在本地 session 讀到 key，而在把 agent 日志原樣貼到雲端對話。防護措施：貼日志前 grep 敏感前綴（sk-、tp-）；代碼從環境變量讀 key 不硬編碼；.gitignore 排除 .env 和日志文件。
