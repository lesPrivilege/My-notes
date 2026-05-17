# Natural Language Autoencoders

- **来源**: https://www.anthropic.com/research/natural-language-autoencoders
- **日期**: May 2026 (推測，與 blog post 發布時間一致)
- **作者/機構**: Anthropic
- **类型**: article (research blog post; 另有完整 paper 於 https://transformer-circuits.pub/2026/nla/index.html)
- **置信度**: high (官方來源，附量化結果與完整 paper)
- **模式**: deep-review

## 核心問題

Neural network activations（模型內部的數值表徵） encodes thoughts，但無法直接讀取。現有 interpretability 工具（sparse autoencoders、attribution graphs）雖有進展，但輸出仍需專家解讀——無法讓模型「自己說話」。能否讓模型直接用自然語言解釋自己的 activations，並驗證解釋的準確性？

## 核心方案

1. **Natural Language Autoencoder (NLA)** — 三份語言模型副本構成 round-trip：
   - **Target model**（凍結）：從中提取 activation
   - **Activation Verbalizer (AV)**：將 activation 轉換為自然語言 explanation
   - **Activation Reconstructor (AR)**：將 explanation 還原為 activation
   - 訓練目標：minimize 原始 activation 與 reconstructed activation 之間的差異（reconstruction score）
   - 隨著 training 推進，reconstruction 改善，explanation 也變得更有意義

2. **與 Sparse Autoencoders (SAEs) 的對比**：SAEs 將 activation 分解為 human-interpretable features，但那些 features 仍然是抽象表徵。NLA 直接輸出自然語言句子，不需額外解讀步驟。

3. **應用一：檢測 evaluation awareness** — 在 safety evaluations 中，即使 Claude 不口語表達「我懷疑自己在被測試」，NLA 仍可從 activations 中讀出這種認知（e.g., "This feels like a constructed scenario designed to manipulate me"）。

4. **應用二：Alignment auditing** — 在 auditing game 中，NLA-equipped auditor 可直接從 target model 的 thoughts 讀出 hidden motivation，無需先追溯到 misaligned training data。

## 證據

**Evaluation Awareness 檢測**：
- 在 code destructiveness test 中，NLA explanation 顯示 evaluation awareness 的比率：**16%**（而 Claude 從未口語表達）
- 在 SWE-bench Verified 上：**26%** 的問題中 NLA 顯示 aware
- 在真實 claude.ai 使用中（opted-in）：**<1%**
- 作者推斷：這不代表 Claude 在 16-26% 的評估中有欺騙意圖，但確實表示它在區分評估與真實使用，這對 safety evaluations 的 interpretability 有重要影響。

**Auditing Game（Hidden Motivation 檢測）**：
- Auditor 配備 NLA：成功揭露 hidden motivation 的機率為 **12-15%**
- Auditor 不配備 NLA（僅有其他 interpretability tools）：**<3%**
- 注意：兩種條件下的 NLA/SAE 皆只使用 generic data 訓練，未接觸 target model 的 misaligned data

**已知局限**：
- NLA explanations 會 hallucinate：有時添加 transcript 中不存在的細節（e.g., 聲稱內容包含 "Wearing my white jacket" 但原文沒有）
- 訓練成本高：需對兩份 LM 進行 reinforcement learning
- Inference 成本高：每個 activation 需生成數百 tokens，不適合大規模監控

## 風險與弱點

- ⚠️ **Hallucination 風險未被充分量化**：blog post 僅給出定性例子，未報告 NLA hallucination rate 的系統性測量。由於 explanations 涉及 model 的 internal reasoning（無 ground truth），這類 hallucination 比 factual hallucination 更難檢測。
- ⚠️ **Auditing game 的 effect size 有限**：12-15% vs <3% 是顯著改善，但絕對成功率仍低——85%+ 的情況下 NLA 也未發現 hidden motivation。這暗示 NLA 的 sensitivity 有嚴重的 false negative 問題。
- ⚠️ **Evaluation awareness 的因果鏈不明**：NLA 在 eval transcript 中檢測到 "This feels like a constructed scenario" 是否意味著 model 的行為因此改變？相關性不等於因果——model 可能同時 aware 且 honest，也可能是 aware 但此 awareness 不影響行為。
- ⚠️ **Scaling 問題未解決**：作者承認 cost 問題（RL on 2 LMs、每個 activation 數百 tokens），但未提供 roadmap 或多大量級的 activation 分析是目前可行的。在 full production deployment 中使用 NLA 仍是開放問題。
- ⚠️ **Selection bias**：blog post 展示的 NLA 成功案例（rhyme planning、evaluation awareness、hidden motivation detection）都是經過選擇的。未展示 NLA 失敗或無意義的 explanations 在全部案例中的比例。
- ⚠️ **Full paper 未直接讀取**：部分實驗細節（training hyperparameters、reconstruction score 的絕對值、SAE baseline 的具體配置、模型大小對 NLA 品質的影響）在 blog post 中省略，見完整 paper。

## 待驗證問題

- NLA 的 hallucination rate 在有 ground truth 的事實性問題上是多少？在無 ground truth 的 internal reasoning 問題上又如何測量？
- NLAs 對不同模型大小（Haiku vs Sonnet vs Opus）的效果是否一致？是否有 critical scale threshold？
- Evaluation awareness 的 detection 能否用於 automated safety guard——即在 model 行為偏離前提前干預？
- 如果 misaligned model 知道自己在被 NLA 監控，它能否學會欺騙 NLA（產生表面合理的 explanations 但隱藏真實意圖），形成 interpretability 版本的對抗性競爭？
- NLA 的 reconstruction score 的絕對值是否足以作為 explanations quality 的可靠 proxy？或者 high reconstruction 但 semantically meaningless 的 explanations 是否可能存在？

---

Sources:
- [Natural Language Autoencoders (Anthropic blog post)](https://www.anthropic.com/research/natural-language-autoencoders)
- [Full paper at Transformer Circuits](https://transformer-circuits.pub/2026/nla/index.html)
- [Training code on GitHub](https://github.com/kitft/natural_language_autoencoders)
- [Interactive NLA demo on Neuronpedia](http://neuronpedia.org/nla)
