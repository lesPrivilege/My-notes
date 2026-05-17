---
title: "Managed Agents 的操作系統類比"
date: 2026-04-26
source: tech
---

- **Managed Agents 的操作系統類比**：`read()` 命令不關心底層是 70 年代磁盤還是現代 SSD。Managed Agents 做的是同樣的事——虛擬化session/harness/sandbox，接口穩定而實現可換。為不確定性設計，不為當前最優解設計。這是真正的工程智慧：知道什麼該耦合什麼該解耦，什麼是會變的什麼是穩定的。
