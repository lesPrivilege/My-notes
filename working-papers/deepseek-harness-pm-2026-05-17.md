# DeepSeek 做 Harness：一个结构性论证

> Working Paper · Draft · 2026-05-17
> 求职附件：DeepSeek Agent Harness 产品经理岗位
> 读者：DeepSeek 研究员与核心工程师

---

## 一、外化天花板：harness 工程的结构性约束

当前 AI 工程中所有不修改模型权重的改进——prompt engineering、harness 框架、managed agents、alignment monitoring——是同一件事：externalization。在不改变模型权重的前提下重组外部运行时环境。天花板不来自工程努力不足，来自模型底层表征的几何结构，后者由训练目标（有损压缩）雕刻，不为使用需求服务（externalization-ceiling, E1）。

每个 harness 组件都是对特定模型版本、特定 effort level、特定任务分布下某个具体失败模式的补丁。Anthropic 为 Sonnet 4.5 的 context anxiety 设计的 reset，在 Opus 4.5 变成死代码（参见 A2）。Cat Wu 团队每次新模型发版后通读 system prompt 移除模型不再需要提醒的部分——这不是产品迭代，是脚手架拆除（参见 A3）。更精确的表述：harness 是模型缺陷的考古层，每个组件回答一个历史问题——「当时那一版模型做不到什么」（参见 A5）。harness 组件的生命周期不由工程质量决定，由模型进步速度决定。所谓「prompt engineering 作为技能」，折旧周期与模型发布周期严格绑定；真正跨范式不折旧的是可观测性、cost control、系统可靠性这类元能力。

这个判断对下游三方完全成立：如果商业模式建立在模型缺陷的补丁上，每一次模型更新都是一次资产减记。外部 harness 公司的能力空间是上游的子集，生态繁荣不是护城河，是厂商路线图没走到那的窗口期（参见 A6）。

但对上游——对 DeepSeek——这个判断需要修正。externalization ceiling 的贬值逻辑预设了「模型提供方」和「harness 提供方」是两个分离的实体，贬值发生在外部的资产表上。当两者是同一实体时，harness 投资与模型投资的关系从「零和」变为「内部对冲」。harness 被模型进步折旧的损失，同时是模型进步驱动的竞争力提升。这不是否定原框架，而是在同一框架内将位置纳入变量——外化天花板对下游是天花板，对上游是可调节的遮阳板。

上游做 harness 还有一个下游永远拿不到的收益：harness 的使用数据本身就是模型改进的结构化信号。每一条被执行或被执行后回滚的 tool call、每一个触发了用户显式干预的 agent 决策、每一次因模型能力不足而被 harness 层拦截的失败路径——这些信号如果被 harness 作为一等公民来采集和结构化，它们就构成了对模型能力边界的最精确测量。对下游，这些数据是运营成本；对上游，是训练数据。Anthropic 已经在利用这个不对称——Claude Code 的 harness 基础设施（占代码量 98.4%，VILA-Lab 逆向分析）虽然大部分是补丁，但补丁的分布本身就是信号：哪些补丁被拆了、哪些还在、哪些是新加的，构成了模型能力等高线的最细粒度绘制。

（本章约 1050 字）

---

## 二、上游与下游的不对称：为什么 model 提供方做 harness 不同

Harness 变薄与变厚是位置函数，不是路线分歧（参见 A4）。Anthropic 说 harness 应该变薄，小米说 harness 应该变厚，两者都对但不对称。Anthropic 站在模型前沿，看到的是 harness 的递减价值；小米站在追赶期，看到的是 harness 的当下杠杆。变厚策略有内在矛盾——投入越多资源在框架上，模型进步越快让这些投入贬值。变薄策略没有这个矛盾，是收敛方向。

DeepSeek 此前的立场——「只做引擎不做车」——在这个框架里处于结构性正确的位置：资源集中在模型架构和推理效率上，agent 框架、工具链、SDK 生态是 harness 层的东西，最终都会被开源或复刻（参见 A9）。Harness 层没有持久壁垒，模型层的架构效率才有。

