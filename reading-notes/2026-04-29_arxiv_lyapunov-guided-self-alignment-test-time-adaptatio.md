# Lyapunov-Guided Self-Alignment: Test-Time Adaptation for Offline Safe Reinforcement Learning

- **Source**: https://arxiv.org/abs/2604.26516
- **Date**: 2026-04-29
- **Authors / Org**: Seungyub Han, Hyungjin Kim, Jungwoo Lee
- **Type**: arxiv

## Core Problem

Offline RL agents often exhibit unsafe behavior when deployed due to distribution shift between training datasets and real environments. Existing safe RL methods typically require retraining or online interaction, which is impractical in many safety-critical applications.

## Core Approach

1. **Self-Alignment via Lyapunov-Guided Imagination**: At test time, the pretrained agent generates multiple imagined trajectories and selects those that satisfy a Lyapunov condition, ensuring safety constraints are met in the imagined rollouts.

2. **In-Context Prompt Recycling**: The selected feasible trajectory segments are used as in-context prompts for the transformer-based agent, allowing it to realign its behavior toward safety without updating model parameters.

3. **Hierarchical RL Interpretation via Bayesian Inference**: The prompting mechanism is interpreted as Bayesian inference over latent skills, where the transformer architecture enables hierarchical decision-making by conditioning on safe prompts.

## Evidence

Across Safety Gymnasium and MuJoCo benchmarks, SAS consistently reduces cost and failure while maintaining or improving return. Specific numerical results are not provided in the abstract, but the claims indicate consistent improvement over baselines.

## Assumptions & Open Questions

- ⚠️ The Lyapunov condition is assumed to be a reliable proxy for safety in imagined trajectories, but its validity under distribution shift is not proven.
- ⚠️ The method assumes that the pretrained agent can generate sufficiently diverse and safe imagined trajectories at test time, which may fail in highly stochastic or unseen environments.
- ⚠️ The abstract does not quantify the computational overhead of generating and filtering multiple imagined trajectories at test time, which could limit real-time applicability.
- ⚠️ The hierarchical RL interpretation is presented as a conceptual framing, but no empirical evidence is given to validate that prompting truly functions as Bayesian inference over latent skills.
- ⚠️ The approach relies on a transformer architecture, but the abstract does not discuss how the model scales with trajectory length or the number of prompts, leaving scalability questions open.
