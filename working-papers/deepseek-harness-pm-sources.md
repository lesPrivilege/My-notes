# DeepSeek 做 Harness — 引用源原文

> 配合 deepseek-harness-pm-2026-05-17.md 使用。按论文章节排列。

---

## 一、外化天花板：harness 工程的结构性约束

### S1. 外化的天花板

> 當前 AI 工程所有改進（prompt / harness / managed agents / alignment monitoring）本質都是同一件事：externalization——在不改變模型權重的前提下重組外部運行時環境。天花板不來自工程努力不足，而來自模型底層表徵的幾何結構，後者由訓練目標（有損壓縮）雕刻，不為使用需求服務。
>
> ——tech.md，2026-04-26

### S2. Harness 是臨時補丁

> 每個 harness 組件都是對當前模型某個缺陷的補丁。Anthropic 為 Sonnet 4.5 的 context anxiety 設計的 reset，在 Opus 4.5 變成死代碼——證明 harness 的保質期就是下一次模型更新的時間。
>
> ——tech.md，2026-04-26

### S3. Harness 是模型缺陷的考古層

> 每個 harness 組件都在回答一個歷史問題——「當時那一版模型做不到什麼？」。Sonnet 4.5 的 context reset 在 Opus 4.5 變成 dead code，Notion 為 Opus 4.6 寫的 tool-failure fallback 在 4.7 變成多餘。這不是「技能會過時」的普通現象，而是技能賴以存在的對象本身每輪都在被替換。更精確地說，harness 組件是對**特定模型、特定 effort level、特定任務分佈下某個具體失敗模式**的補丁——三個維度任一變化都可能讓補丁從有用變成無用甚至有害。
>
> ——tech.md，2026-05-16

### S4. 上游碾壓的不對稱性

> 外部 harness 公司的整個能力空間是上游的子集，生態繁榮不是護城河，是廠商路線圖還沒走到那裡的窗口期。Anthropic 發布 full-stack app builder 的那一刻，外面所有 AI app builder 的商業模式被降維。
>
> ——tech.md，2026-04-26

---

## 二、上游与下游的不对称

### S5. Harness 變薄與變厚是位置函數，不是路線分歧

> Anthropic 說 harness 應該變薄，小米說 harness 應該變厚，兩者都對但不對稱。Anthropic 站在模型前沿，看到的是 harness 的遞減價值；小米站在追趕期，看到的是 harness 的當下槓桿。關鍵在於：變厚策略有內在矛盾——你投入越多資源在框架上，模型進步越快讓這些投入貶值。變薄策略沒有這個矛盾，因此是收斂方向。
>
> ——tech.md，2026-04-26

### S6. 只做引擎不做車的戰略純粹性

> DeepSeek 客戶端極簡（僅 OCR chatbot），不卷 agent 框架、不做複雜 memory 系統，資源集中在模型架構和推理效率上。Agent 框架、工具鏈、SDK 生態是 harness 層的東西，閉源廠商投入大量資源去建，但最終都會開源或被複刻——DeepSeek 甚至直接兼容 Anthropic API format，連等都不用等。Harness 層沒有持久壁壘，模型層的架構效率才有。
>
> ——tech.md，2026-04-28

### S7. Harness 是模型能力的負空間地圖

> 上游廠商（Anthropic、Google）公開的工作流和產品設計選擇，其價值不在工作流本身的精巧，而在於它們暴露了廠商認為模型當前能可靠執行的邊界在哪裡。Cat Wu 團隊每次新模型發版後通讀 system prompt 移除模型不再需要提醒的部分，這個操作本身就是在持續繪製模型能力的等高線——被拆掉的腳手架標記了模型已經內化的能力，保留的腳手架標記了仍需外部補償的缺陷。
>
> ——tech.md，2026-04-28

### S8. 頭部廠商的 Demo 即標準

> OpenAI 開源航司客服 Agent demo 的殺傷力不在技術複雜度，而在它是官方出的。參考實現一旦開源，下游方案商就從賣架構設計降級為賣定制配置。同時 SDK 綁定把開發者鎖進 OpenAI 生態——demo 是教育市場的外衣，生態鎖定才是目的。
>
> ——tech.md，2026-04-26

### S9. Prompt效力的重新分配

> 同一份prompt文本，在個人私部署架構裡接近無用，在官方orchestration + 品牌系統grounding + 精細控件的架構裡產生斷層級效果。Claude Design（隨Opus 4.7推出）就是這個機制的產品化：Brilliant報告原先需20+ prompts的頁面現在2 prompts。不是prompt在貶值，是外化層的效力在沿架構邊界集中到廠商一側。
>
> ——tech.md，2026-05-02

### S10. 上游碾壓是全棧縱深，不是單點優勢

> 上游廠商的碾壓路徑不僅是模型能力，還包括標準定義權（Google Skills Repository + MCP 協議）、開發者工具鏈控制（ADK + agents-cli）、以及運行時基礎設施（Agent Runtime、fleet orchestration）。下游的生存空間不是被一個產品打掉的，而是被整個基礎設施棧逐層擠壓的。
>
> ——power.md，2026-04-28

---

## 三、Cache hit 作为跨层设计原理

### S11. 能離線解決的問題不要在線解決

