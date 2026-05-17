# DeepSeek 做 Harness：一个结构性论证

> Working Paper · Draft · 2026-05-17
> 求职附件：DeepSeek Agent Harness 产品经理岗位
> 读者：DeepSeek 研究员与核心工程师

---

## 一、外化天花板：harness 工程的结构性约束

当前 AI 工程中所有不修改模型权重的改进——prompt engineering、harness 框架、managed agents、alignment monitoring——本质上是同一件事：externalization。在不改变模型权重的前提下重组外部运行时环境。天花板不来自工程努力不足，而来自模型底层表征的几何结构，后者由训练目标（有损压缩）雕刻，不为使用需求服务（externalization-ceiling, E1）。

每个 harness 组件都是对特定模型、特定 effort level、特定任务分布下某个具体失败模式的补丁。Anthropic 为 Sonnet 4.5 的 context anxiety 设计的 reset，在 Opus 4.5 变成死代码（参见 A2）。Cat Wu 团队每次新模型发版后通读 system prompt 移除模型不再需要提醒的部分——这不是产品迭代，是脚手架拆除（参见 A3）。harness 组件的生命周期不由工程质量决定，由模型进步速度决定。投入在 harness 上的资源本质上是在给一个会缩水的资产加杠杆。

更精确地说，harness 是模型缺陷的考古层（参见 A5）。每个 harness 组件都在回答一个历史问题——「当时那一版模型做不到什么？」。Sonnet 4.5 的 context reset 在 Opus 4.5 变成 dead code。Notion 为 Opus 4.6 写的 tool-failure fallback 在 4.7 变成多余。所谓「prompt engineering 作为技能」，其折旧周期与模型发布周期严格绑定；真正跨范式不折旧的是可观测性、cost control、系统可靠性这类元能力——把它们包装成「AI 时代新技能」是修辞通胀。

这个判断对下游三方完全成立：如果商业模式建立在模型缺陷的补丁上，每一次模型更新都是一次资产减记。外部 harness 公司的整个能力空间是上游的子集，生态繁荣不是护城河，是厂商路线图没走到那的窗口期（参见 A6）。

但对上游——对 DeepSeek——这个判断需要修正。harness 的贬值逻辑假设了「模型提供方」和「harness 提供方」是两个不同的实体，贬值发生在后者的资产表上。当两者是同一实体时，harness 投资与模型投资之间的关系就从「零和」变为「内部对冲」——harness 被模型进步折旧的损失，同时是模型进步驱动的竞争力提升。这个修正不是否定 externalization ceiling 的框架，而是在同一框架内将「位置」纳入变量。

Anthropic 已经在这个路径上走了一步。Claude Code 的 98.4% 基础设施 / 1.6% AI 决策逻辑（VILA-Lab 逆向分析，参见 tech.md）说明，对于上游，「harness 变薄」和「harness 作为诊断工具」可以并行推进：在需要补丁的地方补丁越少越好，但 harness 本身作为可观测性基础设施和模型能力等高线的绘制工具，其生命周期不与特定缺陷绑定。

（本章约 950 字）

---

## 二、上游与下游的不对称：为什么 model 提供方做 harness 不同

Harness 变薄与变厚是位置函数，不是路线分歧（参见 A4）。Anthropic 说 harness 应该变薄，小米说 harness 应该变厚，两者都对但不对称。Anthropic 站在模型前沿，看到的是 harness 的递减价值；小米站在追赶期，看到的是 harness 的当下杠杆。变厚策略有内在矛盾——投入越多资源在框架上，模型进步越快让这些投入贬值。变薄策略没有这个矛盾，是收敛方向。

DeepSeek 此前的立场——「只做引擎不做车」——在这个框架里处于结构性正确的位置：资源集中在模型架构和推理效率上，agent 框架、工具链、SDK 生态是 harness 层的东西，最终都会被开源或复刻（参见 A9）。Harness 层没有持久壁垒，模型层的架构效率才有。

这个判断在「追赶期」是正确的。但当 DeepSeek 从追赶者进入至少在某些维度上的并行者甚至领先者位置时，「只做引擎不做车」从策略选择变成了自我设限。模型层的优势——CSA/HCA 混合注意力将百万上下文 KV cache 压至上一代的 10%、FP4 量化感知训练让推理经济性接近十倍提升——这些优势在缺乏 harness 层的情况下只能间接变现（通过 API 定价），而不能直接建立开发者和终端用户的粘性。

上游做 harness 与下游做 harness 有四个结构性差异：

