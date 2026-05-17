# PRISM: Pre-alignment via Black-box On-policy Distillation for Multimodal Reinforcement Learning

- **Source**: https://arxiv.org/abs/2604.28123
- **Date**: 2026-04-30
- **Authors / Org**: Sudong Wang, Weiquan Huang, Xiaomin Yu, Zuhao Yang, Hehai Lin, Keming Wu, Chaojun Xiao, Chen Chen, Wenxuan Wang, Beier Zhu, Yunjian Zhang, Chengwei Qin
- **Type**: arxiv

## Core Problem

Standard post-training for large multimodal models (LMMs) uses supervised fine-tuning (SFT) followed by reinforcement learning with verifiable rewards (RLVR), but SFT introduces distributional drift that degrades original capabilities and mismatches the supervision distribution. This drift is amplified in multimodal reasoning due to compounding perception and reasoning errors during subsequent RL.

## Core Approach

1. 1. Three-stage pipeline: Inserts an explicit distribution-alignment stage between SFT and RLVR to mitigate drift, building on the principle of on-policy distillation (OPD).

2. 2. Black-box adversarial game: Casts alignment as a response-level adversarial game between the policy and a Mixture-of-Experts (MoE) discriminator, without requiring access to teacher logits.

3. 3. MoE discriminator with dedicated experts: Uses separate perception and reasoning experts to provide disentangled corrective signals that steer the policy toward the supervision distribution.

4. 4. Curated high-fidelity demonstrations: Supplements 1.26M public SFT demonstrations with 113K additional demonstrations from Gemini 3 Flash, featuring dense visual grounding and step-by-step reasoning on the hardest unsolved problems.

## Evidence

Experiments on Qwen3-VL show PRISM consistently improves downstream RLVR performance across multiple RL algorithms (GRPO, DAPO, GSPO) and diverse multimodal benchmarks, improving average accuracy by +4.4 points (4B model) and +6.0 points (8B model) over the SFT-to-RLVR baseline.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the 113K curated demonstrations from Gemini 3 Flash are of sufficiently high fidelity to correct drift, but does not quantify the quality or diversity of these demonstrations relative to the original 1.26M set.
- ⚠️ The MoE discriminator's disentangled perception and reasoning experts are claimed to provide corrective signals, but there is no ablation showing that the MoE structure outperforms a single discriminator or simpler alternatives.
- ⚠️ The method assumes that on-policy distillation (OPD) is the optimal way to align distributions, but does not compare against other alignment techniques (e.g., direct preference optimization or distribution matching) in the multimodal setting.
- ⚠️ The paper does not analyze the computational cost or training overhead of the additional alignment stage, which may be significant for larger models or real-world deployment.
- ⚠️ The experiments are limited to Qwen3-VL; it is unclear whether PRISM generalizes to other LMM architectures or domains beyond the benchmarks tested.
