# CLI Review Prompts — 執行路徑追蹤 (v2)

> 四份 prompt：DeepSeek-TUI、Reasonix、ds4（執行路徑追蹤）+ Pi（細顆粒度全面 review）
> 目的：補全現有 audit 的盲區 — 從 claim verification 升級到 execution path tracing
> 用法：在對應 repo 目錄下開 Claude Code session，貼入 prompt 執行
> v2：根據 review 修訂 — 加入異步邊界標記、錯誤路徑、幻覺防護、驗證步驟

---

## Prompt 1: DeepSeek-TUI

```
你是一個架構審計員。這個 repo 是 DeepSeek-TUI，一個 14-crate Rust workspace 的終端 coding agent。

我已經做過 claim-level 的源碼審計（驗證了 README 宣稱 vs 實際代碼）。現在需要補全的是執行路徑級別的理解。不需要驗證功能是否存在，不需要解釋功能語義，只要追蹤函數之間的調用關係。

## 通用規則（每個任務都適用）

1. **所有行號必須是 grep/read 確認過的，不可推測。** 如果你不確定行號，用 grep 搜索函數名確認。
2. **遇到 .await / async boundary 時**，標記為 `[ASYNC]`，記錄 spawned task 的入口函數和 resume 點。
3. **同時追蹤錯誤路徑**：每個關鍵環節，搜索 `Result::Err` / `unwrap` / `expect` / `if let Err` / `?` operator 的處理邏輯，簡要標注「出錯時走哪條路」。
4. **跨 crate 的 trait method dispatch**：如果 call chain 跨了 crate 邊界（通過 trait impl），明確標出 trait 定義位置和具體 impl 位置。
5. **如果 prompt 中給的函數名在代碼中找不到**，不要虛構。搜索相關關鍵詞（如 route、dispatch、handler）找到實際的函數名。
6. 輸出格式：`函數名 (文件:行號)` → 下一個函數。不要加任何「這個設計很好」之類的評價。

## 任務 A：Crate 依賴圖

讀根目錄 Cargo.toml 和每個 crate 目錄的 Cargo.toml，畫出 14 個 crate 之間的依賴關係。格式：

```
crate-name
  ├── depends on: [list]
  └── depended by: [list]
```

特別標出：
- 哪個 crate 是 CLI 入口（`deepseek` binary）
- 哪個 crate 是 TUI 入口（`deepseek-tui` binary）
- 哪個 crate 持有 engine 主循環
- 哪些 crate 是 leaf（不被其他 crate 依賴）

## 任務 B：單輪 Turn 的完整 Call Chain

從用戶在 TUI 中按下 Enter 發送 prompt 開始，追蹤到下一輪 Turn 開始之前。

1. **用戶輸入 → engine 收到 prompt**：TUI 層的事件處理到 engine 入口
2. **Mode 分支點**：找到 Plan/Agent/YOLO 判斷的 if/match 分支點，列出各分支的入口函數（不要解釋語義，只要分支點位置和各分支入口）
3. **Auto mode 的 model/effort selection**：auto_router → auto_reasoning 的調用鏈。先確認這兩個模組的實際文件名和入口函數
4. **Context 組裝**：system prompt + history + tools spec + 動態注入內容的組裝順序和函數
5. **API 調用**：從 context 組裝完畢到發出 HTTP 請求的完整路徑。注意：可能跨 crate（engine crate → client crate → transport layer），特別追蹤 trait dispatch 邊界。先確認 API response 的格式（SSE? raw JSON? streaming chunks?），再追 response parsing
6. **Tool call 解析**：先確認 DeepSeek API 返回的 tool call 格式（標準 tool_use / function_call / 自定義 DSML），再追蹤從 streaming response 到結構化 tool call 的解析路徑
7. **權限判斷**：allow/deny/ask 三值判定的代碼位置。注意：如果 ask 分支是 blocking 等待用戶輸入，標記為 `[ASYNC]` 並記錄 resume 點
8. **工具執行**：並行/串行分批的判斷邏輯 + 執行入口。注意 `[ASYNC]` 邊界
9. **LSP diagnostics injection**：在整個 call chain 中的介入點（在工具執行之後？在 session save 之前？）。注意 `[ASYNC]`（LSP 有 timeout）
10. **Sub-agent**：只追本輪的 spawn 路徑和送進 mailbox 的調用。sub-agent spawn 後進入獨立循環，不要追進去。標記 parent 在哪個點接收 sentinel
11. **Session save**：什麼觸發 save，save 了什麼，save 的 `[ASYNC]` 邊界
12. **下一輪狀態更新**：state 對象的哪些字段被更新

## 任務 C：Snapshot Rollback 的完整生命週期

追蹤 snapshot 系統（搜索 snapshot 相關模組，可能在 snapshot/ 目錄）：
- Pre-turn snapshot 的觸發函數（可能在 engine loop 外層或 src/bin/ / src/main.rs 層級，不一定在 call chain 內部）
- Post-turn snapshot 的觸發函數
- Restore 的完整路徑（用戶觸發 → git checkout → 狀態更新）
- Cap/prune 的觸發條件和執行邏輯

## 驗證步驟

完成後，輸出一段 shell 命令，grep 驗證報告中引用的每個關鍵函數名確實存在於對應文件中。格式：
```bash
grep -n "函數名" 文件路徑
```
每個任務至少驗證 5 個關鍵函數。
```