**第一，上游的 harness 是诊断工具，下游的 harness 是产品。** 上游公开的工作流和产品设计选择，其真正价值在于暴露厂商认为模型当前能可靠执行的边界在哪里（参见 A20）。Cat Wu 团队拆掉的部分标记了模型已内化的能力，保留的部分标记了仍需外部补偿的缺陷。这种诊断价值下游无法获得——他们只能读到系统 prompt 的公开版本，看不到内部演进历史。

**第二，上游的 Demo 即标准。** OpenAI 开源航司客服 Agent demo 的杀伤力不在技术复杂度，而在它是官方出的。参考实现一旦开源，下游方案商就从卖架构设计降级为卖定制配置（参见 A8）。DeepSeek 开源的 harness 参考实现会反向定义下游生态的接口规范，而这个规范本身就是竞争壁垒的组成部件。

**第三，Prompt 效力沿架构边界集中到厂商一侧。** 同一份 prompt 文本，在个人私部署架构里接近无用，在官方 orchestration + 品牌系统 grounding + 精细控件的架构里产生断层级效果（参见 A10）。Claude Design 随 Opus 4.7 推出后，Brilliant 报告原先需 20+ prompts 的页面现在 2 prompts——不是 prompt 在贬值，是外化层的效力在沿架构边界重新分配。

**第四，上游碾压是全栈纵深，不是单点优势。** 上游厂商标配的标准定义权、开发者工具链控制、运行时基础设施（参见 A23）构成一个逐层挤压下游的体系。DeepSeek 如果不进入 harness 层，就是把工具链控制权、接口规范定义权、和开发者社区运营权拱手让给 Anthropic 和 OpenAI——而这恰恰是 coding agent 这个最大且最可持续的 agent 品类的入场券。

DeepSeek 此时下场做 harness，不是背离「只做引擎不做车」，而是「做引擎的人也需要展示引擎能跑多快」。当模型层的竞争力已经需要通过 harness 层的体验来兑现时，「不做车」从战略清晰变成了战略缺失。

（本章约 1250 字）

---

## 三、Cache hit 作为跨层设计原理

DeepSeek 在一个维度上拥有其他厂商难以复制的结构性优势：跨层一致的 cache 设计哲学。

「能离线解决的问题不要在线解决」是多层收敛的统一原理（参见 A11）。这不是工程巧合，是 memory bandwidth 作为硬瓶颈决定了所有抽象层的优化方向必然趋同：从 over-training 把高频 pattern 烧进权重、到 MoE 稀疏激活只调动相关 expert、到 KV cache 命中避免重复 prefill、到 system prompt caching 最大化前缀复用、到 TwELL 的 FFN activation sparsity——每一层都在做同一件事：把重复计算前置，运行时只处理增量（discard-capability, E1）。

DeepSeek 在这个统一原理上的实施深度超出了任何单一点的优化：

**模型层。** CSA/HCA 混合注意力将百万上下文 KV cache 压至 V3.2 的 10%、FLOPs 压至 27%。FP4 量化感知训练不是在部署时量化——是在训练阶段就让模型适应 FP4 精度，推理时直接使用 FP4 权重。推理经济性的优势根植于训练阶段，不是事后权衡。

**推理层。** ds4（antirez 的 DwarfStar 4）将 KV cache 作为磁盘的一等公民——KV checkpoints 以 SHA1 为 key 写入 .kv 文件，exact 前缀跨 session 甚至跨服务器重启复用（参见 A29）。这不是工程技巧，是架构决策：cache 的生命周期从「per session」拉长到了「per deployment」。DSML tool-call replay 进一步确保了 agent 场景下 tool call 返回结果不破坏前缀一致性——这是 Claude Code 团队用 dynamic timestamp 至今未解决的工程包袱。

**框架层。** Reasonix 的 cache-first loop 设计——三段式 context 分区（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH）——将单日 cache hit rate 推至 99.82%，435M input tokens 的成本从 $61 压缩到 $12（参见 A27）。99.82% 之所以可能，恰恰因为 Reasonix 只绑定 DeepSeek 一个后端——「coupling to one backend is the feature, not a limitation」。这不只是成本节约。它意味着 cache 设计不再是「上层框架层面的经验优化」，而是「模型 API 设计、推理引擎实现、和框架 architecture 共享同一套 cache 契约」的系统级结果。

