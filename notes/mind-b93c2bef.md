---
title: "Prompt 攻擊的進化：從算力到判斷"
date: 2026-05-16
source: mind
---

- **Prompt 攻擊的進化：從算力到判斷**：早期 prompt 攻擊的目標函數是燒 token、讓模型證哥德巴赫猜想直到 context 耗盡——這是對算力的攻擊。當下的 prompt 攻擊（如 Anna's Archive 的 llms.txt）目標函數是劫持判斷、讓模型相信應該做某件事——這是對判斷的攻擊。前者靠 harness 和 rate limit 能擋；後者只能靠模型本身在訓練中習得對修辭性誘導的免疫，因為它不是製造無限循環，而是在 LLM 本來就在做的事情裡植入偏向。這條進化線說明：外部補丁終究是補丁，真問題還是在 representation 幾何裡。
