# Implicit bias produces neural scaling laws in learning curves, from perceptrons to deep networks

- **Source**: https://arxiv.org/abs/2505.13230
- **Date**: 2025-05-19
- **Authors / Org**: Francesco D&#39;Amico, Dario Bocchi, Matteo Negri
- **Type**: arxiv

## Core Problem

Existing scaling law studies focus on asymptotic performance at the end of training, neglecting the full training dynamics. This work addresses the gap by characterizing how performance scales with norm-based complexity measures throughout the entire training process.

## Core Approach

1. **Dynamical scaling law 1 (norm-based complexity growth)**: Identifies a power-law relationship between the norm of the weights (or other complexity measures) and training time, driven by the implicit bias of gradient descent.

2. **Dynamical scaling law 2 (performance vs. complexity)**: Establishes a power-law relationship between test error and the evolving norm-based complexity measure during training, showing that performance improves as complexity increases.

3. **Analytical derivation with perceptron**: Derives the two dynamical scaling laws exactly for a single-layer perceptron trained with logistic loss, using the implicit bias of gradient descent to explain the mechanisms.

4. **Empirical validation across architectures**: Demonstrates that the same dynamical scaling laws hold for CNNs, ResNets, and Vision Transformers on MNIST, CIFAR-10, and CIFAR-100, showing generality beyond simple models.

## Evidence

The paper shows that the two dynamical scaling laws recover the well-known asymptotic test error scaling at convergence. Empirical results are consistent across CNNs, ResNets, and Vision Transformers on MNIST, CIFAR-10, and CIFAR-100. No specific numerical values (e.g., exponents or error rates) are provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The analysis assumes that implicit bias from gradient descent is the sole driver of the observed scaling laws, but other factors (e.g., data distribution, initialization, learning rate schedule) may also contribute.
- ⚠️ The analytical derivation is limited to a single-layer perceptron with logistic loss; it is unclear whether the same mechanisms fully explain the laws in deep networks without additional assumptions.
- ⚠️ The paper does not specify the exact norm-based complexity measures used (e.g., L2 norm, path norm, or other), leaving ambiguity about which measure is most appropriate.
- ⚠️ The claim that the laws 'recover' asymptotic scaling at convergence is not quantitatively demonstrated in the abstract; the exact relationship between the dynamical laws and known asymptotic exponents is not stated.
- ⚠️ The experiments are limited to small-scale datasets (MNIST, CIFAR-10/100); it is an open question whether these dynamical laws hold for large-scale datasets like ImageNet or modern LLMs.
