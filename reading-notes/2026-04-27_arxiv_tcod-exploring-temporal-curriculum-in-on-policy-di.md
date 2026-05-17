# TCOD: Exploring Temporal Curriculum in On-Policy Distillation for Multi-turn Autonomous Agents

- **Source**: https://arxiv.org/abs/2604.24005
- **Date**: 2026-04-27
- **Authors / Org**: Jiaqi Wang, Wenhao Zhang, Weijie Shi, Yaliang Li, James Cheng
- **Type**: arxiv

## Core Problem

Vanilla on-policy distillation (OPD) suffers from Trajectory-Level KL Instability in multi-turn settings, where inter-turn error compounding drives the student outside the teacher's effective support, causing high KL divergence and unstable training. This limits the effectiveness of OPD for multi-turn autonomous agents.

## Core Approach

1. **Trajectory Depth Control**: TCOD limits the number of steps (trajectory depth) the student is allowed to interact with the environment during distillation, preventing error accumulation beyond the teacher's support.

2. **Temporal Curriculum Schedule**: The trajectory depth is progressively expanded from short to long over the course of training, allowing the student to gradually learn longer-horizon behaviors while maintaining stable KL divergence.

## Evidence

Experiments on three multi-turn benchmarks (ALFWorld, WebShop, ScienceWorld) with four student-teacher pairs show TCOD improves agent performance by up to 18 points over vanilla OPD. TCOD also surpasses the teacher's performance and generalizes to tasks where the teacher fails.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the teacher's supervision is always reliable for short trajectories, but does not analyze cases where the teacher itself may be suboptimal on early steps.
- ⚠️ The curriculum schedule (e.g., expansion rate, starting depth) is not theoretically justified; it may require tuning per task or teacher-student pair.
- ⚠️ The method is evaluated only on simulated environments; its effectiveness in real-world or more complex multi-turn settings (e.g., with partial observability or long horizons) is unknown.
- ⚠️ The paper does not compare against other curriculum or stabilization techniques (e.g., KL penalty, trust region methods) to isolate the benefit of the temporal curriculum.
- ⚠️ The claim of surpassing the teacher's performance is not deeply analyzed—it is unclear whether this is due to regularization, exploration, or other factors.
