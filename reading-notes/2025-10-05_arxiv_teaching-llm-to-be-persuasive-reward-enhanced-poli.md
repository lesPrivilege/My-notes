# Teaching LLM to be Persuasive: Reward-Enhanced Policy Optimization for Alignment from Heterogeneous Rewards

- **Source**: https://arxiv.org/abs/2510.04214
- **Date**: 2025-10-05
- **Authors / Org**: Xia Zeng, Yihan Chen, Luhui Liu, Chao Luo, Ye Chen, Zhuoran Zhuang
- **Type**: arxiv

## Core Problem

Existing alignment methods (e.g., GRPO) rely on a single reward model, which fails to capture the diverse, often conflicting requirements of real-world persuasive agents—such as strict SOP compliance, guardrails, human-likeness, and emotional nuance—leading to suboptimal performance in multi-turn negotiation dialogues.

## Core Approach

1. **Heterogeneous Reward Composition**: REPO combines three reward sources: a preference-trained reward model (RM) for overall dialogue quality, an LLM-as-a-judge (RJ) for nuanced behavioral aspects (emotional value, SOP compliance), and rule-based reward functions (RF) for deterministic checks (numerics, formatting, guardrails). These rewards are aggregated to guide policy optimization.

2. **Reward-Enhanced Policy Optimization (REPO)**: A reinforcement learning post-training method that extends GRPO by incorporating the heterogeneous reward signals into the policy gradient update, allowing the LLM to balance trade-offs between persuasive effectiveness, safety constraints, and human-like interaction.

## Evidence

In expert consensus evaluation (3 experts, 30 online conversations + 45 curated bad cases): REPO achieves average dialogue rating 4.63 (+0.33 over GRPO), share of conversations with at least one excellent response 66.67% (+23.34 pp over GRPO), bad-case fix rate 93.33% with 75.56% clean fixes. In production A/B test (9,653 real conversations vs. intent-driven system): REPO improves response rate by +12.14 pp and task success rate by +5.94 pp (p<0.001).

## Assumptions & Open Questions

- ⚠️ The paper assumes that the three reward components (RM, RJ, RF) are complementary and that their simple combination (likely weighted sum) is optimal; no ablation or sensitivity analysis on reward weights is provided.
- ⚠️ The LLM-as-a-judge (RJ) introduces a potential circular dependency: the judge model may share biases or limitations with the policy model, especially if both are from the same family (e.g., GPT-4). The paper does not discuss judge model selection or calibration.
- ⚠️ The rule-based reward functions (regex-based) may be brittle and fail to capture semantic nuances of guardrails (e.g., subtle over-promising), potentially leading to reward hacking or false positives/negatives.
- ⚠️ The expert evaluation uses only 30 online conversations and 45 curated bad cases—a small sample that may not generalize to the full distribution of real-world dialogues.
- ⚠️ The production A/B test compares against an 'intent-driven dialogue system' rather than a state-of-the-art LLM baseline (e.g., GRPO), making it unclear how much of the gain is due to REPO vs. the underlying LLM architecture.
- ⚠️ The paper does not report computational cost or training stability of REPO compared to GRPO, which is important for practical deployment.
