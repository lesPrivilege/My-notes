# Turning the TIDE: Cross-Architecture Distillation for Diffusion Large Language Models

- **Source**: https://arxiv.org/abs/2604.26951
- **Date**: 2026-04-29
- **Authors / Org**: Gongbo Zhang, Wen Wang, Ye Tian, Li Yuan
- **Type**: arxiv

## Core Problem

Existing distillation methods for diffusion LLMs only work within a single architecture, failing to transfer knowledge when teacher and student differ in architecture, attention mechanism, or tokenizer. This limits the ability to compress large, state-of-the-art diffusion models into smaller, more efficient ones.

## Core Approach

1. **TIDAL (Timestep-Aware Distillation with Adaptive Learning)**: Jointly modulates distillation strength across both training progress and diffusion timestep, accounting for the teacher's noise-dependent reliability—the teacher's predictions are more reliable at low-noise timesteps and less so at high-noise timesteps.

2. **CompDemo (Complementary Mask Decomposition)**: Enriches the teacher's context by splitting the input mask into complementary parts, allowing the teacher to make better predictions under heavy masking conditions, which is critical for diffusion models that operate on partially masked inputs.

3. **Reverse CALM (Cross-tokenizer Alignment via Likelihood Matching)**: A cross-tokenizer objective that inverts chunk-level likelihood matching, providing bounded gradients and dual-end noise filtering to handle mismatched tokenizers between teacher and student.

## Evidence

Distilling 8B dense and 16B MoE teachers into a 0.6B student outperforms the baseline by an average of 1.53 points across eight benchmarks. In code generation, HumanEval scores reach 48.78 compared to 32.3 for the AR baseline.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the teacher's noise-dependent reliability can be effectively captured by a simple joint modulation of training progress and timestep, without proving that more complex or adaptive schedules wouldn't yield better results.
- ⚠️ CompDemo's complementary mask splitting strategy is not theoretically justified for all types of masking distributions; its effectiveness may depend on the specific masking schedule used during training.
- ⚠️ The cross-tokenizer objective (Reverse CALM) assumes that chunk-level likelihood matching with inversion is sufficient for alignment, but it does not address potential semantic drift or tokenization artifacts that could degrade performance on tasks requiring fine-grained token-level precision.
- ⚠️ The empirical results are limited to two teacher architectures (8B dense, 16B MoE) and one student size (0.6B); it is unclear how well the method scales to other teacher-student size ratios or different architectural families (e.g., encoder-only vs. decoder-only).
- ⚠️ The paper does not analyze the computational overhead of the three components (TIDAL, CompDemo, Reverse CALM) relative to standard distillation, leaving open questions about practical efficiency gains beyond benchmark scores.
