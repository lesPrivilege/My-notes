# ViTaPEs: Visuotactile Position Encodings for Cross-Modal Alignment in Multimodal Transformers

- **Source**: https://arxiv.org/abs/2505.20032
- **Date**: 2025-05-26
- **Authors / Org**: Fotios Lygerakis, Ozan Özdenizci, Elmar Rückert
- **Type**: arxiv

## Core Problem

Existing visuotactile representation learning methods lack explicit study of positional encodings, which limits their ability to capture fine-grained spatial correlations between modalities. Additionally, they often rely on pre-trained vision-language models and struggle with task-agnostic generalization across diverse environments.

## Core Approach

1. **Two-stage positional injection**: Local (modality-specific) positional encodings are added within each vision and tactile stream separately, and a global positional encoding is added on the joint token sequence immediately before self-attention, providing a shared positional vocabulary for cross-modal interaction.

2. **Explicit positional injection points**: The method makes the injection points explicit and conducts controlled ablations to isolate the effect of adding positional encodings before a token-wise nonlinearity versus immediately before self-attention, enabling systematic analysis of their impact.

3. **Transformer-based architecture for task-agnostic learning**: The architecture is designed to learn representations from paired vision and tactile inputs without task-specific supervision, supporting zero-shot generalization and transfer to downstream tasks like robotic grasping.

## Evidence

ViTaPEs surpasses state-of-the-art baselines across various recognition tasks on multiple large-scale real-world datasets. It demonstrates zero-shot generalization to unseen, out-of-domain scenarios. In a robotic grasping task, it outperforms state-of-the-art baselines in predicting grasp success. Specific numerical results are not provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The abstract assumes that two-stage positional injection is universally beneficial, but does not discuss scenarios where local or global encodings might interfere or be redundant.
- ⚠️ The method relies on paired vision and tactile data for training; the cost and difficulty of obtaining such paired data at scale is not addressed.
- ⚠️ The zero-shot generalization claim is not quantified (e.g., accuracy drop compared to in-domain performance), leaving open how robust the representations are to domain shifts.
- ⚠️ The ablation study compares injection before nonlinearity vs. before self-attention, but does not explore other possible injection points (e.g., after attention) or combinations, limiting the understanding of design space.
- ⚠️ The robotic grasping task is mentioned as a transfer-learning demonstration, but no details on the dataset, task complexity, or comparison baselines are provided, making it hard to assess the significance of the improvement.
