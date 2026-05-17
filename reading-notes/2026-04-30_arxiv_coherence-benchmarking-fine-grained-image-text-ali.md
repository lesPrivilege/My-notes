# COHERENCE: Benchmarking Fine-Grained Image-Text Alignment in Interleaved Multimodal Contexts

- **Source**: https://arxiv.org/abs/2604.27389
- **Date**: 2026-04-30
- **Authors / Org**: Bingli Wang, Huanze Tang, Haijun Lv, Zhishan Lin, Lixin Gu, Lei Feng, Qipeng Guo, Kai Chen
- **Type**: arxiv

## Core Problem

Existing multimodal benchmarks focus on single-image or multi-image comprehension, but real-world scenarios like document reading involve interleaved image-text contexts where MLLMs must identify relevant evidence and establish fine-grained alignments. There is no systematic benchmark to quantify this ability.

## Core Approach

1. **Interleaved Context Design**: The benchmark constructs evaluation samples from four representative domains (e.g., documents, web pages) where images and text are interleaved, mimicking real-world multimodal layouts.

2. **Fine-Grained Alignment Questions**: COHERENCE includes 6,161 high-quality questions that require MLLMs to recover specific correspondences between visual elements and textual references, testing beyond surface-level recognition.

3. **Six-Type Error Analysis**: The benchmark provides a taxonomy of six error types to attribute failures in interleaved understanding to specific missing capabilities, enabling fine-grained diagnosis of model weaknesses.

## Evidence

The benchmark covers four domains and contains 6,161 high-quality questions. The paper does not report specific model performance numbers or comparisons in the abstract, but the error analysis framework is a key empirical contribution.

## Assumptions & Open Questions

- ⚠️ The benchmark assumes that interleaved contexts from four domains are representative of all real-world scenarios, but domain coverage may be limited.
- ⚠️ The six error types are defined by the authors; it is not proven that they are exhaustive or that they capture all failure modes in interleaved understanding.
- ⚠️ The quality of the 6,161 questions is asserted but not independently validated; human annotation agreement or bias is not discussed.
- ⚠️ The benchmark does not address whether current MLLMs can leverage the error analysis to improve, or if the errors are irreducible due to model architecture limitations.
