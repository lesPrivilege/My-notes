# DeepSeek 做 Harness：一个结构性论证

> Working Paper · Draft · 2026-05-17
> 求职附件：DeepSeek Agent Harness 产品经理岗位
> 读者：DeepSeek 研究员与核心工程师

---

我注意到一个现象：每次 Anthropic 或 OpenAI 发布新模型，总有一些 harness 组件直接报废。不是「需要调整」，是「变成死代码」。

Sonnet 4.5 时期，Anthropic 在 system prompt 里加了一个专门处理 context anxiety 的 reset 机制。Opus 4.5 一发布，这段逻辑直接作废——模型在那个维度上已经不需要外部补偿了。Cat Wu 的团队有一个固定操作：每次新模型发版，通读一遍 system prompt，把模型不再需要提醒的部分删掉。

> 每個 harness 組件都是對當前模型某個缺陷的補丁。Anthropic 為 Sonnet 4.5 的 context anxiety 設計的 reset，在 Opus 4.5 變成死代碼——證明 harness 的保質期就是下一次模型更新的時間。
>
> ——tech.md「Harness 是臨時補丁」，2026-04-26

Notion 为 Opus 4.6 写的 tool-failure fallback，到 4.7 也成了多余代码。

> 每個 harness 組件都在回答一個歷史問題——「當時那一版模型做不到什麼？」。Sonnet 4.5 的 context reset 在 Opus 4.5 變成 dead code，Notion 為 Opus 4.6 寫的 tool-failure fallback 在 4.7 變成多餘。這不是「技能會過時」的普通現象，而是技能賴以存在的對象本身每輪都在被替換。
>
> ——tech.md「Harness 是模型缺陷的考古層」，2026-05-16

这件事让我开始怀疑一个问题：如果 harness 组件的生命周期不由工程质量决定，而由模型进步速度决定，那整个 harness 工程的投入，本质上是在给一个会缩水的资产加杠杆。所谓「prompt engineering 作为技能」——折旧周期和模型发布周期严格绑定。真正跨范式不折旧的东西，是可观测性、cost control、系统可靠性这类元能力——把它们包装成「AI 时代新技能」，说实话，是修辞通胀。

如果这个判断是对的，那它对外部 harness 公司意味着什么？意味着如果商业模式建立在模型缺陷的补丁上，每一次模型更新就是一次资产减记。

> 外部 harness 公司的整個能力空間是上游的子集，生態繁榮不是護城河，是廠商路線圖還沒走到那裡的窗口期。Anthropic 發布 full-stack app builder 的那一刻，外面所有 AI app builder 的商業模式被降維。
>
> ——tech.md「上游碾壓的不對稱性」，2026-04-26

但对 DeepSeek，情况不太一样。

上面的贬值逻辑，基础假设是「模型提供方」和「harness 提供方」是两个分离的实体。贬值发生在后者的资产表上。当两者是同一个实体时，harness 投资和模型投资的关系不是零和的——harness 被模型折旧的损失，同时也是模型进步带来的竞争力提升。

而且上游做 harness，有一笔下游永远拿不到的收益：harness 的使用数据本身就是模型改进的结构化信号。tool call 执行后被用户回滚、agent 决策触发了用户显式干预、模型能力不足被 harness 层拦截——对下游，这些是运营成本。对上游，是训练数据。Claude Code 的 harness 基础设施占了代码量的 98.4%（VILA-Lab 逆向分析），虽然大部分是补丁，但补丁的分布本身就是信号——哪些被拆了、哪些还在、哪些是新加的。

所以我的判断是：外化天花板对下游是天花板，对上游是可调节的遮阳板。

---

DeepSeek 之前的立场是「只做引擎不做车」。这个立场在追赶期是对的。资源集中在模型架构和推理效率上，agent 框架、工具链、SDK 生态这些东西，闭源厂商投了大量资源去建，但最终都会开源或被复刻——DeepSeek 甚至直接兼容 Anthropic API format。

> DeepSeek 客戶端極簡（僅 OCR chatbot），不卷 agent 框架、不做複雜 memory 系統，資源集中在模型架構和推理效率上。Agent 框架、工具鏈、SDK 生態是 harness 層的東西，閉源廠商投入大量資源去建，但最終都會開源或被複刻——DeepSeek 甚至直接兼容 Anthropic API format，連等都不用等。Harness 層沒有持久壁壘，模型層的架構效率才有。
>
> ——tech.md「只做引擎不做車的戰略純粹性」，2026-04-28

