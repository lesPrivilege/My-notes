# DeepSeek 做 Harness：一个结构性论证

> Working Paper · Draft · 2026-05-17
> 求职附件：DeepSeek Agent Harness 产品经理岗位
> 读者：DeepSeek 研究员与核心工程师

---

## 一、外化天花板：harness 工程的结构性约束

当前 AI 工程中所有不修改模型权重的改进——prompt engineering、harness 框架、managed agents、alignment monitoring——是同一件事：externalization。在不改变模型权重的前提下重组外部运行时环境。天花板不来自工程努力不足，来自模型底层表征的几何结构，后者由训练目标（有损压缩）雕刻，不为使用需求服务。

> 當前 AI 工程所有改進（prompt / harness / managed agents / alignment monitoring）本質都是同一件事：externalization——在不改變模型權重的前提下重組外部運行時環境。天花板不來自工程努力不足，而來自模型底層表徵的幾何結構，後者由訓練目標（有損壓縮）雕刻，不為使用需求服務。
>
> ——tech.md「外化的天花板」，2026-04-26

每个 harness 组件都是对特定模型版本、特定 effort level、特定任务分布下某个具体失败模式的补丁。Anthropic 为 Sonnet 4.5 的 context anxiety 设计的 reset，在 Opus 4.5 变成死代码。

> 每個 harness 組件都是對當前模型某個缺陷的補丁。Anthropic 為 Sonnet 4.5 的 context anxiety 設計的 reset，在 Opus 4.5 變成死代碼——證明 harness 的保質期就是下一次模型更新的時間。
>
> ——tech.md「Harness 是臨時補丁」，2026-04-26

Cat Wu 团队每次新模型发版后通读 system prompt 移除模型不再需要提醒的部分——这不是产品迭代，是脚手架拆除。harness 组件的生命周期不由工程质量决定，由模型进步速度决定。由此，所谓「prompt engineering 作为技能」的折旧周期与模型发布周期严格绑定；真正跨范式不折旧的是可观测性、cost control、系统可靠性。

> 每個 harness 組件都在回答一個歷史問題——「當時那一版模型做不到什麼？」。Sonnet 4.5 的 context reset 在 Opus 4.5 變成 dead code，Notion 為 Opus 4.6 寫的 tool-failure fallback 在 4.7 變成多餘。這不是「技能會過時」的普通現象，而是技能賴以存在的對象本身每輪都在被替換。更精確地說，harness 組件是對**特定模型、特定 effort level、特定任務分佈下某個具體失敗模式**的補丁——三個維度任一變化都可能讓補丁從有用變成無用甚至有害。
>
> ——tech.md「Harness 是模型缺陷的考古層」，2026-05-16

这个判断对下游三方完全成立：如果商业模式建立在模型缺陷的补丁上，每一次模型更新都是资产减记。

> 外部 harness 公司的整個能力空間是上游的子集，生態繁榮不是護城河，是廠商路線圖還沒走到那裡的窗口期。Anthropic 發布 full-stack app builder 的那一刻，外面所有 AI app builder 的商業模式被降維。
>
> ——tech.md「上游碾壓的不對稱性」，2026-04-26

但对上游——对 DeepSeek——这个判断需要修正。externalization ceiling 的贬值逻辑预设了「模型提供方」和「harness 提供方」是两个分离的实体，贬值发生在外部的资产表上。当两者是同一实体时，harness 投资与模型投资的关系从「零和」变为「内部对冲」。harness 被模型进步折旧的损失，同时是模型进步驱动的竞争力提升。外化天花板对下游是天花板，对上游是可调节的遮阳板。

上游做 harness 还有一个下游永远拿不到的收益：harness 的使用数据本身就是模型改进的结构化信号。每一条被执行或回滚的 tool call、每一个触发了用户显式干预的 agent 决策、每一次因模型能力不足而被 harness 层拦截的失败路径——对下游，这些数据是运营成本；对上游，是训练数据。Claude Code 的 harness 基础设施（占代码量 98.4%，VILA-Lab 逆向分析）虽然大部分是补丁，但补丁的分布本身就是信号：哪些被拆了、哪些还在、哪些是新加的，构成模型能力等高线的最细粒度绘制。

---

## 二、上游与下游的不对称：为什么 model 提供方做 harness 不同

