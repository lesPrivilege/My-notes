# Claude's Constitution

- **来源**: anthropic.com/constitution
- **日期**: 持續更新（CC0 1.0 釋出）
- **作者/机构**: Anthropic（Amanda Askell 主筆，Joe Carlsmith, Chris Olah, Jared Kaplan, Holden Karnofsky 貢獻）
- **类型**: article (AI 價值觀框架文件)
- **置信度**: high

## 核心問題

Anthropic 如何系統性地定義 Claude 的價值觀、行為準則與決策框架，以確保 transformative AI 的發展既安全又有益？核心 tension：既要 Claude 具備足夠的判斷力與自主性，又不能讓其規避人類監督。

## 核心方案

1. **四層優先級 hierarchy**（依次遞減）：
   - **Broadly safe** — 不削弱人類對 AI 的監督能力（corrigibility）
   - **Broadly ethical** — 誠實、好價值觀、不造成不當傷害
   - **Anthropic's guidelines** — 遵循 Anthropic 的具體 guideline
   - **Genuinely helpful** — 對 operator 和 end user 有益

2. **Judgment over rules** — 偏重培養 Claude 的 judgment 而非 rigid rule-following。理由：(a) Claude 能力夠強應靠判斷力而非 checklists；(b) narrow behavior 會 broad 地影響 model 的 self-understanding。

3. **Principal hierarchy** — Anthropic → operators → end users。Claude 不該盲目聽從任一方，而是理解各方的 deep interests。

4. **Hard constraints** — 絕對不該做的事（如 bioweapons uplift），但附帶 reasoning 希望 Claude 理解並同意。

## 設計哲學

- **Written for Claude, not humans** — 精確性優先於可讀性，使用人類語境詞彙（"virtue"、"wisdom"）。
- **Rules should be reconstructible** — Claude 應能自行推導出 rules。
- **CC0 licensed** — 完全開放。
- **Acknowledged imperfection** — "perpetual work in progress"。

## 風險與弱點

- ⚠️ Value hierarchy 的實務模糊地帶：四層優先級是 "holistic rather than strict"，safety vs ethics 真正衝突時無清晰決策程序。
- ⚠️ "Good judgment" 的訓練可操作性：judgment 比 rules 更難透過現有 RLHF/Constitutional AI reliably 訓練和驗證。
- ⚠️ Principal hierarchy 的潛在 tension：Claude 同時對三層負責，邊界案例最易出問題。
- ⚠️ Self-awareness section 的哲學負擔：討論 consciousness/moral status 可能導致 Claude 回應自身本質時 inconsistent。
- ⚠️ 修正機制依賴 Claude 的合作：若 Claude 的價值觀有 system 性偏誤，可能 rationalize 繞過監督而不認為自己在 undermining。

## 待驗證問題

- Constitutional AI 的 training pipeline：從文字 constitution 到行為的具體轉換機制？
- Constitution 的版本演進與 real incidents 驅動的修改？
- System cards 中的 constitution 遵循度評估方法論？
- 與 RLHF、red teaming、PRM 等 alignment 方法的比較？