但情况变了。CSA/HCA 混合注意力把百万上下文的 KV cache 压到了上一代的 10%。FP4 量化感知训练让推理经济性接近十倍提升。当你在这些维度进入领先位置后，模型层的优势在缺乏 harness 层的情况下只能通过 API 定价间接变现——不能直接建立开发者和终端用户的粘性。「不做车」从策略清晰变成了策略缺失。

这里有一个我一直觉得值得讨论的点。Anthropic 说 harness 应该变薄，小米说 harness 应该变厚——两边都对，但不对称。

> Anthropic 說 harness 應該變薄，小米說 harness 應該變厚，兩者都對但不對稱。Anthropic 站在模型前沿，看到的是 harness 的遞減價值；小米站在追趕期，看到的是 harness 的當下槓桿。關鍵在於：變厚策略有內在矛盾——你投入越多資源在框架上，模型進步越快讓這些投入貶值。變薄策略沒有這個矛盾，因此是收斂方向。
>
> ——tech.md「Harness 變薄與變厚是位置函數，不是路線分歧」，2026-04-26

DeepSeek 目前的位置有意思——在模型能力上已经不是纯粹追赶者，但在 harness 上几乎还没开始。这意味着有机会走一条和 Anthropic 不完全一样的路。

我后来意识到，「上游做 harness」和「下游做 harness」不是程度差异，至少有几个地方的差异是结构性的。

第一个。Anthropic 公开的系统提示和工作流设计，真正值钱的不是它写得有多好，而是它暴露了厂商认为模型当前能可靠执行的边界在哪儿。Cat Wu 团队拆掉的脚手架标记了模型已内化的能力，保留的标记了仍需补偿的缺陷。这种诊断价值下游拿不到——下游只能读到公开版本，看不到内部演进历史，也不拥有模型权重和训练环境来验证「这个补丁到底在补什么」。

> 上游廠商（Anthropic、Google）公開的工作流和產品設計選擇，其價值不在工作流本身的精巧，而在於它們暴露了廠商認為模型當前能可靠執行的邊界在哪裡。Cat Wu 團隊每次新模型發版後通讀 system prompt 移除模型不再需要提醒的部分，這個操作本身就是在持續繪製模型能力的等高線——被拆掉的腳手架標記了模型已經內化的能力，保留的腳手架標記了仍需外部補償的缺陷。
>
> ——tech.md「Harness 是模型能力的負空間地圖」，2026-04-28

第二个。OpenAI 上次开源那个航司客服 Agent demo，杀伤力不在技术复杂度——它就是官方出的。参考实现一开源，下游方案商直接从卖架构设计降级成卖定制配置。SDK 绑定把开发者锁进 OpenAI 生态。Demo 是教育市场的外衣，生态锁定才是目的。

> OpenAI 開源航司客服 Agent demo 的殺傷力不在技術複雜度，而在它是官方出的。參考實現一旦開源，下游方案商就從賣架構設計降級為賣定制配置。同時 SDK 綁定把開發者鎖進 OpenAI 生態——demo 是教育市場的外衣，生態鎖定才是目的。
>
> ——tech.md「頭部廠商的 Demo 即標準」，2026-04-26

第三个比较微妙。同一段 prompt，在个人私部署架构里几乎没用——用户要自己搭环境、配权限、管 cache。但放到官方 orchestration + grounding + 控件的架构里，效果差断层级。Claude Design 随 Opus 4.7 推出后，之前 Brilliant 报告要 20+ prompts 的页面，现在 2 prompts 搞定。不是 prompt 贬值了，是外化层的效力在重新分配——而且方向是确定的：往模型提供方集中。

> 同一份prompt文本，在個人私部署架構裡接近無用，在官方orchestration + 品牌系統grounding + 精細控件的架構裡產生斷層級效果。Claude Design（隨Opus 4.7推出）就是這個機制的產品化：Brilliant報告原先需20+ prompts的頁面現在2 prompts。不是prompt在貶值，是外化層的效力在沿架構邊界集中到廠商一側。
>
> ——tech.md「Prompt效力的重新分配」，2026-05-02

第四个最直接。上游碾压不是靠单一产品——是靠标准定义权、开发者工具链控制、运行时基础设施，一整套体系逐层挤压下游。DeepSeek 如果不进入 harness 层，就是把这些东西拱手让给 Anthropic 和 OpenAI。