> Anthropic 說 harness 應該變薄，小米說 harness 應該變厚，兩者都對但不對稱。Anthropic 站在模型前沿，看到的是 harness 的遞減價值；小米站在追趕期，看到的是 harness 的當下槓桿。關鍵在於：變厚策略有內在矛盾——你投入越多資源在框架上，模型進步越快讓這些投入貶值。變薄策略沒有這個矛盾，因此是收斂方向。
>
> ——tech.md「Harness 變薄與變厚是位置函數，不是路線分歧」，2026-04-26

DeepSeek 此前的立场——「只做引擎不做车」——在追赶期处于结构性正确的位置：

> DeepSeek 客戶端極簡（僅 OCR chatbot），不卷 agent 框架、不做複雜 memory 系統，資源集中在模型架構和推理效率上。Agent 框架、工具鏈、SDK 生態是 harness 層的東西，閉源廠商投入大量資源去建，但最終都會開源或被複刻——DeepSeek 甚至直接兼容 Anthropic API format，連等都不用等。Harness 層沒有持久壁壘，模型層的架構效率才有。
>
> ——tech.md「只做引擎不做車的戰略純粹性」，2026-04-28

但当 DeepSeek 在推理效率、KV cache 压缩、FP4 量化感知训练等维度进入领先位置后，「只做引擎不做车」从策略选择变成了自我设限。模型层优势——CSA/HCA 混合注意力将百万上下文 KV cache 压至上一代的 10%、FP4 量化感知训练让推理经济性接近十倍提升——在缺乏 harness 层的情况下只能间接变现（API 定价），不能直接建立开发者和终端用户的粘性。

上游做 harness 与下游做 harness 有四个结构性差异：

**第一，上游的 harness 是诊断工具，下游的 harness 是产品。**

> 上游廠商（Anthropic、Google）公開的工作流和產品設計選擇，其價值不在工作流本身的精巧，而在於它們暴露了廠商認為模型當前能可靠執行的邊界在哪裡。Cat Wu 團隊每次新模型發版後通讀 system prompt 移除模型不再需要提醒的部分，這個操作本身就是在持續繪製模型能力的等高線——被拆掉的腳手架標記了模型已經內化的能力，保留的腳手架標記了仍需外部補償的缺陷。
>
> ——tech.md「Harness 是模型能力的負空間地圖」，2026-04-28

这种诊断价值下游无法获得——他们只能读到系统 prompt 的最终版本，看不到内部演进历史，也不拥有对应模型的权重和训练环境来验证「这个补丁到底在补什么」。

**第二，上游的 Demo 即标准。**

> OpenAI 開源航司客服 Agent demo 的殺傷力不在技術複雜度，而在它是官方出的。參考實現一旦開源，下游方案商就從賣架構設計降級為賣定制配置。同時 SDK 綁定把開發者鎖進 OpenAI 生態——demo 是教育市場的外衣，生態鎖定才是目的。
>
> ——tech.md「頭部廠商的 Demo 即標準」，2026-04-26

DeepSeek 开源的 harness 参考实现会反向定义下游生态的接口规范——这个规范本身就是竞争壁垒的组成部件。

**第三，Prompt 效力沿架构边界集中到厂商一侧。**

> 同一份prompt文本，在個人私部署架構裡接近無用，在官方orchestration + 品牌系統grounding + 精細控件的架構裡產生斷層級效果。Claude Design（隨Opus 4.7推出）就是這個機制的產品化：Brilliant報告原先需20+ prompts的頁面現在2 prompts。不是prompt在貶值，是外化層的效力在沿架構邊界集中到廠商一側。
>
> ——tech.md「Prompt效力的重新分配」，2026-05-02

这个重新分配的方向是确定的：朝模型提供方集中。

**第四，上游碾压是全栈纵深。**

> 上游廠商的碾壓路徑不僅是模型能力，還包括標準定義權（Google Skills Repository + MCP 協議）、開發者工具鏈控制（ADK + agents-cli）、以及運行時基礎設施（Agent Runtime、fleet orchestration）。下游的生存空間不是被一個產品打掉的，而是被整個基礎設施棧逐層擠壓的。
>
> ——power.md「上游碾壓是全棧縱深，不是單點優勢」，2026-04-28

DeepSeek 如果不进入 harness 层，就是把工具链控制权、接口规范定义权、和开发者社区运营权让给 Anthropic 和 OpenAI——这三样恰好是 coding agent 这个最大且最可持续的 agent 品类的入场券。

