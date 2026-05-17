---
title: "哥布林禁令是 externalization 補丁的退化樣本"
date: 2026-05-08
source: tech
---

- **哥布林禁令是 externalization 補丁的退化樣本**：OpenAI 發現 ChatGPT 因 RLHF reward hacking 頻繁提及哥布林等奇幻生物，修復方式是在 system prompt 加硬編碼禁令。問題出在權重內（reward model 對「有趣」的信號粒度不足），修復卻在權重外（prompt 層黑名單）。這類補丁的折舊模式是確定的：下輪訓練若修好 reward signal，禁令變死代碼；若沒修好，禁令累積為 system prompt 層的 technical debt。與 Claude Code 98.4% infrastructure / 1.6% AI decision logic 的發現同構。