对比 Anthropic 这边：Claude Code 的系统提示使用 dynamic timestamp，导致每次请求前缀不可 cache；prompt compression 在 autocompact.ts 里以 13,000 token buffer 为单位做「working semantics reconstruction」——本质上是用 prompt engineering 补偿 cache miss 造成的上下文管理问题。Harness 层在给模型层的 cache 设计缺陷打补丁。

DeepSeek 的位置不同：如果在模型 API 设计时就把 cache-friendly prefix 结构（immutable system + append-only log + volatile scratch）作为一等契约——类似 ds4 的 KV checkpoint keying 策略从推理层上移至 API 设计层——那么 harness 层不需要「补偿」模型层的 cache 缺陷，只需「编排」模型层已保证的 cache 能力。这种跨层一致性对任何第三方 harness 都不可复制——因为没有控制推理引擎和模型 API 的能力。

Cache hit 不是性能优化问题，是 harness 架构的核心设计变量。DeepSeek 如果能在 harness 产品中把 cache hit 从「尽力而为的优化」变为「系统级保证」，其 agent 产品的单位经济学将构成对手无法追赶的护城河——不是因为 prompt 写得更好，而是因为 cache 契约从模型训练一路贯穿到终端用户 session 管理。

（本章约 1300 字）

---

## 四、Coding agent 的商业可持续性与飞轮基础

coding 领域是当前唯一同时满足市场规模和商业可持续性的 agent 品类。不是巧合——它具备三个其他领域结构性缺位的条件（参见 A17）：可分解的中间表示（代码本身可局部编辑）、廉价的自动验证（编译器/测试套件）、错误可逆且代价低。

这三个条件组合起来构成一个飞轮（参见 A15）：context 采集层（agentmemory 等 persistent memory，代码仓库天然提供结构化的历史）→ 微调信号层（issue→PR 的天然 (input, output, ground_truth) 三元组）→ 验证闭环层（测试套件 + type checker + compiler）。三层基础设施每一层都有成熟方案，飞轮自转。其他领域（法律、医疗、金融）段段卡住：输入端的 context 散落在邮件链和口耳相传中，中间缺训练信号的三元组，输出端缺自动验证器。

这个不对称决定了 coding agent 的商业逻辑与其他 agent 品类完全不同。在验证器缺位的领域，prompt engineering 的繁荣本身就是症状——用户能操控的界面只剩 prompt 一个旋钮，prompt 模板市场本质是「用经验规则近似替代反馈回路」（参见 A18）。当模型迭代打破这些经验规则的拟合对象时，之前积累的「prompt 资产」归零。但在 coding 领域，验证器在场，prompt 的效力不是靠对模型行为的经验拟合来判断的——编译器说你错了就是错了。这意味着 coding harness 的价值不完全随模型贬值，因为一部分质量保证来自模型外部的确定性检验器。

这个判断对 DeepSeek 做 coding agent 产品策略有直接含义：

**第一，产品向量应该朝飞轮方向倾斜。** 不只是「用 DeepSeek 写代码更快」，而是将用户使用过程产生的信号（代码 → 测试结果 → 修复轨迹 → 最终采纳/拒绝）组织成可反馈的 (input, output, reward) 结构。Claude Code 在这方面设计上几乎完全缺失——它不追踪用户最终采纳了 agent 的哪些建议、拒绝了哪些、修复了什么。

**第二，agent 加速带来的 entropy 问题需要 harness 侧的工程纪律对冲，而非更多 agent（参见 A16）。** 这个悖论级洞察意味着 harness 的价值不是「让 agent 跑更多步」，而是「在 agent 跑完每一步之后判断这一步是否让代码库整体变好了」。如果有好的 model-to-harness 反馈回路，这个判断可以被逐步自动化。

**第三，估值叙事从「市场渗透率」转向「飞轮转动速度」。** 当前市场定价隐含 AI 均匀渗透各行业的假设，实际渗透分布极度不均（参见 A19 和 B28）。Coding 是唯一接近范式替换的领域。但范式替换不是用 AI 替代程序员——是重构「谁写代码、谁验代码、谁修代码」的工作流。Harness PM 的产品判断力不体现在「做了多少个 agent 功能」，体现在「哪些工作流节点应该留给人、哪些交给 agent、哪些交给确定性规则」。

coding agent 的可持续性不是「模型能不能写更好的代码」——那部分由模型团队负责。可持续性来自「验证器在场」这一结构性事实，以及 harness 层能否将验证结果系统地转化为模型改进信号。DeepSeek 如果能在产品中闭合这个回路，其 coding agent 将同时拥有更好的模型（因为有结构化反馈数据）和更低的维护成本（因为 harness 组件不需要猜测模型行为，有编译器/测试结果作为 ground truth）。