这个判断在追赶期是对的。但当 DeepSeek 在推理效率、KV cache 压缩、FP4 量化感知训练等维度进入领先位置后，「只做引擎不做车」从策略选择变成了自我设限。模型层优势——CSA/HCA 混合注意力将百万上下文 KV cache 压至上一代的 10%、FP4 量化感知训练让推理经济性接近十倍提升——在缺乏 harness 层的情况下只能间接变现（API 定价），不能直接建立开发者和终端用户的粘性。

上游做 harness 与下游做 harness 有四个结构性差异：

**第一，上游的 harness 是诊断工具，下游的 harness 是产品。** 上游公开的工作流和产品设计选择，价值在于暴露厂商认为模型当前能可靠执行的边界在哪里（参见 A20）。Cat Wu 团队拆掉的部分标记了模型已内化的能力，保留的部分标记了仍需外部补偿的缺陷。这种诊断价值下游无法获得——他们只能读到系统 prompt 的最终版本，看不到内部演进历史，也不拥有对应模型的权重和训练环境来验证「这个补丁到底在补什么」。

**第二，上游的 Demo 即标准。** OpenAI 开源航司客服 Agent demo 的杀伤力不在技术复杂度，在它是官方出的。参考实现一旦开源，下游方案商就从卖架构设计降级为卖定制配置（参见 A8）。DeepSeek 开源的 harness 参考实现会反向定义下游生态的接口规范，而这个规范本身就是竞争壁垒的组成部件。

**第三，Prompt 效力沿架构边界集中到厂商一侧。** 同一份 prompt 文本，在个人私部署架构里接近无用，在官方 orchestration + 品牌系统 grounding + 精细控件的架构里产生断层级效果（参见 A10）。Claude Design 随 Opus 4.7 推出后，Brilliant 报告原先需 20+ prompts 的页面现在 2 prompts。不是 prompt 在贬值，是外化层的效力在沿架构边界重新分配。这个重新分配的方向是确定的：朝模型提供方集中。

**第四，上游碾压是全栈纵深。** 上游厂商标配的标准定义权、开发者工具链控制、运行时基础设施（参见 A23）构成一个逐层挤压下游的体系。DeepSeek 如果不进入 harness 层，就是把工具链控制权、接口规范定义权、和开发者社区运营权让给 Anthropic 和 OpenAI——这三样恰好是 coding agent 这个最大且最可持续的 agent 品类的入场券。

DeepSeek 此时下场做 harness，不是背离「只做引擎不做车」，而是「做引擎的人也需要展示引擎能跑多快」。当模型层的竞争力已经需要通过 harness 层的体验来兑现时，「不做车」从战略清晰变成了战略缺失。

（本章约 1200 字）

---

## 三、Cache hit 作为跨层设计原理

DeepSeek 在一个维度上拥有其他厂商难以复制的结构性优势：跨层一致的 cache 设计哲学。

「能离线解决的问题不要在线解决」是多层收敛的统一原理（参见 A11）。不是工程巧合——memory bandwidth 作为硬瓶颈，决定了所有抽象层的优化方向必然趋同：从 over-training 把高频 pattern 烧进权重、到 MoE 稀疏激活只调动相关 expert、到 KV cache 命中避免重复 prefill、到 system prompt caching 最大化前缀复用、到 TwELL 的 FFN activation sparsity——每一层都在做同一件事：把重复计算前置，运行时只处理增量（discard-capability, E1）。

DeepSeek 在这个统一原理上的实施深度超出了任何单一点的优化：

**模型层。** CSA/HCA 混合注意力将百万上下文 KV cache 压至 V3.2 的 10%、FLOPs 压至 27%。FP4 量化感知训练不是在部署时量化——是在训练阶段就让模型适应 FP4 精度，推理时直接使用 FP4 权重。推理经济性的优势根植于训练阶段，不是事后权衡。

