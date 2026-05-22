# DeepSeek-TUI 社區口碑問題：源碼根因分析報告

> 基於執行路徑審計報告（`2026-05-21_deepseek-tui-execution-path.md`）與 `crates/tui/src/` 實碼交叉比對。
> 版本 0.8.39，14-crate Rust workspace，~209,569 行 Rust。

---

## 🔴 根因一：Session 隨時間必然退化直至崩潰

### 機制鏈

**① 消息只增不減**

`Session` 結構體的核心欄位：

```rust
// crates/tui/src/core/session.rs:15-94
pub struct Session {
    pub messages: Vec<Message>,     // ← 只進不出（除非手動 /compact）
}
```

每次 append：

```rust
// crates/tui/src/core/session.rs:173-175
pub fn add_message(&mut self, message: Message) {
    self.messages.push(message);
}
```

`handle_deepseek_turn` 中有 **12 處** 調用 `add_session_message`（`turn_loop.rs:62,872,887,941,962,1024,1036,1810,1852,1900`），加上 `lsp_hooks.rs:119` 和 `capacity_flow.rs:573`，每一次工具調用或 assistant 回覆都會 push。

**② 每次 API 請求都發送全部消息**

```rust
// crates/tui/src/core/engine/turn_loop.rs:1937-1943
pub(super) fn messages_with_turn_metadata(&self) -> Vec<Message> {
    self.session.messages.clone()   // 克隆整個 Vec
}
```

請求 payload 隨輪數線性增長：

```rust
// turn_loop.rs:285-288
let request = MessageRequest {
    messages: self.messages_with_turn_metadata(),
};
```

**③ 每次 turn 完成都克隆全部消息發回 UI**

```rust
// crates/tui/src/core/engine.rs:825-836
async fn emit_session_updated(&self) {
    let _ = self.tx_event.send(Event::SessionUpdated {
        messages: self.session.messages.clone(),  // O(messages) 克隆
    }).await;
}
```

**④ 每次 checkpoint 都序列化全部消息為 JSON**

```rust
// crates/tui/src/session_manager.rs:238-251
pub fn save_session(&self, session: &SavedSession) -> std::io::Result<PathBuf> {
    let content = serde_json::to_string_pretty(session)  // 完整 JSON 序列化
        .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    write_atomic(&path, content.as_bytes())?;
}
```

```rust
// crates/tui/src/session_manager.rs:254-262
pub fn save_checkpoint(&self, session: &SavedSession) -> std::io::Result<PathBuf> {
    let content = serde_json::to_string_pretty(session)
        .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
    write_atomic(&path, content.as_bytes())?;
}
```

Persistence actor 用 unbounded channel + latest-wins coalescing（`persistence_actor.rs:100,106-184`），但合併只減少寫入次數，不減少每次寫入的數據量。

**⑤ Auto-compaction 預設關閉**

```rust
// crates/tui/src/settings.rs:278-287
// v0.8.11: default flipped to `false` to stop the engine from
// routinely rewriting the prompt prefix, which breaks DeepSeek
// V4's prefix cache (~90% discount on cached prefix tokens) and
// ends up costing more than the compaction itself saves. With
// V4's 1M-token window the user has plenty of headroom to run
// long sessions without auto-trimming, and the explicit
// `/compact` slash command + `auto_compact = on` opt-in remain
// available for users / agents that decide compaction is
// worth the cache hit on their workload (#664).
auto_compact: false,
```

```rust
// crates/tui/src/settings.rs:999-1007
fn default_settings_disable_auto_compact_to_protect_v4_prefix_cache() {
    let settings = Settings::default();
    assert!(!settings.auto_compact);
}
```

Compaction 啟用時還有 `auto_floor_tokens`（`compaction.rs:29-41`，預設 500K tokens）做為硬地板，低於此值拒絕 compact — 再一次為了保護 prefix cache。

**⑥ AGENTS.md 的承認**

```markdown
// AGENTS.md:116
Long sessions in DeepSeek TUI WILL degrade and crash if you work sequentially.
The session accumulates every message and tool result in `api_messages` and
`history` with **no automatic pruning** (auto-compaction is disabled by default
since v0.6.6).
```

### 衝擊

用戶使用 30–60 分鐘後，session 累積數百條消息 → 每次 API 請求 payload 暴漲 → token 消耗增加 → 序列化/反序列化變慢 → UI 響應退化 → 最終 OOM 或崩潰。**這是 #1 棄用原因。**

---

