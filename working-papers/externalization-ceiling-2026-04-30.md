# Externalization Ceiling: Harness 层价值的折旧与迁移

> Working Paper · Draft · 2026-04-30  
> Rewritten: 2026-05-12; revised: 2026-05-20  
> Source notes: tech.md, power.md, mind.md; supplemented from archive/index.md

---

## 主张

当前 AI 工程中所有不修改模型权重的改进——prompt engineering、harness 框架、managed agents、alignment monitoring、tool routing、运行时治理——都属于 externalization：在权重之外重组模型的运行环境。但 externalization 不是一个同质资产。

更精确的结论是：**补丁型 harness 会随模型进步快速折旧，控制型、观测型、接口型、诊断型 harness 的价值会迁移而非消失。** 模型能力提升会吃掉那些专门补偿某一代模型缺陷的脚手架；但同时，模型越强、部署越广，cost control、observability、governance、evaluation、权限边界、审计轨迹、多模型接口这类元能力反而更重要。

因此，harness 层投资的关键不是“变厚还是变薄”，而是区分两种厚度：一种是围绕当前模型失败模式堆出来的脆弱厚度，另一种是沉淀为系统工程元能力的持久厚度。前者的折旧周期绑定模型发布周期；后者的寿命绑定组织、成本、安全和可靠性问题本身。

---

## 论据链

### E1. 统一归类：externalization 是权重外的运行时重组

> *来源：tech.md —— 外化的天花板*

当前 AI 工程所有改进（prompt / harness / managed agents / alignment monitoring）本质都是同一件事：externalization——在不改变模型权重的前提下重组外部运行时环境。天花板不来自工程努力不足，而来自模型底层表征的几何结构，后者由训练目标（有损压缩）雕刻，不为使用需求服务。

**论据功能**：建立分析对象。只要改进发生在权重之外，它就必须面对同一个问题：它能改变模型被调用的环境，但不能直接改变模型内部表征。

### E2. 补丁型 harness 绑定特定模型缺陷

> *来源：tech.md —— Harness 是临时补丁；Harness 是模型缺陷的考古层*

每个补丁型 harness 组件都在回答一个历史问题：“当时那一版模型做不到什么？” Sonnet 4.5 的 context reset 在 Opus 4.5 变成 dead code，Notion 为 Opus 4.6 写的 tool-failure fallback 在 4.7 变成多余。这类组件不是一般意义上的技能过时，而是其对象本身每轮都在被替换。

更精确地说，补丁型 harness 是对**特定模型、特定 effort level、特定任务分布下某个具体失败模式**的补偿。三个维度任一变化，都可能让补丁从有用变成无用，甚至变成干扰。

**论据功能**：把原稿“每个 harness 都会贬值”的过强命题收窄为“补丁型 harness 会贬值”。这是一条可观察、可追踪的折旧机制。

### E3. 生命周期由模型进步速度决定，不由 harness 工程质量决定

> *来源：tech.md —— Harness 的保质期是下一次模型更新；Harness 层的生命周期极短是结构性的*

Cat Wu 团队每次新模型发布就通读 system prompt 并移除多余指令，to-do list 追踪工具在模型能力上来后被废弃；本地 RAG pipeline 也在百万上下文出现后迅速从最佳实践变成阶段性方案。不是这些 harness 做错了，而是它们要解决的问题被底层模型更新消解了。

**论据功能**：说明折旧的外生性。补丁型 harness 的寿命不是由自身代码质量决定，而是由模型侧进步速度决定。

### E4. Easy/medium 任务掩盖 ceiling，hard case 暴露 ceiling

> *来源：tech.md —— 「85% 接近」的修辞结构*

中层模型借助框架在 85% 的任务上接近 Sonnet，真正的信息在剩下的 15%。框架能补的是流程编排、记忆持久化、tool routing 等结构性能力，把 easy 和 medium 拉平；剩下的 hard case 需要推理深度，是模型本身的事，框架碰不到。

**论据功能**：解释为什么 harness 在短期内看起来很强。externalization ceiling 在 easy tasks 上不可见，在 hard tasks 上暴露。

### E5. 变薄与变厚是位置函数，不是路线分歧

> *来源：tech.md —— Harness 变薄与变厚是位置函数，不是路线分歧；同一判断因位置不同得出相反策略*