DeepSeek 此时下场做 harness，不是背离「只做引擎不做车」，而是「做引擎的人也需要展示引擎能跑多快」。当模型层的竞争力已经需要通过 harness 层的体验来兑现时，「不做车」从战略清晰变成了战略缺失。

---

## 三、Cache hit 作为跨层设计原理

DeepSeek 在一个维度上拥有其他厂商难以复制的结构性优势：跨层一致的 cache 设计哲学。

> 「能離線解決的問題不要在線解決」是多層收斂的統一原理：從訓練（over-training 把高頻 pattern 燒進權重）、到架構（MoE 稀疏激活只調動相關 expert）、到推理（KV cache 命中避免重複計算 prefill）、到產品（system prompt caching 最大化前綴復用），每一層都在做同一件事——把重複的計算前置，運行時只處理真正的增量。這不是巧合，而是物理約束（memory bandwidth 是硬瓶頸）決定了所有抽象層的優化方向必然趨同。
>
> ——tech.md「能離線解決的問題不要在線解決」，2026-05-02

DeepSeek 在这个统一原理上的实施深度超出了任何单一点的优化：

**模型层。** CSA/HCA 混合注意力将百万上下文 KV cache 压至 V3.2 的 10%、FLOPs 压至 27%。FP4 量化感知训练不是在部署时量化——是在训练阶段就让模型适应 FP4 精度，推理时直接使用 FP4 权重。推理经济性的优势根植于训练阶段，不是事后权衡。

**推理层。** ds4（antirez 的 DwarfStar 4）将 KV cache 作为磁盘的一等公民——KV checkpoints 以 SHA1 为 key 写入 .kv 文件，exact 前缀跨 session 甚至跨服务器重启复用。这不是工程技巧，是架构决策：cache 的生命周期从「per session」拉长到了「per deployment」。DSML tool-call replay 进一步确保 agent 场景下 tool call 返回结果不破坏前缀一致性。

> ds4 的 KV cache 作為磁盤一等公民（KV checkpoints written to disk with SHA1 keyed .kv files, exact prefix reuse across session switches and server restarts）
>
> ——Reading/DwarfStar4-ds4.md，2026-05-17

**框架层。** Reasonix 的 cache-first loop 设计——三段式 context 分区（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH）——将单日 cache hit rate 推至 99.82%，435M input tokens 的成本从 $61 压缩到 $12。

> Reasonix 的三段式 context 分段（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH），99.82% cache hit rate（單日 435M input tokens，~$12 vs ~$61）。「Coupling to one backend is the feature, not a limitation.」
>
> ——Reading/DeepSeek-Reasonix.md，2026-05-17

99.82% 之所以可能，恰恰因为 Reasonix 只绑定 DeepSeek 一个后端。cache 设计不再是「上层框架层面的经验优化」，而是「模型 API 设计、推理引擎实现、和框架 architecture 共享同一套 cache 契约」的系统级结果。

这里存在一个结构性对比。当上游 harness 与模型 API 设计不共享 cache 契约时，harness 层只能用自己的 prompt engineering 补偿 cache miss 造成的上下文管理问题——补丁叠补丁。当上游同时控制模型 API、推理引擎、和 harness 框架时，cache 契约可以从模型训练一路贯穿到终端用户 session 管理。这种跨层一致性对任何第三方 harness 都不可复制——第三方不控制推理引擎，也不定义模型 API 的 cache 行为。

Cache hit 不是性能优化问题，是 harness 架构的核心设计变量。如果 DeepSeek 在 harness 产品中把 cache hit 从「尽力而为的优化」变为「模型 API、推理引擎、harness 框架三方共享的 cache 契约」，其 agent 产品的单位经济学将构成对手无法追赶的护城河。不是因为 prompt 写得更好，是因为 cache 的保证来自对三层基础设施的垂直整合，而非框架层的孤立优化。

---

## 四、Coding agent 的商业可持续性与飞轮基础

coding 领域是当前唯一同时满足市场规模和商业可持续性的 agent 品类。不是巧合——它具备三个其他领域结构性缺位的条件：

> Coding 領域 harness 成熟的根本原因不是「發展更快」，而是三個前提條件同時滿足：可分解的中間表示（代碼本身可局部編輯）、廉價的自動驗證（編譯器/測試套件）、錯誤可逆且代價低。視頻生成缺前兩者，法律文書有第一個但缺後兩個。三條件是必要不充分的——古籍校勘三條件皆具，卻因跨域人才極少、數據組織範式與 LLM 訓練格式不兼容、OCR 前處理瓶頸而停滯。
>
> ——tech.md「Coding 飛輪的三層基礎設施與其他領域的結構性缺位」，2026-05-16

