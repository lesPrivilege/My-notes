---
title: "MSM是外化策略的內化，不是表徵突破"
date: 2026-05-16
source: tech
---

- **MSM是外化策略的內化，不是表徵突破**：Model Spec Midtraining在pretrain和AFT之間插入一個階段，用合成文檔把Model Spec的內容和理由燒進權重。核心發現是同樣的fine-tuning數據在不同MSM spec下泛化到完全不同的價值方向。但這不改變外化天花板的判斷——模型並未自己生成更好的spec，spec的質量仍然是外部提供的。MSM的真正意義是把spec設計從philosophical argument變成empirical measurement，是「Model Spec Science」的工具化第一步。高計算量CoT AFT能收斂到MSM效果，進一步說明MSM提供的是更好的初始化，而非新的representational capability。
