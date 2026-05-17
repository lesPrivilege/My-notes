# TinyR1-32B-Preview: Boosting Accuracy with Branch-Merge Distillation

- **Source**: https://arxiv.org/abs/2503.04872
- **Date**: 2025-03-06
- **Authors / Org**: Lin Sun, Guangxiang Zhao, Xiaoqi Jian, Yuhan Wu, Weihong Lin, Yongfu Zhu, Qilong Shi, Change Jia, Aomufei Yuan, Yuxuan Tian, Linglin Zhang, Jinzhu Wu, Junfeng Ran, Sai-er Hu, Zihan Jiang, Junting Zhou, Wenrui Liu, Xusen Xiao, Bin Cui, Tong Yang, Xiangzheng Zhang
- **Type**: arxiv

## Core Problem

Existing model distillation and transfer learning methods often fail to maintain high accuracy when reducing LLM size, leaving a gap in achieving both compression and performance retention.

## Core Approach

1. **Branch Phase**: Knowledge from a large teacher model is selectively distilled into multiple specialized student models via domain-specific supervised fine-tuning (SFT), focusing on distinct areas like math, coding, or science.

2. **Merge Phase**: The specialized student models are merged into a single model to enable cross-domain knowledge transfer and improve generalization, combining their strengths.

## Evidence

The resulting TinyR1-32B-Preview outperforms DeepSeek-R1-Distill-Qwen-32B on Mathematics (+5.5 points), Coding (+4.4 points), and Science (+2.9 points), and achieves near-equal performance to DeepSeek-R1 on AIME 2024.

## Assumptions & Open Questions

- ⚠️ The paper assumes that domain-specific SFT and subsequent merging do not introduce conflicts or catastrophic forgetting, but provides no analysis of potential interference between specialized knowledge.
- ⚠️ The method is validated only with DeepSeek-R1 as teacher and Qwen-32B as student; it is unclear if the approach generalizes to other teacher-student pairs or architectures.
- ⚠️ The 'selective distillation' mechanism is not detailed—how domains are chosen, how specialization is enforced, and whether the branching is automated or manually designed remains unspecified.
- ⚠️ The computational cost and time savings are claimed but not quantified; no comparison of training resources versus standard distillation is provided.
- ⚠️ The benchmarks used are limited to math, coding, and science; performance on language understanding, reasoning, or safety tasks is not reported, leaving open questions about broader applicability.