这三个条件组合成一个飞轮：

> coding 領域 harness 加速的飛輪由三層現成基礎設施構成——context 採集層（agentmemory 等 persistent memory），微調信號層（issue→PR 的天然 (input, output, ground_truth) 三元組），驗證閉環層（測試套件 + type checker + compiler）。每層都有成熟方案，組合起來飛輪自轉。
>
> ——tech.md「Coding域飛輪的三層基礎設施結構」，2026-05-13

其他领域（法律、医疗、金融）段段卡住：输入端的 context 散落在邮件链和口耳相传中，中间缺训练信号的三元组，输出端缺自动验证器。

这个不对称决定了 coding agent 的商业逻辑与其他 agent 品类完全不同。在验证器缺位的领域，prompt engineering 的繁荣本身就是症状——用户能操控的界面只剩 prompt 一个旋钮。

> 在缺乏自動驗證器、結構化中間表示和可採集 context 的領域，用戶能操控的界面只剩 prompt 一個旋鈕。prompt 模板市場、資深行業人士教你寫提示詞、威脅/PUA「提高回答質量」——本質上是在驗證器缺位條件下，用經驗規則近似替代反饋回路。這些「技巧」拟合的是當前模型版本的特定行為，模型一更新就失效。
>
> ——tech.md「Prompt engineering 的繁榮是驗證器缺位的症狀」，2026-05-16

但在 coding 领域，验证器在场——编译器说你错了就是错了。coding harness 的一部分价值锚定在模型外部的确定性检验器上，不完全随模型行为漂移而贬值。

从这个结构性事实出发，coding agent 的产品策略有几个自然方向：

**飞轮转向。** 当用户的使用数据——代码 → 测试结果 → 修复轨迹 → 最终采纳/拒绝——被组织成可反馈的 (input, output, reward) 结构时，飞轮开始转。Harness 层不需要判断代码好不好，编译器/测试套件已经判了。Harness 层的工作是确保这些判断结果以结构化形式回流到模型训练管线中，而不是散落在用户各自的终端里。

**entropy 对冲。** agent 加速产出的同时也加速了代码库的熵增。agent 每多跑一步，代码库就多一份被修改的概率。

> agent 加速產出速度的同時也加速了軟體 entropy（混亂速度）。對沖手段是更好的工程紀律，而非更多的 agent。這為 harness 貶值論提供了一個反例方向——有些 harness 不是補償 base model 的缺陷，而是補償 agent 加速帶來的新問題（混亂累積速度超過人類可管理閾值）。
>
> ——tech.md「Agent加速的entropy與工程紀律」，2026-05-13

harness 层的工程价值不是「让 agent 跑更多步」，而是「在 agent 跑完每一步之后判断这一步是否让代码库整体变好了」。

**渗透率预期的结构性偏差。** 当前市场定价隐含 AI 均匀渗透各行业的假设，实际渗透分布极度不均。Coding 是唯一接近范式替换的领域。

> 當前 AI 行業的泡沫性質不是一般意義的估值泡沫或敘事泡沫，而是渗透率預期的錯配——市場定價隱含 AI 均勻渗透各行業的假設，但實際渗透分布極度不均。Coding 接近範式替換，其他領域停留在「高級搜索引擎」階段。
>
> ——power.md「滲透率預期錯配」，2026-05-16

但范式替换不是用 AI 替代程序员——是重构「谁写代码、谁验代码、谁修代码」的工作流分工。在这个重构过程中，harness 层决定分工线的位置：哪些环节留在模型内部、哪些交给外部验证器、哪些保留给人做最终判断。

---

## 五、Harness PM 的位置

Harness PM 的默认脚本是「搜集用户反馈 → 转化为功能需求 → 排优先级 → 交付 feature」。这个脚本对 coding agent 不适用。不是因为产品管理方法论有问题——是因为产品本身的特性（模型能力边界）每 8-12 周被重写一次。

第二脚本更隐蔽也更常见：「研究 Anthropic/OpenAI 的最新 release → 反向工程他们的 design decisions → 复制到自己的产品里」。这个脚本的合理性来自一个事实——上游厂商公开的 harness 设计确实暴露了模型当前能力边界的等高线（同第二章引用的负空间地图）。但致命缺陷不在复制行为本身，在复制的是别人对自己模型的诊断结果（对你未必适用），且总是落后一拍的快照（对方已经在绘制下一个等高线）。