## 🔴 根因二：Scope Creep 導致膨脹失焦

### 2.1 專案想同時做兩件事，但沒做好任何一件

**量化規模：**

| 指標 | 數值 |
|---|---|
| workspace crates | 14 |
| `.rs` 源文件 | 272 |
| 總行數 | ~209,569 |
| Cargo.lock 依賴 | 655 |
| config.example.toml | 559 行，26+ 區段 |

### 2.2 `deepseek-core` 是空殼

"Core" crate 的 Cargo.toml：

```toml
// crates/core/Cargo.toml
[dependencies]
deepseek-agent      = { path = "../agent" }
deepseek-config     = { path = "../config" }
deepseek-execpolicy = { path = "../execpolicy" }
deepseek-hooks      = { path = "../hooks" }
deepseek-mcp        = { path = "../mcp" }
deepseek-protocol   = { path = "../protocol" }
deepseek-state      = { path = "../state" }
deepseek-tools      = { path = "../tools" }
```

它只是其他 8 個 crate 的 re-export 層。**真正的 engine 在 TUI crate 內**，且完全不引用 `deepseek-core`：

```rust
// crates/tui/src/core/engine.rs:10-73（imports）
use crate::client::DeepSeekClient;
use crate::compaction::{...};
use crate::config::{...};
use crate::mcp::McpPool;
use crate::tools::{...};
// ← 無任何 deepseek_core 引用
```

### 2.3 實際存在的功能面

- 11 個 API provider（`config.rs:67-79`）
- Sub-agent + RLM（`crates/tui/src/tools/subagent/mod.rs`，4,561 行）
- LSP integration（`crates/tui/src/lsp/`）
- MCP server 管理（`crates/tui/src/mcp.rs`，4,021 行）
- Sandbox（landlock/seatbelt/windows，`crates/tui/src/sandbox/`）
- Snapshot git repo（每次 turn 執行完整 git 操作）
- Automation manager + Task manager + Capacity controller
- 三種 mode（Plan/Agent/YOLO），各自行為不同
- 25+ 工具（`crates/tui/src/tools/`，每個獨立 .rs 文件）

Config 結構 42 個欄位 + 12 個子結構（`config.rs:887-1208`），從 `sandbox_backend` 到 `vision_model` 到 `managed_config_path`。

### 衝擊

對只想「在 terminal 裡跟 DeepSeek 聊天」的用戶來說，這是一堵功能牆。學習曲線陡峭，配置複雜，且多數功能與其使用場景無關。

---

## 🔴 根因三：雙 Binary 架構造成安裝/更新混亂

### 3.1 機制

CLI dispatcher（`deepseek`）通過 `Command::new()` 以子進程方式 spawn TUI binary（`deepseek-tui`）：

```rust
// crates/cli/src/lib.rs:1374-1484
fn build_tui_command(...) -> Result<Command> {
    let tui = locate_sibling_tui_binary()?;   // 在 PATH/同目錄找 sibling
    let mut cmd = Command::new(&tui);          // 以子進程 spawn
}
```

```rust
// crates/cli/src/lib.rs:1531-1565
fn locate_sibling_tui_binary() -> Result<PathBuf> {
    let primary = dispatcher.with_file_name("deepseek-tui");
    if primary.is_file() { return Ok(primary); }
    bail!("Companion `deepseek-tui` binary not found at {}.\n\
           The `deepseek` dispatcher delegates interactive sessions to a sibling \
           `deepseek-tui` binary. ...");
}
```

### 3.2 無版本相容性檢查

搜索 `version.*check`、`semver`、`VersionReq` 在 `crates/` 範圍內，**零結果**。CLI 和 TUI 之間沒有版本協商。如果用戶只更新了其中一個，行為未定義。

### 3.3 AGENTS.md 的處理方式

```markdown
// AGENTS.md:15-20
**Two binaries, two installs.** The dispatcher resolves and spawns `deepseek-tui`
as a sibling on PATH for interactive use, so installing only the CLI leaves the
TUI stale and your fix won't appear to run. ...If a fix you just made "isn't
taking effect," check `stat -f '%Sm' ~/.cargo/bin/deepseek-tui` before reaching
for `tracing::debug!`.
```

### 衝擊

用戶執行 `deepseek` 後實際上是在跑一個舊版的 `deepseek-tui`，修了 bug 但沒生效、改了 UX 但沒看到 — **這對初次體驗是毀滅性的**。

---

## 🟡 根因四：單體式 Codebase 拖慢迭代與除錯

### 4.1 巨型函數