Anthropic 说 harness 应该变薄，小米说 harness 应该变厚，两者都对但不对称。Anthropic 站在模型前沿，看到的是补丁型 harness 的递减价值；小米站在追赶期，看到的是 harness 的当下杠杆。下游厂商更应警惕自研复杂框架的诱惑：如果业务只是复刻头部开源 demo 改配置，所谓“Agent 开发”很可能只是把第三层配置工作包装成第二层架构工作。

**论据功能**：把策略分歧还原为位置差异。变厚可以是追赶期策略，但只有沉淀为可观测性、可靠性、成本控制、治理接口的厚度才值得长期持有。

### E6. 模型 commodity 化会先抬高外化价值，再压低补丁溢价

> *来源：power.md —— Harness 层的价值天花板与模型层的 commodity 化是同一枚硬币的两面*

当模型趋向 commodity，externalization（prompt/harness/orchestration）的价值会上升，因为差异化从模型层转向客户端和运行时。但这个上升不是无限的：模型能力继续内化后，补丁型 harness 的溢价会被压低。Anthropic 的赌注是在天花板到来之前，用 governance 控件和企业级粘性把护城河加固到足够深。

**论据功能**：修正原稿中的矛盾。harness 价值与模型 commodity 化不是简单负相关，而是阶段性关系：模型差异收窄时外化层升值；模型能力继续进步时补丁层折旧。

### E7. Harness 本身是独立变量，但不是持久壁垒

> *来源：tech.md —— Harness 本身是独立变量；只做引擎不做车的战略纯粹性*

同一模型在不同 harness 中表现差异可达 16 个百分点。context 管理、tool routing、权限模型会独立塑造模型行为；选择 harness 不只是选 UI，而是在选择一个运行环境。但这种差异并不自动形成持久壁垒：Agent 框架、工具链、SDK 生态最终会开源或被复刻，DeepSeek 甚至可以直接兼容 Anthropic API format。

**论据功能**：避免过度否定 harness。harness 的确影响结果，但影响结果不等于能长期捕获价值。

### E8. 工作流显性化的扩展路径是线性的

> *来源：tech.md —— 工作流显性化的线性扩展结构*

Prompt engineering 的真实上限不是写 prompt 的技巧，而是提取者对目标领域默会工作流的理解深度。每个领域需要单独做一次深度工作流提取，产出的系统提示不能迁移，提取质量取决于同时具备领域实践深度和形式化能力的人。这个交集极小，扩展路径因此是线性而非指数的。

**论据功能**：说明为什么“把专家流程写进 prompt”无法像模型训练一样形成通用飞轮。它有用，但扩展速度慢。

### E9. Coding 是反例，也是边界条件

> *来源：tech.md —— Coding 域飞轮的三层基础设施结构；Coding 飞轮的三层基础设施与其他领域的结构性缺位*

Coding 领域 harness 能形成飞轮，不是因为 harness 本身突破了 ceiling，而是因为该领域同时具备三层基础设施：context 采集层（结构化代码、依赖图、git history）、微调信号层（issue → PR 的 input/output/ground truth 三元组）、验证闭环层（测试、type checker、compiler）。其他领域常缺其中一层或多层：视频生成缺可局部编辑的中间表示和廉价验证；法律、医疗等领域的最终正确性锁在制度判断里。

**论据功能**：把 E8 的“线性扩展”改成有条件命题。满足三层基础设施的领域，harness 可以从补丁变成飞轮；不满足时，仍然停留在线性提取。

### E10. 有些 harness 不是补偿模型缺陷，而是补偿 agent 加速后的 entropy

> *来源：tech.md —— Agent 加速的 entropy 与工程纪律*

Agent 加速产出速度的同时也加速软件 entropy。对冲手段不是更多 agent，而是更好的工程纪律、可观测性、测试、review、成本控制和系统可靠性。这类 harness 不是补偿 base model 的某个缺陷，而是补偿 agent 加速后系统混乱累积速度超过人类管理阈值的问题。

**论据功能**：给出最重要的反例。模型越强、agent 越快，这类基础设施型 harness 的需求可能越强，因此其生命周期可能与模型进步正相关，而非负相关。

### E11. Harness 是负空间地图，也是缺陷考古层

> *来源：tech.md —— Harness 是模型能力的负空间地图；Harness 是模型缺陷的考古层*