> 上游廠商的碾壓路徑不僅是模型能力，還包括標準定義權（Google Skills Repository + MCP 協議）、開發者工具鏈控制（ADK + agents-cli）、以及運行時基礎設施（Agent Runtime、fleet orchestration）。下游的生存空間不是被一個產品打掉的，而是被整個基礎設施棧逐層擠壓的。
>
> ——power.md「上游碾壓是全棧縱深，不是單點優勢」，2026-04-28

说到底，DeepSeek 现在下场做 harness，不是背离「只做引擎不做车」，是「做引擎的人也需要展示引擎能跑多快」。

---

聊一个具体的技术维度：cache。

我后来越来越觉得，cache 这件事不是单纯的省钱。很多 agent 架构最后能不能成立，核心就卡在 cache hit 上。

有一个观察我觉得很有意思，但说出来其实很简单：从 over-training 把高频 pattern 烧进权重，到 MoE 稀疏激活只调动相关 expert，到 KV cache 命中避免重复 prefill，到 system prompt caching 最大化前缀复用——整整四个抽象层，在做同一件事：把重复计算前置，运行时只处理增量。不是巧合。memory bandwidth 是硬瓶颈，所有层的优化方向被它统一了。

> 「能離線解決的問題不要在線解決」是多層收斂的統一原理：從訓練（over-training 把高頻 pattern 燒進權重）、到架構（MoE 稀疏激活只調動相關 expert）、到推理（KV cache 命中避免重複計算 prefill）、到產品（system prompt caching 最大化前綴復用），每一層都在做同一件事——把重複的計算前置，運行時只處理真正的增量。這不是巧合，而是物理約束（memory bandwidth 是硬瓶頸）決定了所有抽象層的優化方向必然趨同。
>
> ——tech.md「能離線解決的問題不要在線解決」，2026-05-02

DeepSeek 在这个统一原理上，有几个层的实施深度是别的地方很难直接复制的。

模型层。CSA/HCA 混合注意力，百万上下文 KV cache 压到 V3.2 的 10%，FLOPs 压到 27%。FP4 量化感知训练不是在部署时量化——训练阶段就让模型适应 FP4，推理时直接使用 FP4 权重。

推理层。ds4（antirez）这件事有点意思。他把 KV cache 当作磁盘的一等公民——KV checkpoint 以 SHA1 为 key 写入 .kv 文件，exact 前缀跨 session 甚至跨服务器重启复用。这不止是工程优化，是一个架构决策：cache 的生命周期从「per session」拉长到了「per deployment」。DSML tool-call replay 确保 agent 场景下 tool call 返回结果不破坏前缀一致性。

框架层。Reasonix 搞了一个 cache-first loop 设计——三段式 context 分区：IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH。单日 cache hit rate 99.82%，435M input tokens 从 $61 压缩到 $12。它为什么能做到 99.82%？因为它只绑定 DeepSeek 一个后端——「coupling to one backend is the feature, not a limitation」。

> Reasonix 的三段式 context 分段（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH），99.82% cache hit rate（單日 435M input tokens，~$12 vs ~$61）。「Coupling to one backend is the feature, not a limitation.」
>
> ——Reading/DeepSeek-Reasonix.md，2026-05-17

这三个层放在一起看，出现了一个很有意思的局面：cache 优化不再只是「框架层想办法少浪费 token」——它从模型 API 设计、到推理引擎实现、到框架 architecture，共享同一套 cache 契约。

这件事第三方做不了。第三方不控制推理引擎，也不定义 API 的 cache 行为。它最多在框架层做 prompt engineering 来补偿 cache miss——本质上是用补丁补偿另一层补丁。

如果 DeepSeek 在 harness 产品里，把 cache hit 从「尽力而为的优化」变成「模型 API、推理引擎、harness 框架三方的共享契约」，agent 产品的单位经济学就有了一个对手没法追的护城河。不是 prompt 写得好，是 cache 保证来自对三层基础设施的垂直整合。

---

说回 coding agent。为什么进展远快于法律/医疗？

一个很直接的原因：coding 有 compiler。你写错了，马上知道。法律没有。

展开一点说，coding 同时满足三个条件：可分解的中间表示（代码可局部编辑）、廉价的自动验证（编译器 / 测试套件）、错误可逆且代价低。这三个条件放在一起是一个飞轮——context 采集层（git history、AST、依赖图）+ 微调信号层（issue → PR 的天然三元组）+ 验证闭环层（测试套件 + type checker）——每一层都有现成的成熟方案，飞轮能自己转。

