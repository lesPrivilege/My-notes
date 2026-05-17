# Marco-MoE: Open Multilingual Mixture-of-Expert Language Models with Efficient Upcycling

- **Source**: https://arxiv.org/abs/2604.25578
- **Date**: 2026-04-28
- **Authors / Org**: Fan Jiang, Yu Zhao, Chenyang Lyu, Tianqi Shi, Yichao Du, Feihu Jiang, Longyue Wang, Weihua Luo
- **Type**: arxiv

## Core Problem

Existing multilingual models often suffer from interference between languages when scaling to more languages, and dense models require high computational cost for comparable performance. There is a need for efficient, open-source models that balance performance, sparsity, and multilingual capability without proprietary constraints.

## Core Approach

1. **Extreme Sparse MoE Design**: Only about 5% of total parameters are activated per input token, drastically reducing computational cost while maintaining model capacity.

2. **Upcycling from Dense Models**: The MoE models are initialized from pre-trained dense models, allowing efficient reuse of learned representations and reducing pre-training compute requirements.

3. **Post-training for Instruct Variants**: After pre-training, models are further fine-tuned to create Marco-MoE-Instruct, which achieves superior performance with 3–14× fewer activated parameters compared to competing models.

4. **Structured Expert Activation Patterns**: The model learns to activate shared experts for related languages and specialized experts for linguistically isolated ones, enabling efficient multilingual handling without interference.

## Evidence

Models surpass similarly-sized competitors on English and multilingual benchmarks, achieving best-in-class performance-to-compute ratio. Instruct variants outperform models with 3–14× more activated parameters. Pre-training on 5T tokens with only 5% parameter activation per token.

## Assumptions & Open Questions

- ⚠️ The paper assumes that upcycling from dense models is universally beneficial, but does not compare against training MoE from scratch with the same compute budget.
- ⚠️ The claim of 'no interference typical of dense models' for language expansion is not quantitatively validated with controlled experiments on language addition.
- ⚠️ The extreme sparsity (5% activation) may lead to underutilization of expert capacity for low-resource languages, which is not explored.
- ⚠️ The benchmarks used for multilingual evaluation are not specified; results may be sensitive to benchmark selection and language coverage.
- ⚠️ The open-source release of full datasets and recipes is stated but not verified; reproducibility depends on actual availability and quality of disclosed materials.