| 函數 | 檔案:行 | 行數 |
|---|---|---|
| `run_event_loop` | `tui/ui.rs:793-3837` | **3,045** |
| `main` | `main.rs:671-918` | 248 |
| `run_interactive` | `main.rs:4262-4368` | 107 |
| `dispatch_user_message` | `tui/ui.rs:3838-3991` | 155 |

```rust
// tui/ui.rs:792
#[allow(clippy::too_many_lines)]  // 容忍 3,045 行
async fn run_event_loop(
    terminal: &mut AppTerminal, app: &mut App, config: &mut Config,
    mut engine_handle: EngineHandle, task_manager: SharedTaskManager,
    event_broker: &EventBroker, // ...
```

### 4.2 巨型結構體

`App` struct 有 **150 個 pub 欄位**，橫跨 381 行（`app.rs:759-1139`）：

```rust
// tui/app.rs:759
#[allow(clippy::struct_excessive_bools)]
pub struct App {
    pub history: Vec<HistoryCell>,
    pub api_messages: Vec<Message>,
    pub is_loading: bool,
    pub offline_mode: bool,
    pub auto_compact: bool,
    // ... 145 個更多欄位
}
```

### 4.3 單體鍵盤分發

主鍵盤分發是單一 `match key.code` block（`ui.rs:2610-3837`，~1,228 行），依賴 match arm 順序而非顯式優先級：

```rust
// tui/ui.rs:2610
match key.code {
    KeyCode::Enter if app.input.is_empty() && ... => { continue; }
    KeyCode::Char('l') => { ... }
    KeyCode::Char('d') => { ... }
    // ... ~60 個 arms，零衝突檢測
}
```

### 4.4 Clippy 壓制統計

**39 處** `#[allow(clippy::*)]`，分佈於 15+ 檔案：

| lint | 次數 | 典型位置 |
|---|---|---|
| `too_many_arguments` | 14 | 工具註冊、shell、subagent、capacity_flow |
| `too_many_lines` | 7 | ui.rs、app.rs、tool_routing.rs |
| `struct_excessive_bools` | 2 | app.rs |
| `print_stdout` | 4 | tui/ui.rs, core/capacity.rs |
| 其他 | 12 | large_enum_variant, cast_precision_loss 等 |

### 4.5 Top 10 最長文件

| # | 行數 | 檔案 |
|---|---|---|
| 1 | 7,534 | `crates/tui/src/tui/ui.rs` |
| 2 | 6,423 | `crates/tui/src/main.rs` |
| 3 | 6,076 | `crates/tui/src/config.rs` |
| 4 | 5,845 | `crates/tui/src/tui/ui/tests.rs` |
| 5 | 5,743 | `crates/tui/src/tui/app.rs` |
| 6 | 5,352 | `crates/tui/src/runtime_threads.rs` |
| 7 | 4,764 | `crates/tui/src/tui/history.rs` |
| 8 | 4,561 | `crates/tui/src/tools/subagent/mod.rs` |
| 9 | 4,021 | `crates/tui/src/mcp.rs` |
| 10 | 3,767 | `crates/tui/src/runtime_api.rs` |

### 衝擊

3,045 行的 event loop + 1,228 行的 key dispatch = **修改任何快捷鍵或 UI 行為都需要通讀整個函數才能確保不破壞其他功能**。這直接導致迭代速度慢、bug 修復週期長。

---

## 🟡 根因五：TUI 層級的不必要性能開銷

### 5.1 每次 turn 的 Git Snapshot

```rust
// core/turn.rs:139-182
pub fn pre_turn_snapshot(workspace: &Path, turn_seq: u64, cap_bytes: u64) -> Option<String> {
    snapshot_with_label(workspace, &format!("pre-turn:{turn_seq}"), cap_bytes)
}
pub fn post_turn_snapshot(workspace: &Path, turn_seq: u64, cap_bytes: u64) -> Option<String> {
    snapshot_with_label(workspace, &format!("post-turn:{turn_seq}"), cap_bytes)
}
```

觸發於 engine.rs：

```rust
// engine.rs:918-926
if self.config.snapshots_enabled {  // 預設 true!
    let _ = tokio::task::spawn_blocking(move || {
        pre_turn_snapshot(&pre_workspace, pre_seq, pre_cap)
    }).await;
}
```

`snapshot()` 方法執行完整 git 工作流：

