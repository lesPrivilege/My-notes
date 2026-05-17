# FinChain: A Symbolic Benchmark for Verifiable Chain-of-Thought Financial Reasoning

- **Source**: https://arxiv.org/abs/2506.02515
- **Date**: 2025-06-03
- **Authors / Org**: Zhuohan Xie, Daniil Orel, Rushil Thareja, Dhruv Sahnan, Hachem Madmoun, Fan Zhang, Debopriyo Banerjee, Georgi Georgiev, Xueqing Peng, Lingfei Qian, Jimin Huang, Jinyan Su, Aaryamonvikram Singh, Rui Xing, Rania Elbadry, Chen Xu, Haonan Li, Fajri Koto, Ivan Koychev, Tanmoy Chakraborty, Yuxia Wang, Salem Lahlou, Veselin Stoyanov, Sophia Ananiadou, Preslav Nakov
- **Type**: arxiv

## Core Problem

Existing financial reasoning benchmarks like FinQA and ConvFinQA focus on final numerical answers and neglect the intermediate reasoning steps needed for transparency and verification, limiting the assessment of multi-step symbolic reasoning in finance.

## Core Approach

1. **Parameterized Symbolic Templates**: Each of the 58 topics across 12 financial domains is represented by a symbolic template with parameters, allowing generation of diverse, contamination-free instances by varying inputs.

2. **Executable Python Code for Verification**: Each template includes executable Python code that produces ground-truth reasoning steps and final answers, enabling fully machine-verifiable evaluation of chain-of-thought outputs.

3. **CHAINEVAL Dynamic Alignment Measure**: A novel evaluation metric that jointly assesses final-answer correctness and step-level reasoning consistency, providing a more comprehensive measure of reasoning quality than accuracy alone.

## Evidence

Evaluation of 26 leading LLMs shows that even frontier models have clear limitations in symbolic financial reasoning, while domain-adapted and math-enhanced fine-tuned models substantially narrow the gap. Specific numerical results are not provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The benchmark assumes that symbolic templates with executable Python code capture the full range of real-world financial reasoning, which may oversimplify nuanced or qualitative financial analysis.
- ⚠️ The CHAINEVAL metric assumes that step-level consistency is a reliable proxy for reasoning quality, but it may penalize valid alternative reasoning paths that are equally correct.
- ⚠️ The abstract does not report inter-annotator agreement or human baseline performance, leaving open the question of how well the benchmark aligns with human expert reasoning.
- ⚠️ The claim of 'contamination-free' data generation assumes that parameterized templates prevent memorization, but models may still overfit to template structures if exposed during training.