---

## Prompt 2: Reasonix

```
你是一個架構審計員。這個 repo 是 Reasonix，一個 TypeScript 的 DeepSeek-native coding agent。

我已經做過 claim-level 的源碼審計。現在需要追蹤 CacheFirstLoop 的完整執行路徑。

## 通用規則

1. **所有行號必須是 grep/read 確認過的，不可推測。**
2. **如果 prompt 中給的函數名在代碼中找不到**，不要虛構。搜索相關關鍵詞找到實際函數名。
3. **同時追蹤錯誤路徑**：每個關鍵環節，搜索 try/catch、throw、reject、error handling，簡要標注「出錯時走哪條路」。
4. 輸出格式：`函數名 (文件:行號)` → 下一個函數。不加評價。

## 任務 A：CacheFirstLoop.step() 一輪的完整邏輯

從 src/loop.ts 的主循環入口開始（可能是 step()、run()、或其他名字 — 先 grep 確認），追蹤一輪完整的執行：

1. **進入時的初始狀態**
2. **Context 組裝順序**：ImmutablePrefix（src/memory/runtime.ts）怎麼構建 → AppendOnlyLog 怎麼 append → VolatileScratch 怎麼 reset → 三者怎麼合併成最終的 messages array 發給 API
3. **API 調用**：先確認 transport 層（SSE streaming? raw JSON?），再追從 context 到 HTTP 請求到 streaming response parsing 的路徑
4. **Tool call 解析**：先確認解析的輸入是什麼 — raw streaming output，還是經過 repair pipeline 修改後的版本？再追解析邏輯
5. **Repair pipeline 介入點**：scavenge / truncation / storm 各自的觸發條件是什麼（不只是「在哪個環節」，要列出每個 pass 的 predicate — 什麼條件為 true 時才執行）。確認 repair 的輸入和輸出分別是什麼
6. **Tool 執行 — 拆成兩個子問題**：
   - (a) parallelSafe 判斷：在哪裡讀取 tool 定義的 parallelSafe 屬性
   - (b) 分組策略：連續的 parallel-safe calls 怎麼被分組，有沒有依賴關係判斷，結果按什麼順序返回
7. **結果回注**：tool results 怎麼 append 到 AppendOnlyLog
8. **VolatileScratch.reset()** 在哪裡調用
9. **循環終止條件** — 列出所有終止路徑：
   - model 返回 stop_reason = end_turn（或等效）
   - max_turns 限制
   - tool call 結果觸發停止
   - 用戶中斷
   - 其他（如果有）

## 任務 B：Compaction 的完整路徑

1. 什麼條件觸發 compaction（token count 閾值？turn count？手動？列出所有觸發路徑）
2. compaction 怎麼修改 AppendOnlyLog（它是 append-only 的唯一例外）
3. compaction 後對 prefix cache 的影響：壓縮改變了 token 序列，cache 一定 miss。代碼裡怎麼處理這個 trade-off？有沒有「壓縮節省的 token 成本 vs cache miss 的成本」的判斷邏輯？

## 任務 C：Cost Control 決策路徑

追蹤 Pillar 3（cost control）的所有決策點：

1. **flash-first 預設**：在哪裡配置，怎麼生效
2. **failure-signal auto-escalation**：列出所有被當作 failure signal 的條件（timeout? error response? empty result? tool call 失敗? 其他?），找到對應的 predicate 判斷代碼
3. **/pro 單 turn 提權**：用戶觸發 → 生效 → 回到 flash 的完整路徑
4. **turn-end auto-compaction**：觸發條件（和任務 B 的觸發可能重疊，標明關係）

## 任務 D：核心文件依賴圖

只列出以下文件的 import 關係（不要列所有文件）：
- src/loop.ts 及其 1-hop imports
- src/memory/runtime.ts 及其被 import 者
- src/repair/*.ts 及其被 import 者
- src/tools/*.ts 的調用者（誰 import 了它們）

如果 src/ 下文件超過 50 個，嚴格只追上述範圍。

## 驗證步驟

完成後，輸出 grep 命令驗證報告中引用的每個關鍵函數名。每個任務至少 5 個。
```

---

## Prompt 3: ds4

