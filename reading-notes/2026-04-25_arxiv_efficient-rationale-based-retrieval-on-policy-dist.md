# Efficient Rationale-based Retrieval: On-policy Distillation from Generative Rerankers based on JEPA

- **Source**: https://arxiv.org/abs/2604.23336
- **Date**: 2026-04-25
- **Authors / Org**: Teng Chen, Sheng Xu, Feixiang Guo, Xiaoyu Wang, Qingqing Gu, Hongyan Li, Luo Ji
- **Type**: arxiv

## Core Problem

Traditional rationale-based retrieval requires cross-encoding of query-document pairs with large language models, leading to high computational costs. Existing independent encoding methods lack the cross-query-document comprehension needed for rationale-based tasks.

## Core Approach

1. **Teacher Generative Reranker Training**: A LLM-based generative reranker is trained by placing the document before the query and using log probabilities to generate relevance scores, serving as the teacher model.

2. **On-policy Distillation with JEPA**: Rabtriever (student) is initialized from the teacher with frozen parameters. A lightweight, trainable predictor is inserted between LLM layers and heads, projecting the query embedding into a new hidden space using the document embedding as a latent vector, then minimizing the distribution difference between this projected embedding and the teacher embedding.

3. **Auxiliary Reverse KL Loss**: An auxiliary loss on the reverse KL divergence of LLM logits is added to reshape the student's logit distribution, improving sampling efficiency during on-policy distillation.

## Evidence

Rabtriever outperforms various retriever baselines on rationale-based tasks (empathetic conversations, robotic manipulations) with minor accuracy degradation from the reranker. It also generalizes well on traditional benchmarks MS MARCO and BEIR, achieving comparable performance to the best retriever baseline. The model reduces teacher's quadratic complexity on document length to linear, verified both theoretically and empirically.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the JEPA-based projection and distribution matching effectively capture cross-query-document comprehension without explicit cross-attention, but does not provide ablation studies isolating the contribution of JEPA versus the distillation framework.
- ⚠️ The 'minor accuracy degradation' from the reranker is not quantified with specific numbers, making it unclear how much performance is sacrificed for efficiency.
- ⚠️ The approach assumes that the teacher reranker's log-probability-based relevance scoring is optimal for rationale-based tasks, but this may not generalize to all types of rationale (e.g., causal or counterfactual reasoning).
- ⚠️ The on-policy distillation relies on the teacher's embeddings being available during training, which may limit applicability to scenarios where the teacher is not accessible or is too large to run repeatedly.
- ⚠️ The generalization to traditional retrieval benchmarks (MS MARCO, BEIR) is claimed but not compared against state-of-the-art dense retrievers (e.g., ColBERT-v2, SPLADE), leaving open whether Rabtriever is competitive in standard ad-hoc retrieval.