> 「能離線解決的問題不要在線解決」是多層收斂的統一原理：從訓練（over-training 把高頻 pattern 燒進權重）、到架構（MoE 稀疏激活只調動相關 expert）、到推理（KV cache 命中避免重複計算 prefill）、到產品（system prompt caching 最大化前綴復用），每一層都在做同一件事——把重複的計算前置，運行時只處理真正的增量。這不是巧合，而是物理約束（memory bandwidth 是硬瓶頸）決定了所有抽象層的優化方向必然趨同。
>
> ——tech.md，2026-05-02

### S12. ds4 的 KV cache 磁盘化

> ds4 的 KV cache 作為磁盤一等公民（KV checkpoints written to disk with SHA1 keyed .kv files, exact prefix reuse across session switches and server restarts）。
>
> ——Reading/DwarfStar4-ds4.md，2026-05-17

### S13. Reasonix 的 cache-first loop

> Reasonix 的三段式 context 分段（IMMUTABLE PREFIX / APPEND-ONLY LOG / VOLATILE SCRATCH），99.82% cache hit rate（單日 435M input tokens，~$12 vs ~$61）。「Coupling to one backend is the feature, not a limitation.」
>
> ——Reading/DeepSeek-Reasonix.md，2026-05-17

---

## 四、Coding agent 的商业可持续性与飞轮基础

### S14. Coding 飛輪的三層基礎設施與其他領域的結構性缺位

> Coding 領域 harness 成熟的根本原因不是「發展更快」，而是三個前提條件同時滿足：可分解的中間表示（代碼本身可局部編輯）、廉價的自動驗證（編譯器/測試套件）、錯誤可逆且代價低。視頻生成缺前兩者，法律文書有第一個但缺後兩個。三條件是必要不充分的——古籍校勘三條件皆具，卻因跨域人才極少、數據組織範式與 LLM 訓練格式不兼容、OCR 前處理瓶頸而停滯。
>
> ——tech.md，2026-05-16

### S15. Coding域飛輪的三層基礎設施結構

> coding 領域 harness 加速的飛輪由三層現成基礎設施構成——context 採集層（agentmemory 等 persistent memory），微調信號層（issue→PR 的天然 (input, output, ground_truth) 三元組），驗證閉環層（測試套件 + type checker + compiler）。每層都有成熟方案，組合起來飛輪自轉。
>
> ——tech.md，2026-05-13

### S16. Prompt engineering 的繁榮是驗證器缺位的症狀

> 在缺乏自動驗證器、結構化中間表示和可採集 context 的領域，用戶能操控的界面只剩 prompt 一個旋鈕。prompt 模板市場、資深行業人士教你寫提示詞、威脅/PUA「提高回答質量」——本質上是在驗證器缺位條件下，用經驗規則近似替代反饋回路。這些「技巧」拟合的是當前模型版本的特定行為，模型一更新就失效。
>
> ——tech.md，2026-05-16

### S17. Agent加速的entropy與工程紀律

> agent 加速產出速度的同時也加速了軟體 entropy（混亂速度）。對沖手段是更好的工程紀律，而非更多的 agent。這為 harness 貶值論提供了一個反例方向——有些 harness 不是補償 base model 的缺陷，而是補償 agent 加速帶來的新問題（混亂累積速度超過人類可管理閾值）。
>
> ——tech.md，2026-05-13

### S18. 渗透率預期錯配

> 當前 AI 行業的泡沫性質不是一般意義的估值泡沫或敘事泡沫，而是渗透率預期的錯配——市場定價隱含 AI 均勻渗透各行業的假設，但實際渗透分布極度不均。Coding 接近範式替換，其他領域停留在「高級搜索引擎」階段。
>
> ——power.md，2026-05-16

---

## 五、Harness PM 的位置

### S19. 弱模型不知道自己到了能力邊界

> 弱模型不知道自己到了能力邊界。這不是 harness 能解決的問題，而是模型 calibration 的內部問題，屬於底層表徵幾何結構的範疇。
>
> ——tech.md，2026-04-28

### S20. Harness 的非貶值價值（externalization-ceiling W5）

> Harness 的非貶值價值被低估：合規與治理（audit trail、policy enforcement）——制度需求，不因模型變強而消失；可觀測性（logging、tracing）——運行時基礎設施的需要獨立於模型能力；多模型編排——當底層模型頻繁切換時，統一接口層的價值反而上升；診斷記錄——哪個 harness 被拆了的歷史本身就是信號。
>
> ——working-papers/externalization-ceiling-2026-04-30.md，W5

### S21. 推理預算與止損判斷的反直覺關係

> Pro 在 agentic fetch 任務中表現比 Flash 更差，不是因為能力不足，而是更高的推理預算讓它在死胡同裡「想得更多」而非「更早放棄」。推理能力的增加不等於元認知判斷的改善——知道什麼時候該停是一個獨立於推理深度的能力維度。
>
> ——power.md，2026-04-28

### S22. discard-capability 主张

> 智能系統的核心競爭力正從「模型能做什麼」轉向「模型決定不做什麼」。在所有抽象層——架構設計、tokenization、attention、FFN 激活、緩存策略、推理調度——信息篩選機制正在取代信息處理能力，成為制約和定義系統上界的統一變量。
>
> ——working-papers/discard-capability-2026-05-11.md，主張
