# Distributional Alignment Games for Answer-Level Fine-Tuning

- **Source**: https://arxiv.org/abs/2604.27166
- **Date**: 2026-04-29
- **Authors / Org**: Mehryar Mohri, Jon Schneider, Yifan Wu
- **Type**: arxiv

## Core Problem

Directly optimizing answer-level objectives in language models requires marginalizing over a vast space of latent reasoning paths, which is computationally intractable. Existing fine-tuning methods often rely on token-level or trace-level supervision, failing to efficiently leverage answer-level feedback.

## Core Approach

1. **Game-Theoretic Lifting to Distributional Alignment Game**: The ALFT problem is reformulated as a two-player game between a Policy (generator) and a Target (auxiliary distribution), where the Nash Equilibrium corresponds to the solution of the original answer-level optimization, turning intractable marginalization into a tractable projection problem.

2. **Variational Projection Perspective**: By treating the problem as a distributional alignment game, the intractable marginalization over reasoning paths is replaced with a projection onto a feasible set of distributions, enabling efficient optimization.

3. **Integration with GRPO (Coherence-GRPO)**: The framework is made compatible with Group Relative Policy Optimization (GRPO) by introducing a coherence-based variant, Coherence-GRPO, which leverages the game structure to achieve significant complexity gains in mathematical reasoning tasks.

## Evidence

The paper claims that Coherence-GRPO yields 'significant complexity gains in mathematical reasoning tasks,' but no specific quantitative numbers (e.g., accuracy, convergence speed, or computational cost) are provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The abstract assumes that the Nash Equilibrium of the distributional alignment game is both unique and reachable via the proposed algorithms, but does not provide convergence guarantees or discuss potential multiple equilibria.
- ⚠️ The claim that the framework 'unifies recent approaches to diversity and self-improvement (coherence)' is asserted without empirical comparison to existing methods or demonstration of unification beyond conceptual similarity.
- ⚠️ The computational tractability of the projection step is assumed but not analyzed; the complexity of projecting onto the feasible distribution set in practice is not quantified.
- ⚠️ The abstract does not specify the scale of language models or tasks tested, leaving open questions about scalability and generalizability beyond mathematical reasoning.