（本章约 1300 字）

---

## 五、Harness PM 的位置：在 model-harness 共同进化中

Harness PM 的默认脚本是「搜集用户反馈 → 转化为功能需求 → 排优先级 → 交付 feature」。这个脚本对 coding agent 不适用。不是因为产品管理方法论有问题，而是因为产品本身的特性（模型能力边界）每 8-12 周被重写一次。

Harness PM 的第二脚本更隐蔽也更常见：「研究 Anthropic/OpenAI 的最新 release → 反向工程他们的 design decisions → 复制到自己的产品中」。这个脚本的合理性来自一个事实——上游厂商公开的 harness 设计选择确实暴露了模型当前能力边界的等高线（参见 A20）。但这个脚本有两个致命缺陷：复制的是别人对自己模型的诊断结果（对你未必适用），且复制的总是落后一拍的快照（对方已经在绘制下一个等高线）。

Harness PM 的正确位置不在上述两个脚本之中。它在三个维度的交叉处：

**第一个维度：model 能力等高线的持续绘制。** 弱模型不知道自己到了能力边界（参见 A21）。这是模型 calibration 的内部问题，不属于 harness 能解决的范畴。但 harness 层可以成为这个问题的探测系统——观测模型在哪些任务类型上频繁拒绝、哪些类型上沉默失败（做了但做错了、没有报错）、哪些类型上反复尝试但仍失败。这些信号不告诉模型团队「怎么修」，但告诉他们「哪里需要修」。Cat Wu 团队的 system prompt 删减操作本质上就是这件事——只是目前做在文档和版本控制里，没有变成产品化的反馈系统。

**第二个维度：推理预算与元认知判断。** Pro 在 agentic fetch 任务中表现比 Flash 更差，不是因为能力不足，而是更高的推理预算让它在死胡同里「想得更多」而非「更早放弃」（参见 A25）。推理能力的增加不等于元认知判断的改善——知道什么时候该停是一个独立于推理深度的能力维度。Harness 层在这里的位置是：设计外部的「止损」机制（超时、结果验证、多路径并行后择优）来补偿模型内部的元认知缺口。这本质上是在 harness 层做 discard function（discard-capability, E3-E4）：把discard 决策从模型内部（它不擅长）上移至 harness 层（可以更具确定性地编码）。

**第三个维度：harness 的非贬值价值。** externalization-ceiling 的 E1-E7 聚焦于 harness 技术补丁价值的贬值，但 W5 识别出几个维度的价值不随模型进步消失（externalization-ceiling, W5）：合规与治理（audit trail、policy enforcement）是制度需求；可观测性（logging、tracing、failure mode taxonomy）是运行时基础设施；多模型编排（底层模型频繁切换时统一接口层价值上升）；诊断记录（哪些 harness 被拆了的历史本身就是信号）。这些维度共同构成 harness 的「OS 层」——不随任何单一模型版本更新而失效。Harness PM 的产品判断力体现在：知道哪些功能是给当前模型版本打补丁（需要预设贬值时间窗口），哪些是给 harness OS 层增加持久能力（不依赖特定模型行为）。

DeepSeek Harness PM 的工作正位不在上述任何一个维度单独地，而在三个维度的交线：用 harness 的可观测性基础设施绘制模型能力的等高线 → 用等高线信息指导「哪些因模型缺陷而建的补丁可以拆了、哪些止损机制仍需保留」→ 用产品接口把「模型越来越好 → harness 越来越薄」这个外在贬值过程变成用户可感知的价值递送。

这也是对「只做引擎不做车」的最终修正：引擎本身在变好，但变好的速度在用户不可见的权重空间里发生。Harness 层把这个「变好」翻译成用户侧的体验差异。翻译质量决定用户是否感知到模型进步，进而决定模型进步的研发投入是否转化为市场竞争力。

Harness PM 不是做补丁的人。Harness PM 是参与模型能力等高线绘制的人——只是用的工具不是训练脚本，是产品设计。

（本章约 1400 字）

---

*全文字数：约 6200 字*
*来源：externalization-ceiling 论文 + discard-capability 论文 + Category A 条目 31 条 + Reading/ 根目录 harness 复盘素材*
*引用标注格式：(参见 An) 对应分类报告中的 Category A 条目编号，最终稿中需替换为正式引用*
