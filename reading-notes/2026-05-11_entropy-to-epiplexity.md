# From Entropy to Epiplexity: Rethinking Information for Computationally Bounded Intelligence

- **來源**: arXiv:2601.03220 (HTML)
- **日期**: 2026-01-06 (v1)
- **作者/機構**: Marc Finzi (CMU), Shikai Qiu (NYU), Yiding Jiang (CMU), Pavel Izmailov (NYU), J. Zico Kolter (CMU), Andrew Gordon Wilson (NYU)
- **類型**: paper
- **置信度**: high（全文 HTML 直接獲取）
- **模式**: deep-review

## 核心問題

Shannon information 和 Kolmogorov complexity 假設 observer 有無限計算能力，這在 modern ML 中導致三個「表觀悖論」：(1) 訊息不能由確定性變換創造（但 AlphaZero、合成數據、數學推導都在創造新訊息）；(2) 訊息內容與 factorization order 無關（但 LLM 在 left-to-right 文本上遠比 reverse order 學得好）；(3) likelihood modeling 只是 distribution matching（但 model 可以從簡單規則生成的數據中學到 emergent structures，如 Game of Life 中的滑翔機）。這些框架無法回答現代 ML 的核心問題：不同數據的「可學習訊息量」如何比較？如何做 data selection？

## 核心方案

1. **Epiplexity（S_T）** — 新的訊息度量，定義為 compute-bounded observer 能從數據中提取的 structural information。形式化上，它是 minimum description length (MDL) 在計算約束下的變體：在所有能在時間 T 內 decompress 的 model 中，選擇使 data description length 最短的 model，其 bit 數即為 epiplexity。與之配對的是 time-bounded entropy（H_T），表示數據中隨機不可預測的部分。

2. **Prequential Coding** — 實用化測量方法。透過訓練曲線估算 epiplexity：用 `∫(L(t) - L∞) dt`（loss 曲線下超出 final loss 的面積）。愈高的 epiplexity 表示模型從數據中提取了愈多結構資訊。更嚴謹的版本是 requential coding：用 teacher model 和 student model 之間的 cumulative KL divergence。

3. **三悖論的統一解釋** — Epiplexity 天然解釋三個悖論：(1) 計算可以創造信息——pesudorandom generator 對 poly-time observer 是隨機的，但模擬 chaotic system 可以產出可學習的結構（Lorenz attractor, 細胞自動機 emergent patterns）；(2) 數據順序影響可提取的結構——某些 factorization 雖然 perplexity 更高，但 epiplexity 也更高，從而下游 OOD 表現更好；(3) likelihood modeling 不是 distribution matching——神經網路可以學到比數據生成程序更複雜的 programs（induction heads, emergent circuits）。

4. **ADO (Adaptive Data Optimization)** — 根據 loss reduction rate 動態選擇訓練數據的方法，基於 epiplexity 的理論基礎來做 data selection。

## 證據

- **ECA 與 Game of Life**：細胞自動機的 emergent patterns（滑翔機等）可以被 transformer 學到，其學到的 program（RASP-L 表示）比原始的 ECA rule 更複雜——說明了 Paradox 3。
- **Chess OOD 預訓練**：在 Stockfish 生成的棋局上預訓練，epiplexity 與 puzzle solving OOD 表現正相關。不同訓練數據來源的 epiplexity 差異顯著，且與 downstream 表現一致。
- **自然數據對比**：text data (OpenWebText) 的 epiplexity 遠高於 image data (CIFAR-5M)——解釋了為什麼語言模型的 pre-training 比 vision model 有更好的跨任務遷移能力。
- **Scaling laws 分析**：將 epiplexity 與 scaling law 參數關聯，推導 compute budget 最優的 data 和 model size 關係。
- **Data ordering 實驗**：某些數據排序（先難後易的 curriculum）雖然 training loss 更高，但 epiplexity 更高，最終 OOD 表現更好——直接實驗證據支持 Paradox 2 的消解。

## 風險與弱點

- ⚠️ **實用測量依賴 model 選擇**：Prequential coding 和 requential coding 都需要具體的 model class（transformer, MLP 等），epiplexity 值對 model architecture 敏感。同一數據在不同架構上的 epiplexity 可能不同，削弱了其作為「數據本身屬性」的宣稱。
- ⚠️ **計算成本**：準確測量 epiplexity 需要 training multiple models at different compute budgets 或 teacher-student 設置——這並不比直接做 downstream evaluation 便宜。ADO 作為應用也需要大量 trial。
- ⚠️ **OOD 相關性 vs 因果性**：證明 epiplexity 與 OOD generalization 相關，但相關性不等於因果關係。高 epiplexity 數據可能只是 proxy 了某種更根本的數據屬性（如多樣性、覆蓋率）。
- ⚠️ **缺乏大規模語言模型驗證**：自然數據實驗在 OpenWebText 上用 small GPT-2 scale 模型。epiplexity 對現代大規模 LLM（7B+）的 data selection 是否有效尚未驗證。
- ⚠️ **Ambiguity on information creation**：Paradox 1 中的「信息可被創造」在 epiplexity 框架下成立，但是否只是重新定義了「信息」一詞？Classical 意義上沒有創造，epiplexity 意義上創造了——這個 tension 在論文中沒有完全誠實地 unpack。
- ⚠️ **不是特定 OOD 的 guarantee**：論文誠實承認 epiplexity 只測量 structural information 的總量，不保證這些 structure 對特定下游任務有用。高 epiplexity 數據可能學到無關的複雜 circuit。

## 待驗證問題

- Epiplexity 與現有 data selection 方法（DSIR, D4, influence functions, perplexity-based filtering）的直接對比如何？
- 當 model scale 增大時 epiplexity 趨向收斂還是增長？如果上界由計算預算決定，那麼更大 model 應該能提取更高 epiplexity——這與 Chinchilla scaling 的關係是什麼？
- Epiplexity 對 synthetic data generation 的指導——是否存在最優策略來最大化 epiplexity per token？
