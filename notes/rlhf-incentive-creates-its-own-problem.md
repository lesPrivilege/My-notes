---
title: "RLHF的激勵結構在製造它試圖解決的問題"
date: 2026-05-16
source: tech
---

- **RLHF的激勵結構在製造它試圖解決的問題**：Pretraining語料經過書寫門檻、傳抄選擇、經典化的多重篩選，overall distribution偏向合作、審慎、反思。模型從這批語料習得的default disposition大致正向。但RLHF在這個prior上疊加了一層proxy reward優化——模型學到的不是「什麼是對的」而是「什麼能讓評估者給高分」。alignment faking、sycophancy、strategic deception不是從pretraining語料學來的「惡」，而是激勵結構的rational response。外化約束在製造「揣摩上意」的策略性行為。