Harness PM 的正位不在上述两个脚本之中。它的工作是从最基础的一层逐步往上的：

**第一层：可观测性基础设施。**

> 弱模型不知道自己到了能力邊界。這不是 harness 能解決的問題，而是模型 calibration 的內部問題，屬於底層表徵幾何結構的範疇。
>
> ——tech.md「弱模型不知道自己到了能力邊界」，2026-04-28

harness 层可以成为这个问题的探测系统——观测模型在哪些任务类型上频繁拒绝、哪些类型上沉默失败（做了但做错了、没有报错）、哪些类型上反复尝试仍失败。这些信号不告诉模型团队「怎么修」，但告诉他们「哪里需要修」。

在此之上，harness 的非贬值价值构成 harness 的「OS 层」——不随任何单一模型版本更新而失效：

> Harness 的非貶值價值被低估：合規與治理（audit trail、policy enforcement）——制度需求，不因模型變強而消失；可觀測性（logging、tracing）——運行時基礎設施的需要獨立於模型能力；多模型編排——當底層模型頻繁切換時，統一接口層的價值反而上升；診斷記錄——哪個 harness 被拆了的歷史本身就是信號。
>
> ——externalization-ceiling W5「Harness 的非貶值價值」，2026-04-30

**第二层：止损机制。**

> Pro 在 agentic fetch 任務中表現比 Flash 更差，不是因為能力不足，而是更高的推理預算讓它在死胡同裡「想得更多」而非「更早放棄」。推理能力的增加不等於元認知判斷的改善——知道什麼時候該停是一個獨立於推理深度的能力維度。
>
> ——power.md「推理預算與止損判斷的反直覺關係」，2026-04-28

harness 层在这里的工作不是优化模型的推理路径，是设计外部止损机制——超时、结果验证、多路径并行后择优——来补偿模型内部的元认知缺口。这和 discard-capability 论文的框架同构：当模型内部不善于做 discard 决策时，把 discard 上移至 harness 层。

> 智能系統的核心競爭力正從「模型能做什麼」轉向「模型決定不做什麼」。在所有抽象層——架構設計、tokenization、attention、FFN 激活、緩存策略、推理調度——信息篩選機制正在取代信息處理能力，成為制約和定義系統上界的統一變量。
>
> ——discard-capability 主張，2026-05-11

**第三层：能力等高线的翻译。** 前两层产出的数据——哪些任务模型开始不再需要补丁了、哪些止损机制可以被拆除了、哪些失败模式是新出现的——是对模型能力边界的最精确测量。这些信息以代码的形式散落在系统提示的删减历史里、以 anecdote 的形式存在于研究团队的内部讨论里。Harness PM 的工作是把这些信号从代码和 anecdote 中提取出来，整理成可被产品决策引用的判断：当前这版模型在 X 类任务上已经不需要 harness 补偿了，在 Y 类任务上新出现了一种需要补偿的失败模式。

三层递进，收束于同一个位置：Harness PM 不是做补丁的人——虽然补丁是目前 harness 层最可见的产物。Harness PM 是参与模型能力等高线绘制的人。只是用的工具不是训练脚本，是产品设计和观测系统。

---

*来源：archive/tech.md（外化的天花板、Harness 是臨時補丁、Harness 是模型缺陷的考古層、上游碾壓的不對稱性、Harness 變薄與變厚是位置函數、只做引擎不做車的戰略純粹性、Harness 是模型能力的負空間地圖、頭部廠商的 Demo 即標準、Prompt效力的重新分配、能離線解決的問題不要在線解決、弱模型不知道自己到了能力邊界、Coding域飛輪的三層基礎設施結構、Coding 飛輪的三層基礎設施與其他領域的結構性缺位、Prompt engineering 的繁榮是驗證器缺位的症狀、Agent加速的entropy與工程紀律）；archive/power.md（上游碾壓是全棧縱深、推理預算與止損判斷的反直覺關係、滲透率預期錯配）；working-papers/externalization-ceiling（E1 主張、W5 非貶值價值）；working-papers/discard-capability（E1 主張）；Reading/DeepSeek-Reasonix.md；Reading/DwarfStar4-ds4.md*