上游厂商公开的工作流和产品设计选择，其价值不只在工作流本身，而在于暴露了厂商认为模型当前能可靠执行的边界。被拆掉的脚手架标记模型已经内化的能力；保留的脚手架标记仍需外部补偿的缺陷。harness 组件消失本身就是信息：它记录了模型能力等高线的移动。

**论据功能**：说明 harness 的诊断价值不随补丁折旧而消失。作为生产组件，它会过时；作为历史记录，它会升值。

### E12. Ceiling 不是固定墙，而是被训练目标、后训练、在线适应共同移动的边界

> *来源：tech.md —— 后训练算力持平是 ceiling 位置可移动的证据；Visual Primitive Tokenization 是部分突破外化天花板的案例；测试时训练（TTT）模糊了训练与部署的边界*

Post-training 正在把 agent 能力 bake into weights；visual primitive tokenization 通过新增 point/box token 改变表征空间，但 trigger word 依赖又把它部分拉回 prompt engineering；测试时训练如果成立，则会在部署期通过世界模型内的 RL 微调修改权重。三者说明 externalization ceiling 不是静态墙，而是一条会被训练阶段、表征设计和在线适应移动的边界。

**论据功能**：扩展原稿的二元划分。问题不再只是“权重内 vs 权重外”，还包括“离线训练 vs 在线适应”“表征改造 vs prompt 触发”“生产补丁 vs 诊断记录”。

### E13. Ceiling 轴与实效轴必须分离

> *来源：mind.md —— Ceiling 轴与实效轴的分离*

对范式内工程的判断需要分两条轴：ceiling 轴回答“这东西在技术史上能站多久”，实效轴回答“这东西现在能不能帮人做事”。混淆两者会导致两种错误：把当前 harness 工程当长期路线而过度押注，或因为“反正都是范式内优化”而拒绝参与真实工程。更稳的姿态是短期参与、长期不押宝：参与得进去，所以不脱离实际；不押宝，所以不被范式变更击碎。

**论据功能**：给出本文的实践立场。外化天花板不是反工程，而是反长期错配；短期实效和长期 ceiling 可以同时成立。

---

## 论证结构图

```
E1 externalization = 权重外运行时重组
 │
 ├── 补丁折旧轴
 │    ├── E2 补丁绑定特定模型/effort/任务分布
 │    ├── E3 生命周期由模型进步决定
 │    ├── E4 easy/medium 可补，hard case 暴露 ceiling
 │    └── E5 变厚是追赶期策略，变薄是前沿收敛方向
 │
 ├── 商业与壁垒轴
 │    ├── E6 commodity 化先抬高外化层，再压低补丁溢价
 │    └── E7 harness 影响表现，但不自动形成持久壁垒
 │
 ├── 扩展边界轴
 │    ├── E8 工作流显性化通常线性扩展
 │    └── E9 coding 因三层基础设施形成局部飞轮
 │
 ├── 持久价值轴
 │    ├── E10 observability / cost control / reliability 对冲 agent entropy
 │    └── E11 harness 历史是模型能力负空间地图
 │
 └── ceiling 移动轴
      └── E12 后训练、表征改造、在线适应移动边界

实践立场轴
 └── E13 ceiling 轴与实效轴分离

结论：
补丁型 harness 折旧；
基础设施型、诊断型、治理型 harness 迁移为持久资产；
策略不是简单变厚/变薄，而是识别哪一种厚度值得持有。
```

---

## 边界条件与待验证点

### W1. “表征几何”仍是解释性隐喻

E1 把天花板归因于模型底层表征几何，但这仍不是一个可直接测量的变量。当前更稳妥的写法是：经验上可观察到补丁型 harness 随模型更新折旧；“表征几何”解释了为什么这种折旧会系统性出现，但还不能直接量化 ceiling 位置。

**修订后处理**：保留为机制解释，不把它当作可测量事实。

### W2. Harness 分类需要更精细

“harness”至少包含五类：

- **补丁型**：context reset、tool-failure fallback、黑名单 prompt、复杂 prompt chain
- **控制型**：权限、KYC、访问门禁、policy enforcement
- **观测型**：logging、tracing、cost accounting、eval harness
- **接口型**：多模型路由、API 兼容层、tool schema、session/sandbox 虚拟化
- **诊断型**：记录哪些脚手架被拆除、哪些失败模式仍需补偿