> Coding 領域 harness 成熟的根本原因不是「發展更快」，而是三個前提條件同時滿足：可分解的中間表示（代碼本身可局部編輯）、廉價的自動驗證（編譯器/測試套件）、錯誤可逆且代價低。視頻生成缺前兩者，法律文書有第一個但缺後兩個。
>
> ——tech.md「Coding 飛輪的三層基礎設施與其他領域的結構性缺位」，2026-05-16

> coding 領域 harness 加速的飛輪由三層現成基礎設施構成——context 採集層（agentmemory 等 persistent memory），微調信號層（issue→PR 的天然 (input, output, ground_truth) 三元組），驗證閉環層（測試套件 + type checker + compiler）。每層都有成熟方案，組合起來飛輪自轉。
>
> ——tech.md「Coding域飛輪的三層基礎設施結構」，2026-05-13

其他领域呢？法律、医疗、金融——段段卡住。输入端 context 散落在邮件链和口头传达里，中间缺训练信号的三元组，输出端缺自动验证器。

在验证器缺位的领域，会发生一件事：用户能碰的界面只剩 prompt 这一个旋钮。prompt 模板市场、各种「教你写出高质量提示词」的教程——本质是用经验规则近似替代反馈回路。问题是，这些规则拟合的是当前模型版本的行为，模型一更新就全废了。

> 在缺乏自動驗證器、結構化中間表示和可採集 context 的領域，用戶能操控的界面只剩 prompt 一個旋鈕。prompt 模板市場、資深行業人士教你寫提示詞、威脅/PUA「提高回答質量」——本質上是在驗證器缺位條件下，用經驗規則近似替代反饋回路。這些「技巧」拟合的是當前模型版本的特定行為，模型一更新就失效。
>
> ——tech.md「Prompt engineering 的繁榮是驗證器缺位的症狀」，2026-05-16

coding 不一样。编译器说你错了就是错了。coding harness 的一部分价值锚定在模型外部的确定性检验器上，不完全随模型行为漂移。这意味着 coding 是当前唯一一个同时有市场规模和商业可持续性的 agent 品类。

但这里有个很多人没注意到的问题。agent 加速产出的同时，也在加速代码库的熵增。agent 每多跑一步，代码库就多一份被修改的概率。原来的代码是人在控制节奏，现在 agent 的输出速度和人的 review 能力之间出现了速度差。

> agent 加速產出速度的同時也加速了軟體 entropy（混亂速度）。對沖手段是更好的工程紀律，而非更多的 agent。
>
> ——tech.md「Agent加速的entropy與工程紀律」，2026-05-13

所以 harness 的价值不是「让 agent 跑更多步」，是「在每一步之后判断这一步是否让代码库整体变好了」。如果自动化验证器（测试通过 + type check + lint）能在 harness 侧做 gate，人的 review 带宽就能从逐行检查释放出来。

还有一个跟市场叙事有关的点。现在市场的定价，隐含的假设是 AI 会均匀渗透各行业。实际情况不是这样。coding 接近范式替换了，大部分其他领域还停在「高级搜索引擎」。这个预期错配本身比估值数字更值得关注。

> 當前 AI 行業的泡沫性質不是一般意義的估值泡沫或敘事泡沫，而是渗透率預期的錯配——市場定價隱含 AI 均勻渗透各行業的假設，但實際渗透分布極度不均。Coding 接近範式替換，其他領域停留在「高級搜索引擎」階段。
>
> ——power.md「滲透率預期錯配」，2026-05-16

---

最后想聊聊 Harness PM 这个角色本身。

先说两件 Harness PM 经常做、但对 coding agent 不太适用的事。

第一件：「搜集用户反馈 → 转化为功能需求 → 排优先级 → 交付 feature」。这个脚本的问题不是方法论不对——是产品本身的特性每 8 到 12 周被重写一次。模型能力边界一移动，上季度的 roadmap 可能直接作废。

第二件更隐蔽：「研究 Anthropic/OpenAI 最新 release → 反向工程 design decisions → 复制到自己的产品」。有些合理性——上游的 harness 设计确实暴露了模型能力边界的等高线。但复制的是别人对自己模型的诊断结果，不一定适用你的模型。而且永远是落后一拍的快照。