```
你是一個架構審計員。這個 repo 是 ds4，一個純 C 的 DeepSeek V4 Flash 專用推理引擎。

我已經做過 claim-level 的源碼審計。現在只需要追蹤兩條路徑。不需要讀量化器（gguf-tools/）和 GPU kernel — 只看 server 端的請求處理和 cache 邏輯。

## 通用規則

1. **所有行號必須是 grep/read 確認過的。** 這是一個上萬行的 C 文件（ds4_server.c），行號偏移風險極高。每個引用的行號都必須用 grep -n 確認。
2. **如果 prompt 給的函數名找不到**，搜索關鍵詞（route、dispatch、handler、cache、sha1、dsml、tool_memory、raw_dsml、remember、attach）找到實際函數名。
3. **同時追蹤錯誤路徑**：每個關鍵環節標注出錯時的處理（return error? goto cleanup? silent fallback?）。
4. 輸出格式：`函數名 (文件:行號)`。不加評價，不解釋什麼是 KV cache。

## 任務 A：請求生命週期

追蹤一個 POST /v1/chat/completions 請求從進入到 response stream 結束的完整路徑：

1. **Server framework 確認**：先確認 ds4 用什麼做 HTTP server（libmicrohttpd? 自定義 epoll? 其他?）。是否有 middleware 層（auth, rate-limit, CORS）。從 accept/listen 開始找
2. **路由分支**：在 ds4_server.c 中搜索 URL path 匹配邏輯（搜索 `/v1/chat/completions` 字符串），找到 handler 入口函數
3. **請求解析**：messages array 提取，token 化
4. **KV cache 查找**：
   - SHA1 key 生成（搜索 sha1 相關函數）
   - .kv 文件查找（搜索 kv_cache、kv_path 相關函數）
   - Cache hit 時：跳過多少 prefill，怎麼 resume
   - Cache miss 時：完整 prefill 的入口
   - 注意：cache 邏輯可能跨多個 .c 文件（ds4_server.c 之外），如果有則追蹤
5. **推理**：概要即可（入口函數 + 出口），不追 kernel 內部
6. **Tool call 生成**：
   - DSML 格式輸出的生成路徑
   - raw_dsml 存儲（搜索 raw_dsml、tool_memory_remember 或類似關鍵詞）
   - 下一輪回放路徑（搜索 tool_memory_attach、append_dsml 或類似關鍵詞）
   - Fallback 到 canonical rendering 的條件
7. **Response streaming**：SSE 格式化 + 輸出
8. **KV cache 持久化 — 分別追四種時機**：
   - cold：觸發函數 + 持久化方式
   - continued：觸發函數 + 持久化方式
   - evict：觸發函數 + 持久化方式
   - shutdown：觸發函數 + 持久化方式
   - 各自是 write-through / write-back / async flush?

## 任務 B：三協議路由

ds4 同時支持 OpenAI chat/completions、OpenAI responses、Anthropic messages。

1. **路由分支點**：三種協議各自的 URL path 和 handler 入口
2. **請求解析差異**：三種協議的 messages/input 格式不同，怎麼統一到內部表示？是否共用中間表示（unified message struct）然後分別 serialize 輸出？
3. **Response 格式化差異**：三種協議的 response 格式化是各自獨立路徑還是共用中間表示後分別 serialize？
4. **DSML 處理差異**：DSML tool call 在三種協議下的處理有無不同

## 驗證步驟

完成後，輸出 grep 命令驗證所有引用的函數名。由於 ds4_server.c 行號偏移風險高，額外輸出：
```bash
grep -n "函數名" ds4_server.c | head -3
```
確認每個關鍵函數的實際行號。至少驗證 10 個函數。
```

---

## Prompt 4: Pi（細顆粒度全面 Review）

