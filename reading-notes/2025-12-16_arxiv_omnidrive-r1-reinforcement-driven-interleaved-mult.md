# OmniDrive-R1: Reinforcement-driven Interleaved Multi-modal Chain-of-Thought for Trustworthy Vision-Language Autonomous Driving

- **Source**: https://arxiv.org/abs/2512.14044
- **Date**: 2025-12-16
- **Authors / Org**: Zhenguo Zhang, Haohan Zheng, Yishen Wang, Le Xu, Tianchen Deng, Xuefeng Chen, Qu Chen, Bo Zhang, Wuxiong Huang
- **Type**: arxiv

## Core Problem

Existing vision-language models for autonomous driving suffer from object hallucination due to ungrounded text-based chain-of-thought reasoning. Current multi-modal CoT approaches are flawed because they decouple perception and reasoning, preventing end-to-end joint optimization, and they rely on expensive dense localization labels.

## Core Approach

1. **Interleaved Multi-modal Chain-of-Thought (iMCoT)**: Unifies perception and reasoning into a single end-to-end framework where visual and textual reasoning steps are interleaved, allowing the model to dynamically attend to and 'zoom in' on critical image regions during the reasoning process.

2. **Reinforcement-driven visual grounding via Clip-GRPO**: A pure two-stage reinforcement learning pipeline that trains the model to autonomously direct its visual attention. Clip-GRPO introduces an annotation-free, process-based grounding reward that enforces real-time cross-modal consistency between the model's visual focus and its textual reasoning, avoiding the need for dense labels and external tool calls.

## Evidence

On the DriveLMM-o1 benchmark, OmniDrive-R1 improves the overall reasoning score from 51.77% to 80.35% and final answer accuracy from 37.81% to 73.62% compared to the baseline Qwen2.5VL-7B.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the Clip-GRPO reward's cross-modal consistency metric is a sufficient proxy for grounding quality, but does not prove that this reward alone eliminates all forms of hallucination or ensures safety-critical reliability.
- ⚠️ The evaluation is limited to the DriveLMM-o1 benchmark; it is unclear how the model generalizes to diverse driving scenarios, weather conditions, or rare edge cases not represented in the benchmark.
- ⚠️ The two-stage reinforcement learning pipeline may introduce training instability or reward hacking that is not fully analyzed; the paper does not provide ablation studies on the reward design or training dynamics.
- ⚠️ The model's 'zoom in' capability is claimed to be autonomous, but the paper does not quantify the precision or recall of the visual attention regions compared to ground-truth object locations, leaving the grounding accuracy unvalidated.
- ⚠️ The approach assumes that interleaved multi-modal CoT is inherently more trustworthy than decoupled methods, but does not compare against other grounding techniques (e.g., explicit object detectors or attention supervision) to isolate the benefit of the reinforcement-driven mechanism.
