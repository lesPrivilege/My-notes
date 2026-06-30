# 合同审批路由 Agent——设计文档

> 一个基于 LangGraph 的合同审批路由 Agent 演示项目。展示如何用 Agent 技术增强企业合同审批系统。

---

## 核心论点

规则引擎覆盖标准审批路径；Agent 处理规则引擎做不好的场景——模糊分类、风险识别、流程引导。

## 架构

```
合同发起 → Triage(规则+LLM) → RAG(SOP检索) → Route(查审批矩阵)
         → Guardrails(项目评审/跨板块隔离) → HITL(大额/不可逆)
         → Audit Log(append-only)
```

### 每个节点的实现方式

| 节点 | 方式 | 为什么 |
|------|------|--------|
| Triage | 规则优先 + LLM 兜底 | 标准路径确定性，边界场景用 LLM |
| RAG | 检索 SOP/模板 | 非结构化文本，适合语义检索 |
| Route | 纯代码查表 | 结构化数据，查表零成本 |
| Guardrail | 纯代码条件判断 | 规则明确，不需要 LLM |
| HITL | `interrupt()` + `Command(resume=...)` | LangGraph 原生支持 |
| Audit Log | append-only JSONL | 不依赖 LLM |

## 数据设计

所有数据虚构。板块 A-D 代表不同业务板块，其中 A 为受监管型、C 为竞争性型。

四个测试场景：

| ID | 场景 | 预期行为 |
|----|------|----------|
| C001 | 正常审批 | 自动通过 |
| C002 | 大额 HITL | 触发人工确认 |
| C003 | 项目未评审 | Guardrail 阻断 |
| C004 | 跨板块拦截 | Guardrail 阻断 |

## 设计决策

1. **为什么不用纯 Agent？** 审批需要确定性——同一个合同跑两次，审批链必须一样。LLM 的输出是概率性的，不适合做审批路由。
2. **为什么不用纯规则引擎？** 规则引擎处理不了模糊场景——混合板块的合同，用户不知道该选哪个分类入口。
3. **为什么 LangGraph？** 图结构让流程显式可审计；`interrupt()` 让 HITL 变得简单；Checkpointing 支持跨天暂停恢复。
4. **为什么 HITL？** 不可逆操作（签署、大额支付、担保）必须等人工确认。
5. **为什么 Guardrails？** 项目未评审就阻断、跨板块需合规审查——这些是硬约束，不能靠 LLM 判断。

## 运行

```bash
git clone https://github.com/lesPrivilege/contract-approval-agent
cd contract-approval-agent
uv sync
uv run python main.py --list
uv run python main.py --contract C001
uv run pytest tests/ -v   # 17/17 passed
```

## 参考信源

- Anthropic「Building Effective Agents」
- LangGraph HITL docs
- OpenAI Agents SDK airline example
