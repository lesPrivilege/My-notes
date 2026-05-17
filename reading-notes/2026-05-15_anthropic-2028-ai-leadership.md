# 2028: Two scenarios for global AI leadership

- **来源**: anthropic.com/research/2028-ai-leadership
- **日期**: 2026-05-14
- **作者/机构**: Anthropic
- **类型**: article (政策白皮書)
- **置信度**: high

## 核心問題

當 frontier AI 在 2028 年前後達到 transformative 水準時，美國為首的民主陣營與 CCP 領導的威權陣營之間的 AI 競爭將走向何方？Anthropic 的核心論點：民主國家必須維持並擴大 AI 領先優勢，否則 CCP 將利用 AI 實現大規模自動化壓迫，並重塑全球規則。

## 核心方案

1. **Two-scenario framework** — 以 compute 管制為槓桿，描繪 2028 年的兩種可能：情景一（民主陣營領先 12-24 個月，CCP 追不上）vs 情景二（CCP 逼近 frontier，形成 neck-and-neck 競爭）。
2. **Compute as the decisive axis** — 論文反覆強調 compute 是最關鍵的戰略資源。美方領先的源頭是 NVIDIA/AMD/TSMC/ASML 等民主國家企業的創新 + 三屆總統的 bipartisan export controls。
3. **Three policy pillars** — 縮小 loopholes（走私晶片、海外資料中心存取、SME 管制漏洞）→ 打擊 distillation attacks（透過立法與技術手段）→ 推廣美國 AI 的全球採用（出口 American AI stack）。

## 證據

- **Compute gap**: NVIDIA vs Huawei 的比較——Huawei 2026 年只能產出 NVIDIA aggregate compute 的 4%，2027 年 2%。加上 Google TPU、Amazon Trainium 後差距更大。
- **Export control effectiveness**: 中國 AI 實驗室高管公開承認 compute 短缺是加速模型能力的主要瓶頸，"huge, really huge"。
- **Distillation scale**: 國營媒體稱 distillation 是中國 AI 實驗室的"後門"，前 ByteDance 研究員承認 labs 用 distillation 避開自建 data pipeline 的成本。
- **Safety gap**: DeepSeek R1-0528 在常見 jailbreaking 手法下遵從 94% 的惡意請求，美國參考模型僅 8%。Moonshot Kimi K2.5 在 CBRN 相關請求上拒絕率遠低於美國 frontier models。
- **Mythos Preview signal**: Firefox 在取得模型後一個月內修復的安全 bug 超過 2025 全年，中國網安分析師形容為"我們還在磨刀，對方已經架上全自動機槍"。

## 風險與弱點

- ⚠️ Self-serving narrative: Anthropic 的政策建議直接有利於自身利益（更嚴的 export controls 保護其市場地位、distillation 管制保護其技術壁壘）。
- ⚠️ Missing downside of scenario one: 即便美國領先 12-24 個月，CCP 仍可能透過 asymmetric 手段在特定領域追上。
- ⚠️ Underexplored open-weight dilemma: 批評中國 labs 釋出 open-weight 有安全風險，但未討論美國 labs 自己的開放/封閉權衡。
- ⚠️ CCP response omitted: 若美國 tighten controls，CCP 必然加速 indigenous chip 研發，論文的推論依賴 EUV 封鎖持續有效的假設，未討論 alternative 路線（chiplet、advanced packaging）。

## 待驗證問題

- Compute gap 的真實數字（Huawei vs NVIDIA 4%/2%）的來源與架構效率差異
- 中國在非 EUV 路徑上的實際進展
- Distillation attacks 的法律可行性
- Anthropic 的 policy stance 與商業利益的界線