**推理层。** ds4（antirez 的 DwarfStar 4）将 KV cache 作为磁盘的一等公民——KV checkpoints 以 SHA1 为 key 写入 .kv 文件，exact 前缀跨 session 甚至跨服务器重启复用（参见 A29）。这是架构决策：cache 的生命周期从「per session」拉长到了「per deployment」。DSML tool-call replay 进一步确保了 agent 场景下 tool call 返回结果不破坏前缀一致性。

**框架层。** Reasonix 的 cache-first loop 设计——三段式 context 分区（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH）——将单日 cache hit rate 推至 99.82%，435M input tokens 的成本从 $61 压缩到 $12（参见 A27）。99.82% 之所以可能，恰恰因为 Reasonix 只绑定 DeepSeek 一个后端——「coupling to one backend is the feature, not a limitation」。cache 设计不再是「上层框架层面的经验优化」，而是「模型 API 设计、推理引擎实现、和框架 architecture 共享同一套 cache 契约」的系统级结果。

这里存在一个结构性对比。当上游 harness 与模型 API 设计不共享 cache 契约时，harness 层只能用自己的 prompt engineering 补偿 cache miss 造成的上下文管理问题——补丁叠补丁。当上游同时控制模型 API、推理引擎、和 harness 框架时，cache 契约可以从模型训练一路贯穿到终端用户 session 管理。这种跨层一致性对任何第三方 harness 都不可复制——第三方不控制推理引擎，也不定义模型 API 的 cache 行为。

Cache hit 不是性能优化问题，是 harness 架构的核心设计变量。如果 DeepSeek 在 harness 产品中把 cache hit 从「尽力而为的优化」变为「模型 API、推理引擎、harness 框架三方共享的 cache 契约」，其 agent 产品的单位经济学将构成对手无法追赶的护城河。不是因为 prompt 写得更好，而是因为 cache 的保证来自对三层基础设施的垂直整合，而非框架层的孤立优化。

（本章约 1150 字）

---

## 四、Coding agent 的商业可持续性与飞轮基础

coding 领域是当前唯一同时满足市场规模和商业可持续性的 agent 品类。不是巧合——它具备三个其他领域结构性缺位的条件（参见 A17）：可分解的中间表示（代码可局部编辑）、廉价的自动验证（编译器/测试套件）、错误可逆且代价低。

这三个条件组合成一个飞轮（参见 A15）：context 采集层（代码仓库天然提供结构化的 git history、依赖图、AST）→ 微调信号层（issue → PR 的天然 (input, output, ground_truth)三元组）→ 验证闭环层（测试套件 + type checker + compiler）。三层基础设施都有成熟方案，飞轮自转。其他领域（法律、医疗、金融）段段卡住：输入端的 context 散落在邮件链和口耳相传中，中间缺训练信号的三元组，输出端缺自动验证器。

这个不对称决定了 coding agent 的商业逻辑与其他 agent 品类完全不同。在验证器缺位的领域，prompt engineering 的繁荣本身就是症状——用户能操控的界面只剩 prompt 一个旋钮，prompt 模板市场是「用经验规则近似替代反馈回路」（参见 A18）。当模型迭代打破这些经验规则的拟合对象时，之前积累的「prompt 资产」归零。但在 coding 领域，验证器在场——编译器说你错了就是错了。coding harness 的一部分价值锚定在模型外部的确定性检验器上，不完全随模型行为漂移而贬值。

从这个结构性事实出发，coding agent 的产品策略有几个自然方向：

**飞轮转向。** 当用户的使用数据——代码 → 测试结果 → 修复轨迹 → 最终采纳/拒绝——被组织成可反馈的 (input, output, reward) 结构时，飞轮开始转。Harness 层不需要判断代码好不好，编译器/测试套件已经判了。Harness 层的工作是确保这些判断结果以结构化形式回流到模型训练管线中，而不是散落在用户各自的终端里。

**entropy 对冲。** agent 加速产出的同时也加速了代码库的熵增（参见 A16）。agent 每多跑一步，代码库就多一份被修改的概率。harness 层的工程价值不是「让 agent 跑更多步」，而是「在 agent 跑完每一步之后判断这一步是否让代码库整体变好了」。这个判断目前靠人工 review——如果未来靠自动化验证器（测试通过 + type check 通过 + lint 通过）作为 harness 侧的 gate，人的 review 带宽可以从「逐行检查」释放到「判断结构和设计」。

