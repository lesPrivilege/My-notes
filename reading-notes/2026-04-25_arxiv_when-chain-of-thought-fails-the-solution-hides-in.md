# When Chain-of-Thought Fails, the Solution Hides in the Hidden States

- **Source**: https://arxiv.org/abs/2604.23351
- **Date**: 2026-04-25
- **Authors / Org**: Houman Mehrafarin, Amit Parekh, Ioannis Konstas
- **Type**: arxiv

## Core Problem

It is unclear whether chain-of-thought tokens are computationally useful for reasoning or merely serve as post-hoc explanations. The paper addresses the gap in understanding where and how task-relevant information is encoded within CoT traces, especially when the trace itself is incorrect.

## Core Approach

1. **Activation Patching**: Hidden states from a CoT generation run are transferred to a direct-answer run for the same question, allowing causal measurement of how individual token-level information affects final answer accuracy.

2. **Token-Level Causal Analysis**: The method patches specific token positions (e.g., verbs, entities, mathematical tokens) to isolate which types of tokens carry task-solving information versus answer-proximal content.

3. **Layer-Wise and Temporal Analysis**: The study examines how patching effects vary across model layers (mid-to-late layers show higher concentration) and across positions in the reasoning trace (information appears earlier in correct runs).

## Evidence

Patching hidden states from CoT to direct-answer runs yields substantially higher accuracy than both direct-answer prompting and the original CoT trace. Patched outputs are often shorter yet exceed the accuracy of a full CoT trace. Task-relevant information is more prevalent in correct than incorrect CoT runs and is unevenly distributed across tokens, concentrating in mid-to-late layers and appearing earlier in the reasoning trace.

## Assumptions & Open Questions

- ⚠️ The analysis assumes that activation patching faithfully transfers causal information without introducing artifacts from the patching process itself (e.g., distributional mismatch).
- ⚠️ The study is limited to GSM8K (math word problems); it is unclear whether findings generalize to other reasoning domains (e.g., commonsense, symbolic, or multi-step logical reasoning).
- ⚠️ The paper does not explore whether the recovered information is specific to the model architecture or if similar patterns hold across different model families and sizes.
- ⚠️ The claim that 'complete reasoning chains are not always necessary' is based on shorter patched outputs, but the mechanism by which shorter outputs achieve higher accuracy is not fully explained.
- ⚠️ The analysis focuses on token-level hidden states but does not address how attention patterns or inter-token interactions contribute to the encoding of task-relevant information.
