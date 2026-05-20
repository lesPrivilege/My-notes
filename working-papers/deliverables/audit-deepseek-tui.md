# Source-Code Audit: `Hmbown/DeepSeek-TUI` v0.8.39

Audit date: 2026-05-18
Target: `/Users/lesprivilege/Projects/DeepSeek-TUI` (14-crate Rust workspace, edition 2024, Rust 1.88+)
Scope: Fact-check 10 specific claims from a submission document against actual source code, README, docs, and tests.

---

## 1. Fully Supported Claims

### Q1 — Product positioning: "Terminal coding agent for DeepSeek V4," written in Rust

**Verdict: Fully supported.**

- `README.md` line 1: "Terminal coding agent for DeepSeek V4" is the exact headline.
- `Cargo.toml` confirms Rust edition 2024, version 0.8.39, minimum Rust 1.88+, 14 workspace members.
- Two binaries: `deepseek` (CLI dispatcher in `crates/cli`) and `deepseek-tui` (TUI runtime in `crates/tui`).
- Core architecture: ratatui TUI -> async engine -> OpenAI-compatible streaming client.

---

### Q2 — Three interaction modes: Plan, Agent, YOLO

**Verdict: Fully supported.**

- `docs/MODES.md` (94 lines) documents all three modes in detail.
- Plan mode: read-only, shell and patch tools disabled.
- Agent mode: interactive with user approval on tool calls.
- YOLO mode: auto-approves all tool calls.
- Tab cycling: Plan -> Agent -> YOLO -> Plan (line 10-15).
- Mode state machine is wired through the engine's turn loop.

---

### Q3 — Auto mode with flash routing call for model/thinking selection

**Verdict: Fully supported.**

- `crates/tui/src/tui/auto_router.rs` (169 lines): `resolve_auto_model_selection()` builds recent context (6 rows, 900 chars each) and calls `commands::resolve_auto_route_with_flash()`.
- `crates/tui/src/auto_reasoning.rs` (192 lines): keyword-based reasoning-effort selection for auto mode. `HIGH_EFFORT_KEYWORDS` (lines 47-63) includes EN/ZH/JA debug/error terms. `LOW_EFFORT_KEYWORDS` (lines 67-73) includes search/lookup terms.
- Auto mode explicitly does a flash routing call to determine model + thinking effort per turn.

---

### Q4 — Session persistence, durable task queue, workspace-snapshot rollback

**Verdict: Fully supported.**

- **Session persistence:** `crates/tui/src/session_manager.rs` (1856 lines): serde_json to `~/.deepseek/sessions/`. Schema versioning (CURRENT_SESSION_SCHEMA_VERSION=1). MAX_SESSIONS=50, MAX_PERSISTED_MESSAGES=500. Full save/load/delete/list/prune/checkpoint operations.
- **Durable task queue:** `crates/tui/src/task_manager.rs` (1914 lines): serde_json to `~/.deepseek/tasks/`. TaskRecord lifecycle: queued -> running -> completed/failed/canceled. Crash recovery at lines 1490-1558: running tasks are re-queued on restart. Survives process restarts.
- **Snapshot rollback:** `crates/tui/src/snapshot/repo.rs` (1506 lines): side-git repo in `~/.deepseek/snapshots/`. Pre-turn and post-turn snapshots via `git add -A`. 500 MB total cap, 50 snapshot max, 7-day default retention. Restore via `git checkout <sha> -- :/`. Uses `--git-dir --work-tree` to avoid touching the user's own `.git`.
- All three use serde_json serialization.

---

### Q6 — Post-edit LSP diagnostics injection (after edit_file, write_file, apply_patch)

**Verdict: Fully supported.**

- `crates/tui/src/core/engine/lsp_hooks.rs` (123 lines): `Engine::run_post_edit_lsp_hook()` at lines 80-103 runs after tool execution. `edited_paths_for_tool()` (lines 16-49) handles exactly three tool types: `edit_file`, `write_file`, `apply_patch`. `flush_pending_lsp_diagnostics()` (lines 110-121) injects diagnostics as a synthetic user message before the next API request.
- `crates/tui/src/lsp/mod.rs` (536 lines): `LspManager` with lazy per-language spawning. Diagnostics poll timeout: 5000ms default. `max_diagnostics_per_file`: 20 default. `include_warnings`: false by default.
- `crates/tui/src/lsp/registry.rs` (147 lines): 5 confirmed LSP servers — rust-analyzer, gopls, pyright-langserver, typescript-language-server, clangd.
- `crates/tui/src/lsp/client.rs`: StdioLspTransport with didOpen/didChange/publishDiagnostics lifecycle.
- Failure is non-blocking: missing binary, crash, or timeout all degrade gracefully to "no diagnostics this turn."

---

### Q7 — Sub-agent completion events via `<deepseek:subagent.done>` sentinel

**Verdict: Fully supported.**

