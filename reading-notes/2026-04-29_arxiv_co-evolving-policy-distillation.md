# Co-Evolving Policy Distillation

- **Source**: https://arxiv.org/abs/2604.27083
- **Date**: 2026-04-29
- **Authors / Org**: Naibin Gu, Chenxu Yang, Qingyi Si, Chuanyu Qin, Dingyu Yao, Peng Fu, Zheng Lin, Weiping Wang, Nan Duan, Jiaqi Wang
- **Type**: arxiv

## Core Problem

Existing paradigms for consolidating multiple expert capabilities into a single model suffer from capability loss: mixed RLVR incurs inter-capability divergence cost, while the pipeline of training experts first and then performing offline policy distillation (OPD) fails to fully absorb teacher capabilities due to large behavioral pattern gaps between teacher and student.

## Core Approach

1. 1. Parallel expert training with interleaved OPD: Experts are trained simultaneously using RLVR, and OPD is introduced during each expert's ongoing RLVR training rather than after complete expert training, enabling more consistent behavioral patterns among experts.

2. 2. Bidirectional mutual teaching: Experts serve as mutual teachers, making OPD bidirectional, which allows them to co-evolve and maintain sufficient complementary knowledge throughout training.

## Evidence

Experiments validate that CoPD achieves all-in-one integration of text, image, and video reasoning capabilities, significantly outperforming strong baselines such as mixed RLVR and MOPD, and even surpassing domain-specific experts.

## Assumptions & Open Questions

- ⚠️ The paper assumes that parallel training of experts with interleaved OPD does not introduce prohibitive computational overhead, but no quantitative analysis of training cost or scalability is provided.
- ⚠️ The claim that CoPD 'may inspire a novel training scaling paradigm' is speculative and not supported by empirical scaling laws or ablation studies on model size or data volume.
- ⚠️ The method assumes that experts can effectively serve as mutual teachers without negative interference, but the potential for harmful cross-contamination (e.g., one expert's weaknesses propagating to others) is not examined.
- ⚠️ The evaluation focuses on reasoning capabilities in text, image, and video domains, but it is unclear how CoPD generalizes to other modalities or tasks (e.g., speech, code generation).
- ⚠️ The paper does not discuss the sensitivity of CoPD to hyperparameters such as the frequency of OPD updates, the number of experts, or the relative weighting of RLVR and distillation losses.
