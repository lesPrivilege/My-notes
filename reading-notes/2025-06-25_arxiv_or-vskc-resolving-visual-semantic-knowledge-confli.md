# OR-VSKC: Resolving Visual-Semantic Knowledge Conflicts in Operating Rooms with Synthetic Data-Guided Alignment

- **Source**: https://arxiv.org/abs/2506.22500
- **Date**: 2025-06-25
- **Authors / Org**: Weiyi Zhao, Xiaoyu Tan, Liang Liu, Sijia Li, Youwei Song, Xihe Qiu
- **Type**: arxiv

## Core Problem

Multimodal Large Language Models (MLLMs) often fail to activate safety knowledge during visual inspection in operating rooms due to Visual-Semantic Knowledge Conflicts (VS-KC), and research on this alignment gap is hindered by the scarcity and privacy restrictions of real-world OR data depicting safety violations.

## Core Approach

1. **Protocol-to-Pixel Generative Framework**: A method to generate high-fidelity synthetic images grounded in authoritative safety standards, producing 28,190 images that simulate OR safety violations without relying on real sensitive data.

2. **Expert-Authored Challenge Subset**: A 713-image subset validated by multiple experts to ensure realism and difficulty, serving as a rigorous test for MLLMs beyond the main synthetic benchmark.

3. **Dual-Dataset Benchmark Construction**: The benchmark is built from authentic OR contexts from the 4D-OR and CAMMA-MVOR datasets, with 4D-OR as the primary core and CAMMA-MVOR reserved for external validation and cross-dataset generalization analysis.

## Evidence

Evaluations of state-of-the-art MLLMs reveal substantial reliability gaps even in advanced generalist models. Fine-tuning on OR-VSKC effectively mitigates VS-KC and enables robust generalization to unseen camera viewpoints. The benchmark comprises 28,190 synthetic images and a 713-image expert-authored challenge subset.

## Assumptions & Open Questions

- ⚠️ The synthetic images, while high-fidelity, may not fully capture the complexity and variability of real OR environments, potentially limiting ecological validity.
- ⚠️ The assumption that fine-tuning on synthetic data generalizes to real-world safety violations is supported only by cross-dataset generalization to CAMMA-MVOR, which itself may have limited diversity.
- ⚠️ The paper does not quantify the degree of VS-KC reduction in absolute terms (e.g., accuracy gains) or compare against other mitigation strategies beyond fine-tuning.
- ⚠️ The reliance on authoritative safety standards for image generation may introduce bias toward known violations, missing novel or rare safety risks.
- ⚠️ The expert validation of the challenge subset, while rigorous, may still reflect subjective interpretations of safety violations, and inter-expert agreement is not reported.