```
你是一個架構審計員。這個 repo 是 Pi，一個 TypeScript 的 minimal coding harness / agent runtime。

這個 repo 體量小，我要做最細顆粒度的 review — 不只追執行路徑，要理解整個 runtime 的設計。

## 通用規則

1. **所有行號必須是 grep/read 確認過的，不可推測。**
2. **如果函數名找不到**，搜索相關關鍵詞找到實際函數名。
3. **同時追蹤錯誤路徑**：每個關鍵環節標注出錯時的處理。
4. 輸出格式：`函數名 (文件:行號)` → 下一個函數。不加評價。

## 任務 A：項目結構全景

1. 列出 src/ 下所有目錄和文件，標注每個文件的職責（一句話）
2. 畫出文件之間的 import 依賴圖（所有文件，不做裁剪 — 體量小可以完整列出）
3. 標出：
   - CLI 入口文件
   - Runtime loop 主文件
   - Context/memory 相關文件
   - Tool 定義和 dispatch 相關文件
   - Extension/plugin 系統文件
   - Provider/model 抽象層文件
   - Session 管理文件
   - Prompt/template 相關文件

## 任務 B：Runtime Loop 完整追蹤

這是最重要的任務。追蹤 Pi 的 agent loop 從頭到尾的完整邏輯：

1. **Loop 入口**：找到主循環的入口函數。是 while loop? recursive call? event-driven?
2. **Context 組裝**：
   - system prompt 從哪裡來（hardcoded? file? template?）
   - history/messages 怎麼管理（array? tree? 其他結構?）
   - tool specs 怎麼注入
   - 動態 context（memory, skills, MCP）怎麼注入
   - 最終組裝成什麼格式發給 API
3. **API 調用**：
   - provider 抽象層（怎麼切換 model/provider）
   - request 構建
   - streaming response parsing
4. **Tool call 處理**：
   - 從 response 中提取 tool calls 的邏輯
   - tool dispatch（怎麼從 tool name 找到具體實現）
   - tool 執行（同步? 異步? 並行?）
   - 結果回注到 context
5. **Context compaction**：
   - 觸發條件
   - 壓縮策略（摘要? 截斷? 選擇性保留?）
   - 壓縮後對 cache 的影響
6. **Session tree**：
   - session 的數據結構（是 tree 不是 linear?）
   - branching 邏輯（什麼時候創建分支）
   - 持久化方式
7. **循環終止**：所有終止路徑

## 任務 C：Extension System 深度分析

Pi 強調 extension 是核心機制。追蹤：

1. **Extension 加載**：怎麼發現和加載 extensions（file scan? config? registry?）
2. **Extension 接口**：extension 能做什麼？（添加 tool? 修改 prompt? 注入 context? 改變 loop 行為?）列出完整的 extension API surface
3. **Extension 和 runtime 的邊界**：extension 能不能修改 runtime loop 本身？還是只能在 hook points 注入？
4. **內建 vs 外部 extension**：哪些能力是 built-in，哪些是通過 extension 實現的？
5. **MCP 整合**：MCP servers 是作為 extension 加載的嗎？還是獨立的機制？

## 任務 D：Tool 系統完整圖譜

1. **所有內建 tools**：列出每一個，標注文件位置和功能（一句話）
2. **Tool 定義格式**：一個 tool 需要定義什麼（name, description, schema, handler, permissions?）
3. **Tool 權限模型**：有沒有 allow/deny/ask？還是其他權限機制？
4. **Tool 結果處理**：tool 返回的結果怎麼格式化後注入 context？

## 任務 E：Provider 抽象層

1. **支持哪些 provider**（列出所有）
2. **Provider 接口**：抽象層定義了什麼（chat, streaming, embedding, other?）
3. **Provider 切換**：怎麼配置和切換 provider
4. **Provider-specific 行為**：有沒有針對特定 provider 的優化（如 DeepSeek 的 prefix cache、Anthropic 的 cache control）

## 任務 F：Prompt / Skill / Template 系統

1. **System prompt**：怎麼構建，是否支持模板化
2. **Skills**：如果有 skill 系統，追蹤 skill 發現、加載、執行的完整路徑
3. **Prompt templates**：如果有模板系統，追蹤模板解析和變量注入

## 任務 G：與 Reasonix / Claude Code 的結構對比

基於前面的分析，列一個對比表：

| 維度 | Pi | Reasonix (已知) | Claude Code (已知) |
|------|-----|-----------------|-------------------|
| Loop 結構 | ? | CacheFirstLoop | queryLoop |
| Context 管理 | ? | 三區塊 partition | SessionMemory + autocompact |
| Tool dispatch | ? | parallel with parallelSafe | concurrency safety 分批 |
| Extension/Plugin | ? | Skills + MCP | Skills + MCP + hooks |
| Session | ? | per-workspace persistent | MEMORY.md + SessionMemory |
| Compaction | ? | turn-end auto | staged collapse + reactive |
| Provider | ? | DeepSeek-only | Anthropic-native |
| Philosophy | ? | cache-first | continuity-first |

只填你在本次 review 中能從源碼確認的部分。Reasonix 和 Claude Code 欄位我已經知道，填 "已知" 的值即可。

## 驗證步驟

完成後，輸出 grep 命令驗證報告中引用的每個關鍵函數名。每個任務至少 3 個。
```

---

## 執行建議

1. **順序**：Pi 先（體量最小，且細顆粒度 review 能建立 baseline 理解）→ Reasonix → DeepSeek-TUI → ds4
2. **Session**：每個 repo 開獨立 session，在 repo 根目錄下執行
3. **產出命名**：`2026-05-21_{project}-execution-path.md`（Pi 的命名為 `2026-05-21_Pi-deep-review.md`）
4. **驗證**：CLI 產出的驗證 grep 命令直接跑一遍，確認函數名和行號
5. **回收**：跑完後把報告複製到 source-audit/ 文件夾
