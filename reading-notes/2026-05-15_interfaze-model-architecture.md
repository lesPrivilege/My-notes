# Interfaze: A new model architecture built for high accuracy at scale

- **来源**: interfaze.ai/blog
- **日期**: ~2026-04/05（未標明確切日期）
- **作者/机构**: Interfaze (Jigsawstack)
- **类型**: article (產品發表)
- **置信度**: high

## 核心問題

通用 transformer/LLM 在 deterministic tasks（OCR、結構化輸出、物件偵測、STT）上準確度不足、成本偏高；傳統 DNN/CNN 雖準確但 inflexible。Interfaze 試圖 hybrid 兩者，在 flash/mini 級定價區間提供更高的確定性任務準確度。

## 核心方案

1. **Hybrid architecture** — 融合 DNN/CNN 的 task-specific 精度（bounding boxes、confidence scores）與 omni-transformer 的 flexibility，在同一個 shared vector space 中運作。
2. **Partial model activation（`<task>` tags）** — 透過 system prompt 中的 `<task>ocr</task>` tag 只激活部分權重，換取 faster + cheaper + deterministic output。
3. **Built-in toolchains** — web index（多個 SERP + 自有 crawler）+ long audio transcription（1h35m 在 ~50 秒內完成）+ 1M context window。
4. **Pricing parity with flash/mini tier** — $1.50/M input, $3.50/M output。

## 證據

| Benchmark | Interfaze | 最佳對手 | Delta |
|-----------|-----------|---------|-------|
| OCRBench V2 | **70.7%** | 55.8% (G3F) | +14.9pp |
| olmOCR | **85.7%** | 81.9% (Grok) | +3.8pp |
| RefCOCO (object detection) | **82.1%** | 75.5% (Sonnet 4.6) | +6.6pp |
| VoxPopuli WER ↓ | **2.4%** | 4.0% (G3F) | -1.6pp |
| Spider 2.0-Lite | **52.9%** | 49.6% (Sonnet) | +3.3pp |
| GPQA Diamond | 89.9% | 89.9% (Sonnet) | tie |

STT 速度：209s audio / 1s compute，~1.5× Deepgram Nova-3，~11× Gemini-3-Flash。

## 風險與弱點

- ⚠️ 所有 benchmark 數據由 Interfaze 自行發布，無第三方審計。
- ⚠️ Architecture 細節模糊——無技術論文或架構圖，"hybrid" 可能是 product-level routing 而非真正的 architecture innovation。
- ⚠️ General reasoning 無優勢（GPQA Diamond 與 Sonnet 4.6 打平），非 general-purpose replacement。
- ⚠️ SOB benchmark 由 Interfaze 自己上週才推出，尚未經第三方採用。

## 待驗證問題

- Hybrid architecture 的真實實現方式（同一模型權重 vs product-level routing）？
- Partial model activation 的 compute saving 量化數據？
- Interfaze 與 Jigsawstack 的關係——從 product 長出來的能力還是獨立 research？
