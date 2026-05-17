# Federated Knowledge Distillation for Multi-Model Architectures Lithography Hotspot Detection

- **Source**: https://arxiv.org/abs/2501.04066
- **Date**: 2025-01-07
- **Authors / Org**: Yuqi Li, Xingyou Lin, Yanli Li, Kai Zhang, Chuanguang Yang, Zhongliang Guo, Jianping Gou, Tingwen Huang, Yingli Tian
- **Type**: arxiv

## Core Problem

Existing federated learning approaches for lithography hotspot detection rely on either parameter aggregation or knowledge distillation alone, failing to fully exploit collaborative learning. This limits effectiveness and robustness in detecting hotspots across distributed datasets with privacy constraints.

## Core Approach

1. **Hybrid Information Exchange**: Clients exchange both parameters of agreed-upon layers and logits (output probabilities) using a public dataset as a consensus reference, enabling richer knowledge transfer than either method alone.

2. **Public Dataset for Consensus**: A shared public dataset is used to facilitate alignment between client models, allowing logit-based distillation and parameter aggregation to work synergistically without exposing private data.

3. **Aggregation and Refinement**: The hybrid information (parameters and logits) is aggregated on a central server, and the result is used to refine local models, enhancing collaborative learning across heterogeneous client architectures.

## Evidence

Experiments on ICCAD-2012 and real-world FAB datasets show FedKD-hybrid consistently outperforms state-of-the-art methods in both effectiveness and robustness. Specific numerical results (e.g., accuracy, F1-score, or false alarm rates) are not provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The abstract assumes a public dataset is available and representative of the private data distribution, which may not hold in practice and could introduce bias.
- ⚠️ The method assumes clients agree on which layers to share parameters, but this coordination overhead and its impact on privacy are not discussed.
- ⚠️ No quantitative evidence (e.g., exact detection accuracy, false positive rates, or convergence speed) is provided to support the claimed outperformance.
- ⚠️ The robustness claim is not defined or measured; it is unclear whether it refers to adversarial robustness, data heterogeneity, or communication failures.
- ⚠️ The framework's scalability to a large number of clients or heterogeneous model architectures is not addressed.
