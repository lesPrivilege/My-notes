# Agent 开发范式——各厂商实践与标准编定

> 来源：Anthropic「Building Effective Agents」、OpenAI Agents SDK、Google ADK、Microsoft AutoGen、DSPy、MCP/A2A 协议

---

## 一、什么是 Agent

Anthropic 的定义（2024-12）：

> **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**.
> **Agents**, on the other hand, are systems where LLMs **dynamically direct their own processes** and tool usage.

**Workflow 是人编排 LLM，Agent 是 LLM 编排自己。**

判断标准（从简到繁）：

| 复杂度 | 方案 | 适用场景 |
|--------|------|----------|
| 0 | 单次 LLM + RAG | 问答、摘要 |
| 1 | Prompt chaining | 固定步骤任务 |
| 2 | Routing | 分类后走不同流程 |
| 3 | Parallelization | 独立子任务并行 |
| 4 | Orchestrator-worker | 动态分解任务 |
| 5 | Evaluator-optimizer | 生成→评估→修正 |
| 6 | Full Agent | 完全自主决策 |

---

## 二、各厂商实践

### OpenAI Agents SDK

两个原语：**Agent**（instructions + tools + handoffs）和 **Handoff**（agent 间转接，建模为 tool call）。

垂直领域样板：customer_service（航空客服，triage+handoff）、financial_research_agent（Plan-and-Execute）、personal_shopper。

### Anthropic

「找最简单的方案」。两条路线：
- **Messages API + tool_use**：开发者自建 agent loop
- **Managed Agents**：Anthropic 托管 agent harness（sandbox + session + tracing）

MCP 是工具连接标准。

### Google ADK

多语言（Python/TS/Go/Java/Kotlin）。Graph Workflows（确定性+AI 混合）是特色。内置 evaluation 框架。

### Microsoft AutoGen

事件驱动架构。分布式 agent 支持（gRPC runtime）。MCP 集成。

### DSPy

「Program, don't prompt」。Signature（声明式任务）+ Module（执行策略）+ Optimizer（自动调优）。适合需要持续优化准确率的环节。

---

## 三、标准范式编定

### 工具连接：MCP

MCP（Model Context Protocol）是 agent-tool 连接的事实标准。JSON-RPC 2.0 over stdio/SHE/HTTP。Anthropic 主导，OpenAI、Google、Microsoft 均已支持。

### Agent 间通信：A2A

Google 主导的 agent 间通信协议。MCP 纵向（agent↔tool），A2A 横向（agent↔agent）。

### 可观测：OTel

OpenTelemetry 正在成为 agent observability 的标准协议。

### 范式稳定性分层

```
标准层（6-12 个月更新）：MCP / A2A / OTel / Chat Completions API
范式层（12-18 个月有效）：ReAct / Handoff / Manager / Graph / Guardrails
框架层（3-6 个月刷新）：Agents SDK / ADK / AutoGen / PydanticAI / LangChain
应用层（持续演进）：领域 SOP → Instructions / 领域 API → MCP Server
```

**关键判断**：投资标准层和范式层，框架层是可替换的中间件。