**渗透率预期的结构性偏差。** 当前市场定价隐含 AI 均匀渗透各行业的假设，实际渗透分布极度不均（参见 B28）。Coding 是唯一接近范式替换的领域。但范式替换不是用 AI 替代程序员——是重构「谁写代码、谁验代码、谁修代码」的工作流分工。在这个重构过程中，harness 层决定分工线的位置：哪些环节留在模型内部、哪些交给外部验证器、哪些保留给人做最终判断。

（本章约 1100 字）

---

## 五、Harness PM 的位置

Harness PM 的默认脚本是「搜集用户反馈 → 转化为功能需求 → 排优先级 → 交付 feature」。这个脚本对 coding agent 不适用。不是因为产品管理方法论有问题——是因为产品本身的特性（模型能力边界）每 8-12 周被重写一次。

第二脚本更隐蔽也更常见：「研究 Anthropic/OpenAI 的最新 release → 反向工程他们的 design decisions → 复制到自己的产品里」。这个脚本的合理性来自一个事实——上游厂商公开的 harness 设计确实暴露了模型当前能力边界的等高线（参见 A20）。但致命缺陷不在复制行为本身，在复制的是别人对自己模型的诊断结果（对你未必适用），且总是落后一拍的快照（对方已经在绘制下一个等高线）。

Harness PM 的正位不在上述两个脚本之中。它的工作是从最基础的一层逐步往上的：

**第一层：可观测性基础设施。** 弱模型不知道自己到了能力边界（参见 A21）。这是 calibration 的内部问题，不属于 harness 能解决的范畴。但 harness 层可以成为这个问题的探测系统——观测模型在哪些任务类型上频繁拒绝、哪些类型上沉默失败（做了但做错了、没有报错）、哪些类型上反复尝试仍失败。这些信号不告诉模型团队「怎么修」，但告诉他们「哪里需要修」。在此之上，harness 的非贬值价值——logging、tracing、failure mode taxonomy、audit trail（参见 externalization-ceiling, W5）——构成 harness 的「OS 层」，不随任何单一模型版本更新而失效。

**在第一层的基础上，第二层：止损机制。** Pro 在 agentic fetch 任务中表现比 Flash 更差，不是因为能力不足，是因为更高的推理预算让它在死胡同里「想得更多」而非「更早放弃」（参见 A25）。推理能力的增加不等于元认知判断的改善——知道什么时候该停是独立于推理深度的能力维度。harness 层在这里的工作不是优化模型的推理路径，是设计外部止损机制（超时、结果验证、多路径并行后择优）来补偿模型内部的元认知缺口。这和 discard-capability 论文的框架同构：当模型内部不善于做 discard 决策时，把 discard 上移至 harness 层。

**前两层之上，第三层：能力等高线的翻译。** 前两层产出的数据——哪些任务模型开始不再需要补丁了、哪些止损机制可以被拆除了、哪些失败模式是新出现的——是对模型能力边界的最精确测量。这些信息以代码的形式散落在系统提示的删减历史里、以 anecdote 的形式存在于研究团队的内部讨论里。Harness PM 的工作是把这些信号从代码和 anecdote 中提取出来，整理成可被产品决策引用的判断：当前这版模型在 X 类任务上已经不需要 harness 补偿了，在 Y 类任务上新出现了一种需要补偿的失败模式。

三层递进，收束于同一个位置：Harness PM 不是做补丁的人——虽然补丁是目前 harness 层最可见的产物。Harness PM 是参与模型能力等高线绘制的人。只是用的工具不是训练脚本，是产品设计和观测系统。

（本章约 1200 字）

---

*全文字数：约 5700 字*
*来源：externalization-ceiling 论文 + discard-capability 论文 + Category A 条目 31 条 + Reading/ 根目录 harness 复盘素材*
*引用标注格式：(参见 An) 为分类报告中的条目编号，最终稿需替换为正式引用*
