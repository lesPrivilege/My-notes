# nanowhale 🐳：DeepSeek-V4 at 110M Parameters

- **來源**: https://github.com/huggingface/nanowhale
- **日期**: 2026-05-04 建立（4 天前）
- **作者**: HuggingFace（主要貢獻者：cmpatino）
- **類型**: repo
- **置信度**: high
- **模式**: deep-review

## 核心問題

DeepSeek-V4 的先進架構（MLA、Hyper-Connections、MoE with Sinkhorn routing）目前只有超大規模實現（158B），研究者和學習者無法在不動用數百 GB GPU 記憶體的情況下探索這些架構特性。現有 open-source reproduction 要嘛不完整、要嘛高度依賴 custom CUDA kernels。需要一個**純 PyTorch、可訓練、微型化**的參考實現。

## 核心方案

一個 **110M 參數**的語言模型，完整實作 DeepSeek-V4 的所有核心架構特徵，使用 **pure PyTorch**（無 custom CUDA/tilelang kernels），可在單張 H100 80GB 上從頭訓練：

1. **Multi-Head Latent Attention (MLA)** — 8 heads, 1 KV head (MQA), head_dim=96（32 RoPE + 64 NoPE）, q_lora_rank=160。這是 DeepSeek 系列最關鍵的 KV cache 壓縮創新，nanowhale 以 ~28K 行純 PyTorch 實現。

2. **Mixture-of-Experts (MoE)** — 4 routed + 1 shared expert, top-2 routing, SwiGLU FFN（dim 640）。在微型 scale 保留了 MoE 的 routing 和 load balancing 特性。

3. **Hyper-Connections (HC)** — hc_mult=4, Sinkhorn routing（2 iterations）。這是 DeepSeek-V4 區別於 V3 的重要創新之一，用於改善深層網路的梯度流。

4. **Multi-Token Prediction (MTP)** — 1 next-token prediction layer。雖然在小模型上 disabled，但實作保留。

5. **Training Pipeline** — 完整的 pretrain（5K steps on FineWeb-Edu）+ SFT（3K steps on SmolTalk）腳本，使用 HuggingFace Trainer/SFTTrainer。

## 訓練結果

| 階段 | 指標 | 數值 |
|------|------|------|
| Pretrain | Seen tokens | ~2.6B |
| Pretrain | Final loss | ~5.3 |
| Pretrain | Token accuracy | 33.8% |
| Pretrain | Throughput | 72ms/step (torch.compile, 1×H100) |
| SFT | Train loss | 15.41 → 10.22 |
| SFT | Eval loss | 2.873 → 2.607 |
| SFT | Token accuracy | 36.2% → 48.5% |
| Eval | PPL (pretrained) | 13.62 |
| Eval | PPL (SFT) | 12.90 |

## 證據

- **230 stars / 19 forks**（4 天內）— HuggingFace 官方 repo 的號召力
- **完整的 DeepSeek-V4 架構實現** — MLA、MoE with sqrtsoftplus scoring + hash-based routing、Hyper-Connections with Sinkhorn、Compressed Sparse Attention (CSA)、Grouped low-rank output projection
- **~28,000 行 `modeling_deepseek_v4.py`** — 從 deepseek-ai/DeepSeek-V4 官方 inference/model.py ported 到 HuggingFace Transformers 格式
- **模型已上傳 HuggingFace Hub**（`cmpatino/nanowhale-100m-base` 和 `cmpatino/nanowhale-100m`）
- **已知問題透明公開** — bf16 NaN、`from_pretrained` 相容性、vocab/embedding 比例失調
- **MIT License**
- **無法繞過的問題：129K vocab 中 embeddings 佔了 41M/110M = 37%**，限制了語言建模的實際 capacity

## 風險與弱點

- ⚠️ **bf16 NaN bug** — Hyper-Connections 架構在 bf16 下 overflow。必須用 fp32 訓練和推理。這對 H100 使用者意味著 ~2x memory 開銷和速度損失。README 坦承了但沒有給出 root cause 或 fix。
- ⚠️ **`from_pretrained` quirk** — Custom architecture 導致 HF 的 `from_pretrained` 會重新初始化部分權重。使用者必須手動 `load_state_dict`。這對 downstream 使用者是不小的 friction。
- ⚠️ **37% 參數在 embedding** — 129K vocab 對 110M 模型來說過大。這不是 bug 而是 design choice（為了 match DeepSeek-V4 tokenizer），但意味著模型實際的語言建模 capacity 遠低於 110M。有效非 embedding 參數只有 ~69M。
- ⚠️ **Training scale 極小** — Pretrain 只看了 ~2.6B tokens（對比 Llama 1 的 1T+）。SFT 只有 3K steps。這是一個 architecture validation 而非 production model。
- ⚠️ **4 天前才建立** — 專案處於非常早期的階段。沒有 issues（可能是關閉了或還沒人試），沒有 roadmap。
- ⚠️ **MoE 在小 scale 的意義不明** — 110M 參數下 4 experts + top-2 routing 的 MoE 是否比同等 dense model 更好？README 沒有提供 dense baseline 比較。

## 與其他 DeepSeek-V4 實現的對比

| 維度 | nanowhale | ds4.c (antirez) | DeepSeek 官方 |
|------|-----------|-----------------|---------------|
| Scale | 110M | 158B | 158B+ |
| Hardare | 1×H100 (train), CPU (eval) | 128GB+ Mac (Metal) | GPU cluster |
| 語言 | Python (PyTorch) | C + Metal | Python + tilelang |
| Custom kernels | 無（pure PyTorch） | 19 Metal shaders | tilelang CUDA |
| 目的 | 教學/架構驗證 | 本地推理 | Production |
| 可訓練 | ✅ | ❌ | Limited |
| 可用於 downstream | ⚠️ 僅供實驗 | ✅ Agent integration | ✅ |

## 待驗證問題

- **Hyper-Connections 在小模型上真的有幫助嗎？** — HC 是為超大模型設計的梯度流機制。在 8 層 110M 模型上，它的貢獻可以與標準殘差連接區分嗎？如果沒有消融實驗（with/without HC），我們無法判斷。
- **bf16 NaN 的根本原因** — 是 HC 的數值範圍問題還是 MLA 的某個中間激活值 overflow？如果能確定，可能有簡單 fix（如 gradient scaling 或特定的 activation normalization）。
- **tokenizer 選擇的 tradeoff** — 選用 DeepSeek-V4 的 129K vocab 使得 embedding 層佔了 37% 參數。這對於多語言任務可能有必要，但對純英文實驗的代價很高。是否有計畫提供更小 vocab 的變體？
- **MoE vs Dense 在 110M scale 的比較** — 如果 MoE 在 110M 不比 dense 好，那這個 repo 的價值在於「完整實現」而非「高效架構」。這是否偏離了「研究 DeepSeek-V4 架構」的目標？