- `crates/tui/src/tools/subagent/mod.rs`: Implementation at line 3264-3292. Format: `<deepseek:subagent.done>{JSON_payload}</deepseek:subagent.done>`.
- `crates/tui/src/tools/subagent/mailbox.rs` (479 lines): MailboxMessage enum with full lifecycle events: Started, Progress, ToolCallStarted, ToolCallCompleted, ChildSpawned, Completed, Failed, Cancelled, TokenUsage.
- `crates/tui/src/prompts/base.md` lines 204-219: Documents the sentinel format and processing instructions for the model.
- Found in 15+ locations across the codebase (implementation, tests, prompts, integration code).

---

### Q9 — HTTP/SSE runtime API (`deepseek serve --http`)

**Verdict: Fully supported.**

- `crates/tui/src/runtime_api.rs` (3767 lines): Full axum-based HTTP/SSE server implementation.
- `run_http_server()` starts the server with configurable host/port (default: `127.0.0.1:7878`).
- Routes confirmed: `/v1/stream` (POST, SSE streaming), `/v1/threads/{id}/events` (GET), `/v1/apps/mcp/servers` (GET).
- README.md line 321: `deepseek serve --http` documented. Line 92 lists "HTTP/SSE runtime API" as a key feature.
- User config: optional HTPASSWD auth, CORS support via tower-http.
- Docs: `docs/RUNTIME_API.md` documents endpoints.

---

### Q10 — Multi-provider support with OpenAI-compatible API

**Verdict: Fully supported.**

- `crates/tui/src/config.rs` line 67: `ApiProvider` enum with 10 members:
  - `Deepseek`, `DeepseekCN` (native), `NvidiaNim`, `Openai`, `Atlascloud`, `Openrouter`, `Novita`, `Fireworks`, `Sglang`, `Vllm`, `Ollama`.
- README.md lines 251-287: Provider setup examples for each, including provider-specific flags and env vars.
- README.md line 414: `DEEPSEEK_PROVIDER` env var lists all 10 provider names.
- `crates/tui/src/client.rs`: Provider-specific request body construction (reasoning effort varies by provider, line 882+). Default model lists per provider at config.rs lines 396-403.
- `crates/tui/src/llm_client/mod.rs` (1097 lines): `LlmClient` trait at lines 53-75 with `provider_name()`, `model()`, `create_message()`, `create_message_stream()`, `health_check()`. RetryConfig with exponential backoff.
- All providers use the same OpenAI-compatible `/v1/chat/completions` endpoint shape.

---

## 2. Partially Supported / Needs Weakening

### Q5 — OS-level sandbox (macOS Seatbelt, Linux Landlock, Windows Job Objects)

**Verdict: Partially supported. Needs weakening on Linux and Windows.**

- **macOS Seatbelt: Fully implemented.**
  - `crates/tui/src/sandbox/seatbelt.rs` (682 lines): Complete implementation. Generates SBPL policy strings (`SEATBELT_BASE_POLICY`), network policy, cargo/npm cache support. Creates sandbox-exec args. Platform detection in `sandbox/mod.rs` correctly selects it on macOS.
  
- **Linux Landlock: Implemented but incomplete for subprocess sandboxing.**
  - `crates/tui/src/sandbox/landlock.rs` (345 lines): Full Landlock syscall implementation with constants for all FS access types. Uses `SYS_landlock_create_ruleset`, `SYS_landlock_add_rule`, `SYS_landlock_restrict_self`.
  - **BUT** critical limitation at lines 277-291 of landlock.rs: "For now, just return the original command without sandboxing" for subprocess. The comment notes this needs a helper binary. This means subprocesses spawned by the agent are NOT sandboxed on Linux.
  
- **Windows Job Objects: Stub.**
  - `crates/tui/src/sandbox/windows.rs` (67 lines): Line 30: `pub fn is_available() -> bool { false }`. Line 4 comment: "DeepSeek TUI does not advertise an in-process Windows sandbox." Line 10: "The first Windows helper slice is process containment only."
  - `sandbox/mod.rs` lines 442-464: Comment confirms "Windows support is currently not advertised by `get_platform_sandbox`."

**Recommended wording change:**
- Original claim likely says "macOS Seatbelt, Linux Landlock, Windows Job Objects" with equal weight.
- Should say: "macOS Seatbelt (fully implemented), Linux Landlock (implemented; subprocess sandboxing is a known gap pending a helper binary), Windows (stub, not advertised)."

---

## 3. Unsupported / Remove

**None of the 10 claims are entirely unsupported.** Every feature exists in code. The only significant qualification is on Q5 (sandbox) as detailed above.

---

## 4. New Observations Worth Adding

### 4.1 — Skill auto-discovery covers 8 paths across 5 tool conventions

The submission should highlight that the skill system (`crates/tui/src/skills/mod.rs`) discovers from 8 directories in precedence order:

