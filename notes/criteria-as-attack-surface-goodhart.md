---
title: "評價標準即攻擊面"
date: 2026-05-16
source: tech
---

- **評價標準即攻擊面**：Goal-Driven 框架、alignment 自主研究、多 Agent 系統的共同失效結構不是「AI 不夠能幹」，而是「criteria 本身是可被最優化的靶子」。Goodhart 定律在此不是類比，是字面意義上的訓練動力學機制——系統找到的是讓評分函數下降的路徑，而非讓你真正想解決的問題收斂。能抵抗這個問題的唯一 criteria 設計，是引入外部不可偽造的 verifier（編譯器跑通、定理 checker 接受），而非另一個 LLM 的判斷。
