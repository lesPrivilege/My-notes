# Awakening Dormant Experts:Counterfactual Routing to Mitigate MoE Hallucinations

- **Source**: https://arxiv.org/abs/2604.14246
- **Date**: 2026-04-15
- **Authors / Org**: Wentao Hu, Yanbo Zhai, Xiaohui Hu, Mingkuan Zhao, Shanhong yu, Xue Liu, Kaidong Yu, Shuangyong Song, Xuelong Li
- **Type**: arxiv

## Core Problem

Sparse MoE models suffer from hallucinations, especially on long-tail knowledge, because static Top-k routing favors high-frequency patterns, causing specialist experts with critical rare knowledge to be under-prioritized (dormant) despite their causal importance.

## Core Approach

1. **Layer-wise Perturbation Analysis**: CoR performs a perturbation analysis at each layer to identify which experts are causally important for factual accuracy, using virtual ablation to measure the impact of removing or adding specific experts.

2. **Counterfactual Expert Impact (CEI) Metric**: CEI quantifies the causal contribution of each expert to the model's output by comparing the model's performance with and without that expert, enabling the identification of dormant experts that are crucial for long-tail knowledge.

3. **Dynamic Resource Reallocation**: CoR shifts computational resources from syntax-dominant layers to knowledge-intensive layers by re-routing tokens to causally decisive experts, while keeping the total number of activated experts constant, thus avoiding any increase in inference budget.

## Evidence

On TruthfulQA, FACTOR, and TriviaQA benchmarks, CoR improves factual accuracy by 3.1% on average compared to static Top-k routing, without increasing the inference budget, establishing a superior Pareto frontier over static scaling strategies.

## Assumptions & Open Questions

- ⚠️ The paper assumes that dormant experts are consistently beneficial for long-tail knowledge, but does not prove that awakening them never introduces new errors or biases in other contexts.
- ⚠️ The CEI metric relies on virtual ablation, which may not perfectly simulate the actual effect of removing an expert during inference, potentially overestimating or underestimating causal impact.
- ⚠️ The method is evaluated only on factual accuracy benchmarks; its effect on other aspects like fluency, coherence, or safety is not explored.
- ⚠️ The approach assumes that syntax-dominant layers are less critical for factual accuracy, but this distinction may not hold for all types of inputs or tasks.
- ⚠️ The paper does not address how CoR scales with the number of experts or layers, nor its computational overhead for the perturbation analysis step.
