# RIHA: Report-Image Hierarchical Alignment for Radiology Report Generation

- **Source**: https://arxiv.org/abs/2604.27559
- **Date**: 2026-04-30
- **Authors / Org**: Yucheng Chen, Yang Yu, Yufei Shi, Conghao Xiong, Xulei Yang, Si Yong Yeo
- **Type**: arxiv

## Core Problem

Existing radiology report generation methods treat reports as flat sequences, ignoring their structured sections and semantic hierarchies, which hinders precise cross-modal alignment and reduces generation accuracy.

## Core Approach

1. **Visual Feature Pyramid (VFP)**: Extracts multi-scale visual features from radiological images to capture details at different resolutions, enabling alignment with hierarchical text structures.

2. **Text Feature Pyramid (TFP)**: Represents multi-granularity textual structures (paragraph, sentence, word levels) from reports, mirroring the hierarchical organization of clinical narratives.

3. **Cross-modal Hierarchical Alignment (CHA) module**: Leverages optimal transport to align visual and textual features across paragraph, sentence, and word levels, ensuring precise cross-modal mapping.

4. **Relative Positional Encoding (RPE) in decoder**: Models spatial and semantic relationships among tokens to enhance token-level alignment between visual features and generated text.

## Evidence

Experiments on IU-Xray and MIMIC-CXR datasets show RIHA outperforms existing state-of-the-art models in both natural language generation metrics (e.g., BLEU, ROUGE, METEOR) and clinical efficacy metrics (e.g., CheXpert-based scores). Specific numbers are not provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The abstract assumes that hierarchical alignment (paragraph/sentence/word) is the primary bottleneck; it does not compare against methods that use other forms of structured representation (e.g., graph-based or template-based).
- ⚠️ Optimal transport is used for alignment, but the computational cost and scalability to larger datasets or higher-resolution images are not discussed.
- ⚠️ The method is evaluated only on chest X-ray datasets; generalizability to other imaging modalities (e.g., CT, MRI) or different report styles is unaddressed.
- ⚠️ The abstract claims 'outperforms existing state-of-the-art models' but does not specify which models or provide effect sizes, making it hard to assess practical significance.
- ⚠️ The reliance on pre-defined hierarchical levels (paragraph, sentence, word) may not capture all clinically relevant structures (e.g., findings vs. impression sections have different importance).
