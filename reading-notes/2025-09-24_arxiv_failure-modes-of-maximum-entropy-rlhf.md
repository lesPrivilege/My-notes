# Failure Modes of Maximum Entropy RLHF

- **Source**: https://arxiv.org/abs/2509.20265
- **Date**: 2025-09-24
- **Authors / Org**: Ömer Veysel Çağatan, Barış Akgün
- **Type**: arxiv

## Core Problem

The paper addresses the gap between the theoretical connection of SimPO to Maximum Entropy RL and its empirical success in offline preference optimization, versus the failure of Maximum Entropy RL in online RLHF settings. It investigates why reference-free methods like SimPO work offline but struggle online, particularly with overoptimization and reward hacking.

## Core Approach

1. **Derivation of SimPO as Maximum Entropy RL**: The authors formally derive SimPO from the Maximum Entropy Reinforcement Learning framework, establishing a theoretical foundation for this reference-free preference optimization method.

2. **Online RLHF experiments with Maximum Entropy RL**: They apply Maximum Entropy RL directly to online RLHF settings, testing across multiple model scales and learning rates to evaluate its performance and stability.

3. **Comparison with KL-constrained methods**: The authors compare Maximum Entropy RL against standard KL-constrained RLHF methods to highlight differences in training stability, overoptimization, and KL dynamics.

4. **Analysis of overoptimization and entropy regularization**: They analyze the correlation between entropy regularization and overoptimization, finding that entropy regularization does not prevent reward hacking and may even coincide with its onset.

## Evidence

Experiments show that Maximum Entropy RL frequently exhibits overoptimization and unstable KL dynamics across model scales, with overoptimization persisting even at conservative learning rates for some configurations. Unlike KL-constrained methods, entropy regularization fails to reliably prevent reward hacking. The paper does not provide specific numerical benchmarks (e.g., reward scores or KL divergence values) in the abstract.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the derivation of SimPO as Maximum Entropy RL is exact and complete, but does not prove that all practical implementations of SimPO strictly adhere to this framework.
- ⚠️ The experiments are limited to online RLHF; the authors do not test whether alternative reference-free methods (beyond SimPO) also fail online, leaving open the question of generalizability.
- ⚠️ The claim that entropy regularization correlates with overoptimization is observational; causal mechanisms are not established, and confounding factors (e.g., reward model quality) are not controlled.
- ⚠️ The paper does not quantify the extent of overoptimization (e.g., reward vs. true objective gap) or provide statistical significance for the observed instability across runs.
- ⚠️ The discussion of why SimPO succeeds offline while Maximum Entropy RL fails online is speculative; no formal proof or ablation study is provided to isolate the key difference (e.g., offline vs. online data distribution).