```rust
// snapshot/repo.rs:304-411
let add = run_git(&self.git_dir, &self.work_tree, &["add", "-A"])?;
let tree = run_git(&self.git_dir, &self.work_tree, &["write-tree"])?;
let commit = run_git(&self.git_dir, &self.work_tree, &arg_refs)?;
let update = run_git(&self.git_dir, &self.work_tree, &["update-ref", "HEAD", &sha])?;
```

### 5.2 完整 session 序列化

```rust
// session_manager.rs:238-251
let content = serde_json::to_string_pretty(session)?;  // O(messages)
```

Persistence actor 用 latest-wins coalescing（`persistence_actor.rs:106-184`）減少寫入頻率，但**每次寫入的數據量仍隨 session 增長而增加**。

### 5.3 其他背景開銷

- `lsp_hooks.rs:80` — 每個工具成功後觸發 LSP diagnostics
- `capacity_controller` — 雖然預設 disabled（`core/capacity.rs:622-627`），但 opt-in 後每次 turn 做觀察/決策
- `AutomationManager` + `TaskManager` — 獨立 tokio task 定期掃描

### 衝擊

每輪 turn 多做一次 `git add -A` + JSON 序列化的成本在用戶感知上是**明顯的延遲**。Snapshot 預設啟用（`config.rs:517` → `default_snapshots_enabled() -> bool { true }`），多數用戶不知道可以關閉。

---

## 🟢 意外好的部分（非口碑問題來源）

**Error handling** — `error_taxonomy.rs` 定義了完整的 `ErrorCategory`（10 variants）、`ErrorSeverity`（4 級）、`ErrorEnvelope`（含 machine-readable code + human-readable message）。跨子系統的 `From<LlmError>` 和 `From<ToolError>` 實現。

**Retry system** — 完整 exponential backoff + jitter + `Retry-After` header 尊重 + TUI 可見的 `RetryState`（`retry_status.rs:88` → `cell().lock().unwrap_or(RetryState::Idle)`）：

```rust
// llm_client/mod.rs:297-650
pub struct RetryConfig {
    max_retries: u32,      // default 3
    initial_delay: f64,    // 1.0s
    max_delay: f64,        // 60.0s
    exponential_base: f64, // 2.0
    jitter: f64,           // 0.1
}
```

**Config validation** — 每個欄位驗證，錯誤訊息精確到列出所有合法值：

```rust
// config.rs:1326-1419
bail!("Invalid default_text_model '{model}': expected auto or a DeepSeek model ID
    (for example: deepseek-v4-flash, deepseek-v4-pro, deepseek-chat, ...)");
```

**Terminal compatibility** — Kitty protocol、OSC 8/9/99/777、tmux DCS passthrough、bracketed paste、DEC 2026 sync、mouse capture。

---

## 綜合判斷

```
                  ┌─────────────────────────────────────┐
                  │    Session 隨時間退化直到崩潰         │ ← 最致命（立即棄用）
                  │    （auto_compact 預設關閉）          │
                  └──────────┬──────────────────────────┘
                             │
                  ┌──────────▼──────────────────────────┐
                  │    Scope creep → 功能太多、配置複雜    │ ← 新用戶留不住
                  │    （11 providers、25+ tools、       │
                  │     3 modes、sub-agent/RLM/LSP/...）│
                  └──────────┬──────────────────────────┘
                             │
                  ┌──────────▼──────────────────────────┐
                  │    雙 binary → 安裝更新混亂           │ ← 修了像沒修
                  │    （無版本相容性檢查）                │
                  └──────────┬──────────────────────────┘
                             │
                  ┌──────────▼──────────────────────────┐
                  │    單體架構 → 迭代慢、除錯難           │ ← 品質進步慢
                  │    （3,045 行 event loop、           │
                  │     1,228 行 key dispatch）           │
                  └──────────┬──────────────────────────┘
                             │
                  ┌──────────▼──────────────────────────┐
                  │    性能開銷 → 體感卡                  │
                  │    （git snapshot + JSON 序列化）     │
                  └─────────────────────────────────────┘
```

**一句話**：這是一個擁有專業級基礎設施（error taxonomy、retry system、config validation）的 prototype — infrastructure 成熟，但 product decisions 不成熟。Auto-compaction 為了節省 API 費用（prefix cache 90% discount）而預設關閉，導致 session 註定退化，是其中最致命的一個 tradeoff。雙 binary 無版本檢查次之。而 272 個源文件 / 209K 行 / 39 處 clippy 壓制表現出**功能不斷堆疊但缺乏架構治理**，使得即使社區提出 issue，修復也難以快速落地。