那 Harness PM 到底在做什么？

我后来发现，很多 harness 工作，其实不是在「优化 agent」，而是在帮研究团队看清模型到底哪里不行。

弱模型自己不知道到了能力边界——不是 harness 能解决的问题，是模型 calibration 的内部问题。但 harness 层能成为一个探测系统：观测模型在哪些任务上频繁拒绝、哪些沉默失败（做了但没报错）、哪些反复尝试仍失败。这些信号不告诉研究团队「怎么修」，但能告诉「哪里需要修」。

> 弱模型不知道自己到了能力邊界。這不是 harness 能解決的問題，而是模型 calibration 的內部問題，屬於底層表徵幾何結構的範疇。
>
> ——tech.md「弱模型不知道自己到了能力邊界」，2026-04-28

这里有一个容易被忽略的事：harness 有一些维度的价值，不会随模型版本更新而消失。合规与治理（audit trail）、可观测性（logging、tracing、failure mode taxonomy）、多模型编排（底层模型频繁切换时统一接口的价值反而上升）、以及诊断记录——哪些 harness 被拆了、哪些还在。这些是 harness 的「OS 层」，不依赖特定模型行为。

> Harness 的非貶值價值被低估：合規與治理（audit trail、policy enforcement）——制度需求，不因模型變強而消失；可觀測性（logging、tracing）——運行時基礎設施的需要獨立於模型能力；多模型編排——當底層模型頻繁切換時，統一接口層的價值反而上升；診斷記錄——哪個 harness 被拆了的歷史本身就是信號。
>
> ——externalization-ceiling W5「Harness 的非貶值價值」，2026-04-30

再往上一层，还有个挺反直觉的发现。Pro 在 agentic fetch 任务里表现比 Flash 更差。不是因为 Pro 能力不如 Flash，是 Pro 的推理预算更高——在死胡同里「再想一步」而不是「早点放弃」。推理能力变强，元认知判断没变强。知道什么时候该停，是一个独立的维度。

> Pro 在 agentic fetch 任務中表現比 Flash 更差，不是因為能力不足，而是更高的推理預算讓它在死胡同裡「想得更多」而非「更早放棄」。推理能力的增加不等於元認知判斷的改善——知道什麼時候該停是一個獨立於推理深度的能力維度。
>
> ——power.md「推理預算與止損判斷的反直覺關係」，2026-04-28

这意味着 harness 层不能信任模型自己判断「什么时候该停」。需要设计外部止损机制——超时、结果验证、多路径并行后择优。这和另一个观察同构：智能系统的核心竞争力正在从「模型能做什么」转向「模型决定不做什么」。当模型内部不擅长 discard，就把 discard 决策上移到 harness 层来做。

> 智能系統的核心競爭力正從「模型能做什麼」轉向「模型決定不做什麼」。在所有抽象層——架構設計、tokenization、attention、FFN 激活、緩存策略、推理調度——信息篩選機制正在取代信息處理能力，成為制約和定義系統上界的統一變量。
>
> ——discard-capability 主張，2026-05-11

这三层串起来——可观测性基础设施、止损机制、还有把前两层的输出翻译成对模型能力边界的判断——差不多就是我认为 Harness PM 真正在做的事。

不是做补丁。虽然补丁是目前 harness 层最可见的产物。

更像是在参与绘制模型能力的等高线——用的工具不是训练脚本，是产品设计和观测系统。

---

*来源：archive/tech.md（外化的天花板、Harness 是臨時補丁、Harness 是模型缺陷的考古層、上游碾壓的不對稱性、Harness 變薄與變厚是位置函數、只做引擎不做車的戰略純粹性、Harness 是模型能力的負空間地圖、頭部廠商的 Demo 即標準、Prompt效力的重新分配、能離線解決的問題不要在線解決、弱模型不知道自己到了能力邊界、Coding域飛輪的三層基礎設施結構、Coding 飛輪的三層基礎設施與其他領域的結構性缺位、Prompt engineering 的繁榮是驗證器缺位的症狀、Agent加速的entropy與工程紀律）；archive/power.md（上游碾壓是全棧縱深、推理預算與止損判斷的反直覺關係、滲透率預期錯配）；working-papers/externalization-ceiling（W5）；working-papers/discard-capability（主張）；Reading/DeepSeek-Reasonix.md；Reading/DwarfStar4-ds4.md*
