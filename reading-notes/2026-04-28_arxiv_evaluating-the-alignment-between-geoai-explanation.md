# Evaluating the Alignment Between GeoAI Explanations and Domain Knowledge in Satellite-Based Flood Mapping

- **Source**: https://arxiv.org/abs/2604.26051
- **Date**: 2026-04-28
- **Authors / Org**: Hyunho Lee, Wenwen Li
- **Type**: arxiv

## Core Problem

Deep learning models for satellite-based flood mapping achieve high performance but lack transparency, hindering their adoption in critical scientific and operational workflows. There is no systematic method to assess whether model explanations align with domain knowledge about spectral properties of Earth's surface.

## Core Approach

1. **ADAGE Framework**: A systematic evaluation framework designed to quantify the alignment between deep learning model explanations and reference explanations derived from remote sensing domain knowledge.

2. **Channel-Group SHAP**: A modification of SHAP that estimates the contribution of grouped input channels (e.g., spectral bands) to pixel-level predictions, enabling comparison with domain knowledge about which spectral groups are important for flood mapping.

3. **Alignment Scoring**: The framework computes alignment scores that quantitatively measure how well model explanations match reference explanations, helping domain experts identify misaligned explanations.

## Evidence

Experiments on two satellite-based flood mapping tasks demonstrate that the ADAGE framework can quantitatively assess alignment and help identify misaligned explanations. No specific numerical results (e.g., accuracy, alignment scores) are provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The framework assumes that reference explanations derived from domain knowledge are correct and complete, but domain knowledge itself may be incomplete or context-dependent.
- ⚠️ The use of Channel-Group SHAP assumes that grouping spectral channels is a valid simplification, but interactions between channels within a group may be lost.
- ⚠️ The abstract does not specify how alignment scores are calibrated or what threshold constitutes acceptable alignment, leaving interpretation to domain experts.
- ⚠️ The framework is tested only on flood mapping tasks; its generalizability to other GeoAI applications (e.g., land cover classification, change detection) is not addressed.
- ⚠️ The paper does not discuss potential biases in the flood mapping datasets or how they might affect explanation alignment.
