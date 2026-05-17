# TeD-Loc: Text Distillation for Weakly Supervised Object Localization

- **Source**: https://arxiv.org/abs/2501.12632
- **Date**: 2025-01-22
- **Authors / Org**: Shakeeb Murtaza, Soufiane Belharbi, Alexis Guichemerre, Marco Pedersoli, Eric Granger
- **Type**: arxiv

## Core Problem

Traditional WSOL methods (e.g., CAM) focus on the most discriminative object parts, missing full spatial extent. Vision-language models like CLIP are not directly suited for WSOL because global text/class-token embeddings lack explicit alignment with local patch embeddings, and existing adaptations (e.g., GenPrompt) introduce high complexity.

## Core Approach

1. **Contrastive Text-to-Patch Distillation**: Transfers knowledge from CLIP text embeddings to patch embeddings by aligning them contrastively, enabling patch-level foreground/background discrimination without requiring bounding box annotations.

2. **Localization-Guided Classification Module**: Uses localization scores to aggregate foreground patch embeddings for joint classification and localization within a single model, integrating both tasks end-to-end.

3. **QR-Based Orthogonalization of Class Text Embeddings**: Applies QR decomposition to orthogonalize class text embeddings before distillation, improving discrimination between semantically similar classes by reducing embedding overlap.

## Evidence

TeD-Loc improves Top-1 Loc by ~5% on CUB and ILSVRC, and PxAP by ~31% on histopathology benchmarks, while achieving more efficient inference than GenPrompt.

## Assumptions & Open Questions

- ⚠️ The method assumes that CLIP text embeddings contain sufficient semantic information for patch-level localization, which may not hold for highly abstract or visually ambiguous classes.
- ⚠️ The contrastive alignment relies on image-level labels only; it does not explicitly verify that distilled patch embeddings capture object boundaries rather than correlated background patterns.
- ⚠️ The QR orthogonalization step assumes that class text embeddings are linearly separable in the CLIP space, which may not be true for fine-grained categories with subtle differences.
- ⚠️ The paper does not analyze failure cases where localization fails (e.g., multiple objects, occlusions, or small objects), leaving robustness questions open.
- ⚠️ The efficiency claim over GenPrompt is not quantified in terms of FLOPs or wall-clock time, only mentioned qualitatively.
