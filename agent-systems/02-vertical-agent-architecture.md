# 垂直领域 Agent 架构——从概念到组件

> 目标：能画出组件架构图、能选型、能讲清数据流与风险点。

---

## Agent 系统参考架构

```
            ┌────────────────────────────────────────────┐
   用户/业务系统 ──▶│  编排层 Orchestration                       │
            │  · agent loop（思考→行动→观察）              │
            │  · LangGraph 图 / handoff / manager          │◀── 可观测 Observability
            │                                              │    · tracing(OTel)
            ├───────────────┬──────────────┬───────────────┤    · trajectory eval
            ▼               ▼              ▼               │    · token/cost
       LLM 层            RAG 管线         工具层 Tools        │
   · 模型选择/路由    · ingest:清洗/分块   · function calling │◀── 安全 Guardrails
   · 大模型+小模型     /embedding/向量库   · MCP server       │    · 输入/输出/工具三层
                    · retrieve:检索/rerank · 业务API/DB      │    · HITL 审批
                     /注入 context                          │
            └───────────────┴──────────────┴───────────────┘
                            ▲
                       记忆/状态 Memory：session / 结构化业务数据 / 向量知识 / append-only log
```

## 逐块要点

| 组件 | 关键子环节 | 选型/坑 |
|------|-----------|---------|
| **LLM 层** | 模型选择、多模型路由 | 简单步用小模型省成本；强推理用大模型 |
| **RAG 管线** | 清洗→分块→embedding→向量库→检索→rerank→注入 | 效果差 80% 出在分块和检索质量，不是模型 |
| **工具层** | function calling / MCP server / 业务 API | schema 精确、description 即 prompt、result 即 context |
| **编排层** | agent loop / 图 / handoff | 确定性可审计→图；开放探索→agent/ReAct；专业分工→handoff |
| **记忆/状态** | session / 结构化 / 向量 / log | 按持久化层级选；长任务要 compaction |
| **安全** | 输入/输出/工具 guardrail + HITL | 不可逆/高风险动作必须 HITL |
| **可观测/eval** | tracing + trajectory eval + cost | 区分「会调 API」和「会做可靠 agent」的分水岭 |

## 技术路径选型

| 场景特征 | 推荐路径 | 理由 |
|----------|----------|------|
| 通用知识问答、无私有数据 | 大模型直答 | 最简，别过度设计 |
| 答案在私有文档/知识库里 | 大模型 + RAG | 用检索补私有知识，降幻觉 |
| 高频、结构化、对延迟成本敏感 | 小模型 + 规则 | 分类/抽取用小模型更快更便宜 |
| 需要调外部系统/执行动作 | 大模型 + 工具(MCP) | tool use 让它「行动」 |
| 多步、需自主决策 | + agent 编排(ReAct/图) | 仅当确实需要自主决策才上 |

**心法**：从左到右逐级加，能用左边解决就别上右边。

## 编排范式

| 范式 | 机制 | 适用场景 |
|------|------|----------|
| **Handoff（路由式）** | Triage→specialist 接管对话 | 客服、咨询 |
| **Manager（管弦式）** | Orchestrator 控制，specialist 作为 tool | 研究、报告 |
| **Graph（图式）** | 确定性代码 + AI 混合 | 业务流程、合规敏感 |
| **ReAct（推理式）** | Thought→Action→Observation 循环 | 通用工具调用 |
