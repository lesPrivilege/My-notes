# Test-Time Safety Alignment

- **Source**: https://arxiv.org/abs/2604.26167
- **Date**: 2026-04-28
- **Authors / Org**: Baturay Saglam, Dionysis Kalogerias
- **Type**: arxiv

## Core Problem

Existing work on steering model behavior via input embeddings has only been applied to pretrained text-completion models for reducing surface-level profanity, not to aligned models that exhibit a bimodal refuse-or-comply output distribution. The gap is how to effectively control aligned models to minimize semantic harmfulness without retraining or access to model internals.

## Core Approach

1. **Sub-lexical embedding optimization**: The method optimizes input word embeddings at a sub-lexical level (e.g., token or character granularity) rather than full words, allowing fine-grained control over the model's output distribution.

2. **Zeroth-order gradient estimation**: Gradients of a black-box text-moderation API with respect to the input embeddings are estimated using zeroth-order methods (e.g., finite differences), enabling optimization without access to the model's internal gradients.

3. **Gradient descent on embeddings**: The estimated gradients are used to perform gradient descent directly on the input embeddings to minimize the semantic harmfulness score of the generated text, iteratively adjusting the prompt.

## Evidence

The method neutralizes every safety-flagged response on standard safety benchmarks, achieving 100% success rate in removing safety flags. No specific numerical benchmarks (e.g., exact harmfulness scores or dataset sizes) are provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The approach assumes that the black-box moderation API provides a reliable and continuous measure of harmfulness, but the API's own biases and limitations are not discussed.
- ⚠️ The method assumes that optimizing embeddings at test time does not degrade other desirable properties (e.g., fluency, coherence, or factual accuracy), but no evaluation of side effects is mentioned.
- ⚠️ The abstract claims neutralization of 'every safety-flagged response' but does not specify the diversity or difficulty of the benchmarks (e.g., adversarial prompts, edge cases).
- ⚠️ The computational cost of zeroth-order gradient estimation (e.g., number of API calls per optimization step) is not addressed, raising questions about practical scalability.
- ⚠️ The method may inadvertently introduce adversarial or unnatural embeddings that could be detected or exploited, but robustness to such detection is not explored.
