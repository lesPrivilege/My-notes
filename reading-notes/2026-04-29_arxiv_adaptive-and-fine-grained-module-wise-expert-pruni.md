# Adaptive and Fine-grained Module-wise Expert Pruning for Efficient LoRA-MoE Fine-Tuning

- **Source**: https://arxiv.org/abs/2604.26340
- **Date**: 2026-04-29
- **Authors / Org**: Weihang Li, Jianchun Liu, Hongli Xu
- **Type**: arxiv

## Core Problem

Existing LoRA-MoE frameworks use a fixed, uniform expert configuration across all Transformer modules, ignoring their distinct functional roles, leading to redundant parameters and optimizer-state overhead. Additionally, enforcing load balancing throughout training restricts expert specialization once routing patterns stabilize.

## Core Approach

1. 1. Dynamic Module-wise Expert Pruning: Tracks expert utilization during training and physically removes low-utility experts on a per-module basis, creating a compact expert structure tailored to each module's needs.

2. 2. Post-Pruning Load-Balancing Removal: After pruning, the load-balancing constraint is eliminated, allowing remaining experts to specialize fully on the downstream task without forced balancing.

## Evidence

Experiments on multiple reasoning benchmarks show DMEP reduces trainable parameters by 35%–43% and improves training throughput by about 10%, while maintaining or surpassing the downstream reasoning accuracy of uniform LoRA-MoE baselines.

## Assumptions & Open Questions

- ⚠️ The paper assumes that expert utilization during training is a reliable proxy for future utility, but does not prove that low-utility experts cannot become useful later under different routing dynamics.
- ⚠️ The removal of load-balancing after pruning assumes that routing patterns have stabilized, but the criteria for 'stabilization' are not explicitly defined or validated.
- ⚠️ The method is evaluated only on reasoning benchmarks; its effectiveness on other tasks (e.g., generation, classification) or with different base models is not explored.
- ⚠️ The paper does not analyze the computational overhead of tracking expert utilization and performing pruning decisions, which could offset throughput gains in some settings.