**修订后处理**：主张只把“快速折旧”施加在补丁型 harness 上。

### W3. 变厚到变薄的切换判据仍缺失

追赶者在模型能力不足时必须变厚；前沿厂商在能力内化后自然变薄。真正缺的是切换判据：什么时候应该继续堆 harness，什么时候应该停止加杠杆，什么时候应该把经验沉淀为元能力？

**可能判据**：某个 harness 组件的维护频率高于收益增长；同类模型更新后组件需要重标定；组件只服务单一模型/单一任务分布；或组件无法转化为观测、评估、成本、可靠性资产。

### W4. Coding 反例需要严格限定

Coding 域可形成飞轮，容易被误读成“harness 也能指数扩展”。但 coding 的特殊性在于中间表示、验证器、错误可逆性、训练信号都极其成熟。其他领域若缺这些条件，不能直接外推。

**修订后处理**：把 coding 作为边界条件，而不是推翻主张的反例。

### W5. Online adaptation 可能改变整篇论文的轴

如果测试时训练、部署期 RL、世界模型内自我练习成为开放环境中的稳定机制，“不改权重的 externalization”与“改权重的训练”之间会出现连续谱。届时 externalization ceiling 不能再用二元轴描述，需要加入第三轴：离线训练 / 在线适应。

**修订后处理**：列为高优先级观察轴，但当前证据仍限于受控任务。

### W6. 可观测性本身也有上限

Harness 工程捕获的是输出行为，不是模型高维中间表示中的真实推理路径。当模型能力超过 harness 的理解能力，外部约束最终会退化为粗粒度隔离（例如物理隔离、权限隔离、访问门禁）。这说明基础设施型 harness 虽然比补丁型更持久，但也不是无限可扩展。

---

## 待补充

| # | 需要什么 | 为什么需要 | 预期来源 |
|---|---------|-----------|---------|
| 1 | 主要实验室 system prompt / harness 变化的版本追踪 | 验证补丁型 harness 的折旧周期是否绑定模型发布周期 | 厂商 changelog、公开 system prompt、产品 diff |
| 2 | 被移除 harness 组件与模型 benchmark / eval 进步的对照 | 验证“负空间地图”假说 | 模型发版历史 + harness changelog |
| 3 | 不同 harness 对同一模型表现的系统评测 | 量化 E7 中 harness 作为独立变量的影响 | OpenCode / Claude Code / Codex / 自研 harness 对照 |
| 4 | coding 域三层基础设施在其他领域的缺位清单 | 区分哪些领域能形成 harness 飞轮，哪些只能线性扩展 | 法律、医疗、视频生成、古籍校勘案例 |
| 5 | Agent 加速后的 entropy 指标 | 验证基础设施型 harness 是否随模型进步升值 | repo entropy、bug rate、review load、rollback frequency |
| 6 | 企业客户对补丁型 vs 治理型 harness 的支付意愿 | 区分技术补丁价值和制度基础设施价值 | 采购访谈、SaaS 合同结构、enterprise feature adoption |
| 7 | 在线适应 / TTT 在开放环境中的复现情况 | 判断 externalization ceiling 是否需要从二元轴扩展为三轴 | World model / robotics / agent RL 论文 |

---

## 可操作结论

1. 不要把“熟练使用某个 agent 框架”当作长期资产。它更像绑定某一代模型的方言。
2. 可以短期参与补丁型 harness，但不要长期押宝。参与是为了获得模型边界的一手感，而不是囤积会折旧的技巧。
3. 长期值得沉淀的是 observability、evaluation、cost control、reliability、权限边界、接口设计、workflow extraction 方法论。
4. 对厂商而言，harness 变厚要服务于数据、治理、企业粘性或成本结构；只服务于当前模型缺陷的厚度应尽快拆除。
5. 对个人而言，最佳策略是短期跟进、长期不押宝：用现成工具做真实工作，把经验抽象成跨模型、跨框架仍成立的元能力。

---

*生成：paper-mill Mode B, 2026-05-12 (rewrite of 2026-04-30 original)*  
*修订：Codex review pass, 2026-05-20*  
*来源：tech.md + power.md + mind.md；补充吸收 archive/index.md 中 2026-05-02 至 2026-05-20 的相关条目*
