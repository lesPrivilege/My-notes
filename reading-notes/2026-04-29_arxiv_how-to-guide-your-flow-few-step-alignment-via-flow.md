# How to Guide Your Flow: Few-Step Alignment via Flow Map Reward Guidance

- **Source**: https://arxiv.org/abs/2604.27147
- **Date**: 2026-04-29
- **Authors / Org**: Jerry Y. Huang, Justin Lin, Sheel Shah, Kartik Nair, Nicholas M. Boffi
- **Type**: arxiv

## Core Problem

Existing guidance methods for generative models either require expensive multi-particle, many-step sampling or rely on poorly understood approximations, limiting their efficiency and applicability for fast, high-quality sample generation.

## Core Approach

1. **Optimal Control Reformulation**: The authors reformulate guidance as a deterministic optimal control problem, deriving a hierarchy of algorithms that subsumes existing approaches at the coarsest level and provides a principled foundation for guidance.

2. **Flow Map Utilization**: The flow map, previously studied for fast inference, is shown to arise naturally in the optimal solution and is used to both integrate and guide the generative flow, enabling single-trajectory guidance without training.

3. **Flow Map Reward Guidance (FMRG)**: FMRG is a training-free framework that leverages the flow map to compute gradients of the reward function along the trajectory, allowing efficient, few-step guidance without requiring multiple particles or many sampling steps.

## Evidence

At text-to-image scale, FMRG matches or surpasses baselines across inverse problems, style transfer, human preferences, and VLM rewards with as few as 3 neural function evaluations (NFEs), achieving at least an order-of-magnitude speedup compared to prior state-of-the-art methods.

## Assumptions & Open Questions

- ⚠️ The method assumes that the flow map can be accurately approximated or computed for the given generative model, which may not hold for all architectures or training regimes.
- ⚠️ The optimal control reformulation relies on a deterministic setting; the extension to stochastic flows or non-differentiable rewards is not addressed.
- ⚠️ The paper does not provide a theoretical guarantee that the single-trajectory guidance converges to the global optimum of the reward, especially for complex, non-convex reward landscapes.
- ⚠️ The empirical evaluation is limited to text-to-image tasks; it is unclear how well FMRG generalizes to other modalities (e.g., audio, video, or 3D) or to very high-dimensional data.
- ⚠️ The comparison to baselines may be sensitive to hyperparameter tuning (e.g., guidance scale), and the paper does not fully explore the robustness of FMRG to such choices.
