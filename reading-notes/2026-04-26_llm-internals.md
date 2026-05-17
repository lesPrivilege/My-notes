# LLM Internals

- **来源**: github.com/amitshekhariitbhu/llm-internals
- **日期**: 2026-04-12 创建，2026-04-25 最后推送
- **作者**: Amit Shekhar（Outcome School 创始人）
- **类型**: repo（教育资源 / 教程系列）
- **星数**: 640，forks: 49
- **许可证**: Apache-2.0

## 这是什么

一个**博客 + 视频教程**的知识集合，而非一个软件项目。内容是从 tokenization 到 inference optimization 的 LLM 底层理解，以视频和文字形式呈现（内容托管在 Outcome School 网站和 YouTube）。

## 覆盖的内容

- LLM 基本概念（含 RAG、MCP、Agent、Fine-tuning、Quantization 的概述）
- Tokenization 与 Byte Pair Encoding（BPE）
- Attention 的数学原理：Q、K、V、√dₖ 缩放因子、Causal Masking
- Backpropagation 的数学原理（含数值示例）
- Cross-Entropy Loss 的数学原理
- Transformer 架构逐块解构（Encoder/Decoder、Multi-Head Attention、FFN、Residual Connection、Layer Norm）
- Feed-Forward Networks in LLMs
- KV Cache（资源中提及的 inference optimization 相关主题）

## 评价

**优势：**
- 结构清晰，从底层概念逐步向上构建，适合初学者入门
- 每个主题都有 step-by-step 的数值示例（Attention QKV、Backpropagation 尤其详细）
- 视频 + 文字双重形式，学习路径完整

**注意点：**
- 这是一个**视频/博客索引**，不是代码库——repo 本身只是一个 README + banner 图片，没有任何实际代码或 notebook
- 深度有限：覆盖的是标准 Transformer 和基础数学，不涉及 MoE、GQA、MLA、speculative decoding、distillation 等进阶推理优化主题
- 极新（2026-04-12 创建，未满一个月），内容还在持续增加中
- 640 stars 在如此新的 repo 中算不错的起步，但尚未经过社群大规模验证

**适合谁：** 想从底层数学理解 Transformer 的 LLM 初学者。不适合想学实现（代码）或想了解前沿优化的读者。