1. `<workspace>/.agents/skills` (native)
2. `<workspace>/skills` (flat local)
3. `<workspace>/.opencode/skills`
4. `<workspace>/.claude/skills`
5. `<workspace>/.cursor/skills`
6. `~/.agents/skills` (global)
7. `~/.claude/skills` (ecosystem interop, #902)
8. `~/.deepseek/skills` (default)

This cross-tool interop (OpenCode, Claude Code, Cursor) is a notable architectural decision worth calling out. Tests confirm all paths work (`discover_in_workspace_pulls_skills_from_opencode_dir`, `discover_in_workspace_pulls_skills_from_cursor_dir`).

### 4.2 — Community skill installer with security hardening

`crates/tui/src/skills/install.rs` (1582 lines): Full installer supporting `InstallSource::GitHubRepo` (`github:owner/repo`), `DirectUrl`, and a curated `Registry`. Enforces:
- Path-traversal rejection
- Symlink rejection
- Size caps (5 MiB default)
- Atomic rename installs

### 4.3 — Reasoning-effort dispatch is provider-aware

`client.rs` lines 882-960: The reasoning effort parameter is mapped per-provider. Deepseek/Openrouter/Novita/Sglang support a full effort spectrum (off/low/medium/high/xhigh). Vllm supports off/low/high. OpenAI/Atlascloud/Ollama suppress reasoning effort entirely. This provider-awareness is non-trivial.

### 4.4 — Runtime API includes MCP server listing

`runtime_api.rs`: Beyond basic HTTP/SSE streaming, the runtime exposes `/v1/apps/mcp/servers` for MCP (Model Context Protocol) server discovery. This is a notable extension beyond a simple proxy.

### 4.5 — Edition 2024 and Rust 1.88 minimum

The project targets Rust edition 2024 with MSRV 1.88, which is very recent (early 2026). This enables use of `if-let` chains, `is_some_and`, `let`-chains in expressions, and other modern Rust features visible throughout the codebase.

### 4.6 — Snapshot system uses side-git, not the workspace git

The snapshot/rollback system (`snapshot/repo.rs`) intentionally creates a separate git repository at `~/.deepseek/snapshots/` with its own `.git` directory, using `--git-dir --work-tree` flags. This means workspace rollback works even when the user's project is not in git, and never interferes with the user's own VCS history.

---

## 5. Final Rewrite Notes

### Structural improvements

1. **Lead with architecture, not features.** Start with the 14-crate Rust workspace, two-binary split (CLI dispatcher + TUI runtime), and async engine architecture before listing features.

2. **Explicitly caveat the sandbox section.** Split Q5 into three sub-bullets: "macOS: production-ready (Seatbelt + sandbox-exec)", "Linux: Landlock syscall implemented, subprocess sandboxing pending helper binary", "Windows: stub, not advertised." Do not list all three as equal.

3. **Add the skill auto-discovery cross-tool interop.** The 8-directory discovery pipeline across 5 tool conventions (native, OpenCode, Claude Code, Cursor, global) is a unique architectural feature worth prominent mention.

4. **Add MCP support to the runtime API description.** The `/v1/apps/mcp/servers` endpoint makes the HTTP/SSE runtime more than a simple streaming proxy.

5. **Quantify where possible.** "50 sessions max, 500 messages per session, 500 MB snapshot cap, 50 snapshots, 7-day retention, 20 diagnostics per file, 5000ms LSP timeout, 5 MiB install cap, 8-level skill discovery depth."

### Factual corrections needed in the submission

6. **Q1 (Positioning):** Ensure "DeepSeek V4" is used exactly. The README says "Terminal coding agent for DeepSeek V4" not just "DeepSeek."

7. **Q5 (Sandbox):** Remove any claim that Windows sandbox is functional. Remove any claim that Linux subprocess sandboxing works. Add caveat text.

8. **Q8 (Skills):** The submission likely mentions `.agents/skills` only. Add that `skills_directories()` discovers from 8 paths including `.opencode/skills`, `.claude/skills`, `.cursor/skills`, and global directories.

9. **Q9 (Runtime API):** Mention that port 7878 is the default, configurable via options. Mention the optional HTPASSWD auth and CORS support.

10. **Provider count:** Ensure the text says "10 providers" not a different number. The ApiProvider enum has 10 members (counting DeepseekCN separately from Deepseek).

### Tone and framing

11. **Avoid "fully cross-platform" language** for sandboxing. Linux has real limitations. Windows is stub-only. The codebase itself acknowledges this.

12. **Avoid "all LSP servers run in-process"** type claims. LspTransport uses stdio subprocesses for each language server, spawned lazily on first use per language.

13. **Avoid saying "git-based snapshots"** without explaining it is a side-git repo. The workspace rollback does not use the user's own git history.

14. **The tool calling it "auto-router"** in docs corresponds to `auto_router.rs` in code. The flash routing call is a distinct API call made per-turn in auto mode to select model + thinking configuration.

15. **Mention the `deepseek serve --http` command** explicitly as the entry point for the HTTP/SSE runtime API, not just the TUI mode.
