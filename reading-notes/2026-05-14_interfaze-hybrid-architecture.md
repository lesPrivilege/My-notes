# Interfaze：为高精度大规模推理设计的混合模型架构

- **来源**: https://interfaze.ai/blog/interfaze-a-new-model-architecture-built-for-high-accuracy-at-scale
- **论文**: arXiv:2602.04101，已被 IEEE CAI 2026 接收
- **日期**: 2026-02-04（arXiv），2026-05-11（博客）
- **作者/机构**: Harsha Vardhan Khurdula, Vineet Agarwal, Yoeven D Khemlani / JigsawStack, Inc.（YC 孵化）
- **类型**: article + paper
- **置信度**: high
- **模式**: deep-review

## 核心问题

现有 AI 架构在处理确定性开发任务（OCR、结构化输出、语音识别、目标检测）时面临两难：通用大模型（如 GPT-5.5、Claude Opus 4.7）在特定任务上成本高、速度慢；而纯 DNN/CNN 专用模型虽然精度高、成本低，但缺乏灵活性和语义理解能力（如 CNN 能提取护照上的出生日期，但无法计算年龄）。开发者被迫在"灵活但贵"和"便宜但死板"之间做选择。

## 核心方案

Interfaze 提出一种混合架构（Hybrid Architecture），将 DNN/CNN 专用小模型的精确性、低成本与 Transformer 通用模型的灵活性结合。架构分三层：

1. **异构 DNN/CNN 感知栈（Perception Stack）** — 一组任务专用的小模型：
   - OCR 与文档布局分析（文本框、bounding box、置信度分数）
   - YOLO+SAM 2 驱动的目标检测与 GUI 解析
   - 多语言 ASR 含说话人分离（卷积 + 自注意力编码器）
   - 轻量级图像/文本分类器

2. **上下文构建层（Context Construction Layer）** — 爬取、索引、解析外部数据（网页、代码、PDF、图表），压缩为四个结构化 schema 字段：`observations`、`entities`、`relations`、`provenance`。

3. **行动层 + 薄控制器（Action Layer + Thin Controller）** — 控制器决定调用哪些小模型和工具（浏览、检索、沙盒执行代码、驱动 headless 浏览器），然后将蒸馏后的结构化上下文（而非原始像素/波形/整个网页）转发给用户选定的 LLM 生成最终响应。

**核心设计哲学**：大 LLM 从不见原始输入，只处理压缩后的结构化上下文。大部分计算从昂贵的大模型转移到便宜的小模型栈。

**API 层面**：兼容 OpenAI Chat Completions API，可 drop-in 替换。支持部分模型激活（partial model activation）——在 system prompt 中声明 `<interfaze_mode>ocr</interfaze_mode>` 即可只跑 OCR 子模型。

## 证据

**论文基准测试（Interfaze-Beta）：**

| 基准 | Interfaze | 对比参考 |
|------|-----------|---------|
| MMLU-Pro | 83.6% | — |
| GPQA-Diamond | 81.3% | — |
| AIME-2025 | 90.0% | — |
| LiveCodeBench v5 | 57.8% | — |
| MMMU (val) | 77.3% | — |
| AI2D | 91.5% | — |
| ChartQA | 90.9% | — |
| Common Voice v16 | 90.8% | — |
| **OCRBench V2** | **70.7%** | Gemini-3-Flash 55.8% |
| olmOCR（复杂文档）| 85.7% | — |
| SOB 值准确率 | 79.5% | 所有 28 模型（含 Pro 级）中领先 |

**STT 性能**：每秒处理 209 秒音频，约 Deepgram Nova-3 的 1.5 倍、Gemini-3-Flash 的 11 倍。

**定价**：约 $1.50/百万输入 tokens，$3.50/百万输出 tokens（Gemini-3-Flash 同一价位）。

**同行评审**：论文已被 IEEE CAI 2026 接收。

## 风险与弱点

- ⚠️ **"小模型栈"的具体构成未完全开源**：论文描述了架构思路，但 DNN/CNN 感知层的具体模型选择、训练数据和权重未见公开。这是 JigsawStack 的商业产品，不是纯研究开源项目。
- ⚠️ **薄控制器本身的质量瓶颈**：所有请求首先经过控制器路由。如果控制器误判任务应该调用哪个子模型，错误会级联。论文未充分讨论控制器的失败模式。
- ⚠️ **蒸馏上下文的"压缩损失"**：将原始数据（像素、波形、长文档）压缩为四个结构化字段意味着信息损失。对于非典型或边缘情况，丢失的信息可能正是关键信号。
- ⚠️ **Lean 架构，重的外部依赖**：Interfaze 本身轻量，但依赖外部 LLM（用户选定的模型）做最终回答。这意味着最终输出质量受制于所选 LLM，定价中的"推理"部分可能隐含双重计费。
- ⚠️ **与一般性基准的可比性问题**：在 MMLU/GPQA 等通用基准上得分与同价位模型相当，但宣称的主要优势（OCR、结构化输出）缺乏标准化第三方审计。
- ⚠️ **SOB 是他们自建的基准**：Structured Output Benchmark（SOB）由 Interfaze 团队发布，测试方法虽公开但缺少外部独立复现。

## 待验证问题

- 薄控制器的路由决策逻辑是否可解释？当它把一个 OCR 请求误路由到分类模型时，如何检测和恢复？
- 四种 schema 字段（observations/entities/relations/provenance）的压缩比和保真度如何？是否存在典型的信息丢失模式？
- Interfaze 声称"大 LLM 不见原始输入"——但用户选择的 LLM（如 Claude 或 GPT）是否要为此改动做适配，还是完全透明？
- 部分模型激活模式（`<interfaze_mode>`）下，输出 schema 是否完全固定？这与通用 API 模式的灵活性如何权衡？
- 同一 JigsawStack 还运营 JigsawStack 平台（一个更泛化的 AI API 聚合），Interfaze 与其的关系和定位差异是什么？
