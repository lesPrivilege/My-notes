# DS4 (DwarfStar 4) Execution Path Audit

**Date:** 2026-05-21  
**Source:** `/Users/lesprivilege/Projects/ds4/ds4_server.c` (15,581 lines) + `ds4.c`, `ds4.h`  
**Scope:** Server-side request processing + KV cache logic. Excludes quantizer (`gguf-tools/`), GPU kernels (`metal/`, `ds4_cuda.cu`, `ds4_metal.m`).

---

## 1. Server Framework

### 1.1 HTTP Layer

No libmicrohttpd. Custom raw socket + poll + pthreads.

| Component | Function | File:Line |
|---|---|---|
| Listen | `listen_on()` | `ds4_server.c:11641` |
| Accept loop | inline in `main()` | `ds4_server.c:12067-12098` |
| Per-connection handler | `client_main()` | `ds4_server.c:11550` |
| HTTP request parser | `read_http_request()` | `ds4_server.c:11435` |
| HTTP response helper | `http_response()` | `ds4_server.c:4697` |
| HTTP error helper | `http_error()` | `ds4_server.c:4723` |
| CORS support | `enable_cors` flag | `ds4_server.c:7644` |

**Architecture:**
- `main()` creates one listening socket (AF_INET, SOCK_STREAM), calls `listen(fd, 128)`.
- Accept loop at `ds4_server.c:12067`: calls `accept(lfd, NULL, NULL)`, for each new connection calls `configure_client_socket()` then spawns a detached `pthread` running `client_main()`.
- `client_main()` calls `read_http_request()` to parse method/path/body, then dispatches to route handlers.
- After dispatching, the client thread enqueues a `job` struct into a linked list (`server.head`/`tail`) and waits on a condition variable (`j->cv`) for the single worker thread.
- The worker thread (`worker_main()`) dequeues jobs and calls `generate_job()`.
- **No middleware layer**: no auth, no rate-limiting, no request-level CORS pre-check beyond the OPTIONS handler.

### 1.2 Signal Handling

| Signal | Handler | File:Line |
|---|---|---|
| SIGINT, SIGTERM | `stop_signal_handler()` | sets `g_stop_requested` |
| SIGPIPE | `SIG_IGN` | `ds4_server.c:11989` |

### 1.3 Client/Worker Model

```
main()                  -- accept loop, spawn client threads (detached)
  └─ client_main()      -- per-connection: parse HTTP, parse JSON, enqueue job, wait
  └─ worker_main()      -- single thread: dequeue jobs, call generate_job(), signal done
```

**Job enqueue/dequeue** (`ds4_server.c:11354-11380`): linked list with `pthread_cond_wait`/`pthread_cond_signal`.

---

## 2. Route Dispatch (Three-Protocol + Legacy)

**Dispatch point:** `client_main()` at `ds4_server.c:11562-11599`

| Method | Path | Handler | Line |
|---|---|---|---|
| GET | `/v1/models` | `send_models()` | 11569 |
| GET | `/v1/models/deepseek-v4-flash` | `send_model()` | 11574 |
| OPTIONS | any | CORS preflight (204) | 11563 |
| POST | `/v1/messages` | `parse_anthropic_request()` | 11583 |
| POST | `/v1/chat/completions` | `parse_chat_request()` | 11586 |
| POST | `/v1/responses` | `parse_responses_request()` | 11589 |
| POST | `/v1/completions` | `parse_completion_request()` | 11592 |

**Fallback:** 404 for unknown paths.

All four POST handlers parse the JSON body into a `request` struct, then `client_main()` checks `request_exceeds_context()` before enqueuing the job. The worker thread's `generate_job()` handles all remaining logic (cache, inference, response).

---

## 3. Request Lifecycle (`POST /v1/chat/completions`)

### 3.1 Route Entry

```
client_main() :11550
  read_http_request() :11435          -- parse method, path, headers, body
  strcmp(hr.method,"POST") &&          -- route match :11586
    strcmp(hr.path,"/v1/chat/completions")
  parse_chat_request() :2607           -- parse JSON body
  request_exceeds_context() :4739      -- guard
  enqueue() :11354                     -- job -> worker queue
  pthread_cond_wait(&j->cv)            -- wait for worker
```

### 3.2 JSON Parse (`parse_chat_request`, line 2607)

1. Initialize `request` via `request_init()` (line 743) with `REQ_CHAT` kind, `API_OPENAI` style (default).
2. Iterate JSON object keys:
   - `"messages"` -> `parse_messages()`: extracts `chat_msg` array (role, content, reasoning, tool_call_id, tool_calls with raw_dsml)
   - `"tools"` -> `parse_tools_value()`: extracts tool schemas JSON string + key-order preferences
   - `"tool_choice"` -> detect `"none"`
   - `"model"`, `"max_tokens"/"max_completion_tokens"`, `"temperature"`, `"top_p"`, `"min_p"`, `"top_k"`, `"seed"`
   - `"stream"`, `"stream_options"` (for `stream_include_usage`)
   - `"thinking"`, `"reasoning_effort"`, `"think"` -> `ds4_think_mode`
   - `"stop"` -> `parse_stop()`
3. After parsing: tool_choice_none disables `has_tools`.
4. `kv_cache_restore_tool_memory_for_messages()` (line 2758): scan through all `.kv` files, load tool-id-to-DSML block mappings for any tool_call_ids found in messages.
5. `tool_memory_attach_to_messages()` (line 2759): for each tool_calls in messages, look up call IDs in `tool_memory` (rax tree); if all IDs from a group map to the same block, replace `raw_dsml` with the exact sampled DSML for KV-prefix alignment.
6. `render_chat_prompt_text()` (line 2763): render the full prompt text with DSML markers, `<｜User｜>`, `<｜Assistant｜>`, tool schemas, thinking markers.
7. `ds4_tokenize_rendered_chat()` (line 2765): BPE-tokenize the rendered text into `r->prompt`.

**Error path:** returns `false` + error message -> `client_main` sends HTTP 400.

### 3.3 KV Cache Lookup — Cache Hierarchy

**Entry point:** `generate_job()` at `ds4_server.c:10505`

The function implements a tiered cache lookup, trying each strategy in order:

**Tier 1: Responses visible-prefix (live)**  
`responses_live_visible_prefix_prompt()` (`ds4_server.c:9685`)
- Only for `api == API_RESPONSES` (skipped for chat/completions).
- For chat/completions, this returns 0 immediately.

**Tier 2: Responses tool-output continuation (live)**  
`responses_live_continuation_prompt()` (`ds4_server.c:9626`)
- Only for `api == API_RESPONSES`, skipped for chat/completions.

**Tier 3: Anthropic tool-output continuation (live)**  
`anthropic_live_continuation_prompt()` (`ds4_server.c:9652`)
- Only for `api == API_ANTHROPIC`, skipped for chat/completions.

**Tier 4: Memory token-prefix match (live)**  
`common == old_pos && prompt.len >= old_pos` (inline, `ds4_server.c:10578`)
- Checks if the live Metal session's token sequence is a prefix of the new prompt, at the exact same token position. This is the core cache hit for short tool-result continuations.

**Tier 5: Thinking visible-prefix (live)**  
`thinking_live_visible_prefix_prompt()` (`ds4_server.c:9727`)
- Checks if the prompt begins with the remembered visible text from a prior thinking turn. Applies to all non-Responses chat APIs.

**Tier 6: Memory text-prefix match (live)**  
`live_text_prefix_prompt()` (`ds4_server.c:9593`)
- Renders live token state back to text, checks if it's a byte-prefix of the new prompt.
- If match: tokenizes only the suffix after the common text.

**Tier 7: Disk cache load**  
`kv_cache_try_load()` -> `kv_cache_try_load_text()` (`ds4_server.c:9464`)
- `kv_cache_find_text_prefix()` (`ds4_server.c:9440`): scans in-memory entry list for entries whose SHA1 matches a prefix of the new prompt.
- `sha1_bytes_hex()` (`ds4_server.c:8382`): computes SHA1 of rendered text bytes.
- Reads `.kv` file, validates header+hash+prefix, calls `ds4_session_load_payload()` to restore graph state from file.
- On success: builds effective prompt from exact loaded tokens + text suffix.
- On failure (corrupt/mismatch): calls `ds4_session_invalidate()`.

### 3.3a Cache Hit: Resume from Cache

When `cached > 0`:
- `effective_prompt` contains only the **new** tokens (suffix after the cache prefix).
- `prompt_for_sync` points to `effective_prompt`.
- The live (or restored) session already has the prefix; `ds4_session_sync()` will evaluate only the new suffix.
- `cache_read_tokens = cached`, `cache_write_tokens = prompt_tokens - cached`.

### 3.3b Cache Miss: Full Prefill

When `cached == 0`:
- If old session was long enough: `kv_cache_store_current(s, "evict")` at line 10615 — persists current session to disk before discarding.
- Full `ds4_session_sync(s->session, prompt_for_sync)` at line 10738/10759.

### 3.3c Error Paths

| Condition | Response | Line |
|---|---|---|
| Bad JSON | 400 "invalid JSON request" | 11603 |
| Context exceeded | 400 with context_length_exceeded | 11607 |
| Server stopping | 503 | 11623 |
| Live state missing (Responses) | 409 | 10567 |
| Live state missing (Anthropic) | 409 | 10574 |
| Prefill fails | 500 with error message | 10745/10765 |
| Stream write fails | silent return (error logged) | 10812+ |

### 3.4 Prefill + Cold Cache Store

After sync, before generation:

1. **Cold cache store** (line 10707-10757): If `cached == 0`, prompt meets `min_tokens`, and `cold_max_tokens > 0`:
   - Find anchor position via `kv_cache_chat_anchor_pos()` (line 8998): last `<｜User｜>` token before first `<｜Assistant｜>` token.
   - Prefill to the anchor point, then `kv_cache_store_live_prefix(s, tokens, store_len, "cold")`.
2. **Full prefill** (line 10759): `ds4_session_sync()` with the complete `prompt_for_sync`.
3. **Live state cleanup** (lines 10770-10772): clear protocol live bindings unless this request continued from them.
4. **Continued store check** (line 10774): `kv_cache_maybe_store_continued()` after prefill completes.
5. **Edge case** (lines 10782-10790): if `cold_store_len == prompt_for_sync->len` (entire prompt was a cold checkpoint), store as "cold" post-sync instead.

### 3.5 Inference

- **Sampling loop** at `ds4_server.c:10872-11284`:
  ```
  while not stopped and completion < max_tokens and ctx not full:
    token = ds4_session_sample()                                        :10893
    if EOS: finish="stop"
    if temp==0 and MTP enabled: ds4_session_eval_speculative_argmax()   :10905
    else: ds4_session_eval()                                            :10918
    ds4_token_text() -> piece                                           :10936
    buf_append(&text, piece)
    thinking_state_feed()
    dsml_decode_tracker_update()                                        :10943
    stop_list_find_from()                                               :10947
    SSE stream updates                                                  :10960-11001
    if tool_end marker seen: finish="tool_calls"; break                 :11078
  ```
- **MTP (Multi-Token Prediction)**: speculative argmax with drafted tokens, controlled by `--mtp` options, disabled via `DS4_MTP_SPEC_DISABLE` env var.
- **DSML decode tracking** (`dsml_decode_tracker`, `ds4_server.c:5338`): tracks whether the current decode is inside a DSML tool-call block. Used to set temperature to 0 for parameter payloads and to trigger the `finish="tool_calls"` break.

### 3.6 Tool Call Generation

#### 3.6.1 DSML Output Path

- Model generates raw text containing DSML markers (`<｜DSML｜tool_calls>`, `<｜DSML｜invoke>`, etc., defined at lines 4182-4189).
- `dsml_decode_tracker_update()` (line 10943) monitors the output stream for DSML markers.
- `observe_tool_markers()` (line 11024) detects `saw_tool_start` / `saw_tool_end`.
- When `saw_tool_end` triggers, the decode loop breaks with `finish = "tool_calls"`.

#### 3.6.2 Post-Decode Parsing

- `parse_generated_message_for_response()` (`ds4_server.c:4602`): extracts content, reasoning, and parsed `tool_calls` from raw generated text.
- On parse failure: if tool call is malformed, returns as assistant text instead of error (line 11138-11148).
- Unfinished tool calls (saw_tool_start but no saw_tool_end): `finish = "error"` with "unterminated tool call" (line 11092-11100).

#### 3.6.3 Tool ID Assignment

```
apply_openai_stream_tool_ids()    -- inherit IDs from stream            :8127
apply_anthropic_stream_tool_ids() -- inherit IDs from stream            :8137
assign_tool_call_ids()            -- assign random IDs for missing      :8114
```

- `random_tool_id()` (`ds4_server.c:483`): generates `call_` or `toolu_` prefixed hex IDs (depending on api_style). Falls back to time+pid mixing if `/dev/urandom` unavailable.
- IDs are guaranteed unique against memory and within the current call group.

#### 3.6.4 tool_memory_remember()

`tool_memory_remember()` (`ds4_server.c:8024`):
- Stores the **exact sampled DSML block** (`calls->raw_dsml`) in a rax tree keyed by each tool call ID.
- Only stores if `disable_exact_dsml_tool_replay` is false and `raw_dsml` exists.
- **Guarded by `s->tool_mu` mutex.**
- Entries are capped by `max_entries` (default 100k) and `max_bytes` (default 512 MB), with LRU eviction.

#### 3.6.5 Next-Round Replay Path

On the next request containing tool_call_ids:
1. `kv_cache_restore_tool_memory_for_messages()` (`ds4_server.c:8770`): scans `.kv` files on disk for tool maps, loads DSML blocks referenced by call IDs.
2. `tool_memory_attach_to_messages()` (`ds4_server.c:8050`): for each tool_calls group in messages, looks up all IDs in the rax tree. If all IDs from one call group are found in the same block, replaces `calls->raw_dsml` with the exact DSML.
3. If all IDs match and belong to the same DSML block: `calls->raw_dsml` is set to the exact sampled DSML → `render_chat_prompt_text()` calls `append_dsml_tool_calls_text()` which prefers raw_dsml over canonical JSON (line 2216: `if (calls->raw_dsml && calls->raw_dsml[0])`).

#### 3.6.6 Fallback to Canonical Rendering

When `tool_memory_attach_to_messages()` cannot find all IDs or IDs span different blocks:
- `stats.missing_ids` incremented, `stats.canonical` incremented.
- `calls->raw_dsml` remains NULL → `append_dsml_tool_calls_text()` renders tool calls via canonical JSON (`append_dsml_arguments_from_json()` at line 2226).
- This produces `DS4_PROMPT_TEXT_DIFFERS` classification, triggering `canonicalize_tool_checkpoint()` (`ds4_server.c:10346`) which rebuilds the session to the canonical text.

**Condition to canonicalize**: `should_canonicalize_tool_checkpoint()` (`ds4_server.c:10483`) returns true when `tool_replay.canonical > 0` (i.e., any tool call group used canonical rendering instead of exact DSML replay). **Skipped for Responses API** (line 11200-11201: `if (j->req.kind == REQ_CHAT && parsed_calls.len && j->req.api != API_RESPONSES`).

### 3.7 Response Streaming / Final Output

**Decision point** at `ds4_server.c:11224-11284`:

| API | Streaming path | Non-streaming path |
|---|---|---|
| Anthropic | `anthropic_sse_start_live()` :6865 → `anthropic_sse_stream_update()` :7318 → `anthropic_sse_finish_live()` :7473 | `anthropic_final_response()` :6790 |
| OpenAI (live chat) | `openai_stream_start()` :4965 → `openai_sse_stream_update()` :5744 → `openai_sse_finish_live()` :5826 | `final_response()` :6677 |
| Responses (live) | `responses_sse_created()` :5974 → `responses_sse_stream_update()` :6427 → `responses_sse_finish_live()` :6537 | `responses_final_response()` :6603 |
| Structured stream | `sse_chat_finish()` :4879 | — |
| Legacy completions | `sse_chunk()` :4796 + `sse_done()` :4873 | `final_response()` :6677 |

**SSE format (OpenAI streaming):**
```
data: {"id":"chatcmpl-N","object":"chat.completion.chunk",...,"choices":[{"index":0,"delta":{...},"finish_reason":null}]}\n\n
```
Final chunk:
```
data: {"id":"chatcmpl-N","object":"chat.completion.chunk",...,"choices":[],"usage":{...}}\n\n
data: [DONE]\n\n
```

**Final non-streaming JSON:**
```json
{"id":"chatcmpl-N","object":"chat.completion","choices":[{"index":0,"message":{"role":"assistant","content":"...","reasoning_content":"...","tool_calls":[...]},"finish_reason":"stop"}],"usage":{...}}
```

### 3.8 KV Cache Persistence — Four Timing Modes

All writes go through the same core function: `kv_cache_store_live_prefix_text()` (`ds4_server.c:9180`).

**Mechanism:** compute SHA1 of rendered text → write `.kv` file with fixed header + text + KV payload + tool map → atomically rename `.tmp` to `.kv` path. Calls `kv_cache_evict()` after write to enforce budget.

**Persistence model:** write-through (synchronous `fflush` + `rename` before acknowledging the store).

#### Cold (`reason="cold"`)

| Trigger | Location | Condition |
|---|---|---|
| `kv_cache_store_live_prefix()` | `ds4_server.c:10748` | Prefill to anchor point, before full sync |
| `kv_cache_store_live_prefix()` | `ds4_server.c:10783` | Entire prompt fits cold limit |

- Triggered when `cached == 0`, prompt >= `min_tokens`, prompt <= `cold_max_tokens`.
- Anchor: `kv_cache_chat_anchor_pos()` — last `<｜User｜>` token before first `<｜Assistant｜>` token.
- **write-through**: synchronous file write before prefill continues.

#### Continued (`reason="continued"`)

| Trigger | Location | Condition |
|---|---|---|
| `kv_cache_maybe_store_continued()` in prefill | `ds4_server.c:10774` | After full prefill |
| `kv_cache_maybe_store_continued()` during decode | `ds4_server.c:10878` | Every decode step when not in tool call |

- `kv_cache_continued_step()` (`ds4_server.c:9028`): default `continued_interval_tokens = 10000`, aligned to `boundary_align_tokens = 2048` boundary.
- `kv_cache_continued_store_target()` (`ds4_server.c:9039`): stores at exact `live_tokens % step == 0` frontiers, only if `live_tokens > continued_last_store_tokens`.
- `kv_cache_maybe_store_continued()` (`ds4_server.c:9429`): checks target and calls `kv_cache_store_live_prefix(s, tokens, target, "continued")`.
- **write-through**: synchronous write during decode loop (every token step, but only when target matches).
- **Suppressed during cold write**: `kv_cache_suppress_continued_store()` prevents double-write at same frontier.

#### Evict (`reason="evict"`)

| Trigger | Location | Condition |
|---|---|---|
| `kv_cache_store_current(s, "evict")` | `ds4_server.c:10615` | Before loading disk snapshot, old session >= min_tokens |

- Called when `cached == 0` (full cache miss) and the live session is long enough.
- `kv_cache_store_current()` (`ds4_server.c:9367`): persists full session. If a Responses visible-text or thinking-visible live state exists at the same token frontier, stores with that visible text as cache key (ext flag). Otherwise stores via token-text key.
- **write-through**: synchronous write before the disk load.

#### Shutdown (`reason="shutdown"`)

| Trigger | Location | Condition |
|---|---|---|
| `kv_cache_store_current(s, "shutdown")` | `ds4_server.c:12119` | After accept loop exits, before `server_close_resources()` |

- Only if session has >= `min_tokens`.
- Same `kv_cache_store_current()` as evict, writes full session.
- **write-through**: synchronous write during shutdown drain (`main()` line 12099-12122).

#### Eviction Algorithm

`kv_cache_evict()` (`ds4_server.c:8849`):
- Refreshes entry list from disk (`kv_cache_refresh()`, line 8735).
- Computes eviction score per entry: `(effective_hits + 1.0) * tokens / file_size`.
- `effective_hits` decays exponentially over `KV_CACHE_HIT_HALF_LIFE_SECONDS` (6 hours).
- Deletes lowest-score entries until total file size <= budget.
- Protected SHA (just-written file) gets `DBL_MAX` score to prevent immediate eviction.

---

## 4. Three-Protocol Routing

### 4.1 Common Architecture

All three protocols share the same internal pipeline after parse:

```
JSON body → parse_*_request() → chat_msgs → tool_memory_attach() → render_chat_prompt_text() → ds4_tokenize_rendered_chat() → generate_job()
```

The `request` struct (ds4_server.c:581-630) carries both `api_style` and `req_kind`:

```c
typedef enum { API_OPENAI, API_ANTHROPIC, API_RESPONSES } api_style;
typedef enum { REQ_CHAT, REQ_COMPLETION } req_kind;
```

### 4.2 Parse Differences

| Aspect | `/v1/chat/completions` | `/v1/messages` | `/v1/responses` |
|---|---|---|---|
| Parse function | `parse_chat_request()` :2607 | `parse_anthropic_request()` :2777 | `parse_responses_request()` :3666 |
| Input field | `messages[]` | `messages[]` + `system` | `input[]` + `instructions` |
| Tool schema | `tools` field | `tools` field | `tools` field + inline hosted-tool schemas |
| Thinking | `thinking` / `reasoning_effort` / `think` | `thinking` / `reasoning_effort` / `output_config` | `reasoning.effort` / `reasoning.summary` |
| Stream options | `stream` + `stream_options.include_usage` | `stream` | `stream` |
| Max tokens | `max_tokens` / `max_completion_tokens` | `max_tokens` | `max_output_tokens` / `max_tokens` |
| Tool choice | `"none"` disables tools | `{"type":"none"}` disables | `"none"` / `"auto"` (others rejected) |
| Continuation | N/A | `anthropic_validate_tool_results()` checks tool IDs | `responses_validate_tool_outputs()` checks tool IDs |
| Live binding setup | N/A | `anthropic_prepare_live_continuation()` :2578 | `responses_prepare_live_continuation()` :2493 |

### 4.3 Unified Intermediate Representation

All three protocols produce the same `chat_msgs` array and use the shared `render_chat_prompt_text()` function (`ds4_server.c:2258`). Key differences:

- **Anthropic**: `system` field is converted to a synthetic system-role message prepended to `msgs` (line 2944-2949). Tool result validation via `anthropic_validate_tool_results()`.
- **Responses**: `instructions` is prepended as system message (line 3866-3877). Tool schemas from `tools` field are combined with inline hosted-tool schemas from input items. `previous_response_id`/`conversation` with non-null values are rejected (line 3832-3843).
- **Chat/completions**: no system message prepending, direct `messages` field.

### 4.4 Response Format Differences

| Aspect | OpenAI | Anthropic | Responses |
|---|---|---|---|
| Non-streaming | `final_response()` :6677 | `anthropic_final_response()` :6790 | `responses_final_response()` :6603 |
| Streaming start | `sse_chunk(role)` → `openai_stream_start()` | `anthropic_sse_start_live()` :6865 | `responses_sse_created()` :5974 |
| Streaming body | `openai_sse_stream_update()` :5744 | `anthropic_sse_stream_update()` :7318 | `responses_sse_stream_update()` :6427 |
| Streaming finish | `openai_sse_finish_live()` :5826 | `anthropic_sse_finish_live()` :7473 | `responses_sse_finish_live()` :6537 |
| Object type | `chat.completion` / `chat.completion.chunk` | `message` | `response` |
| Tool field | `tool_calls[]` in message | `content[].type="tool_use"` | `output[].type="function_call"` |
| Usage format | `{prompt_tokens, completion_tokens, prompt_tokens_details}` | `{input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens}` | `{input_tokens, output_tokens}` (less detail) |

### 4.5 DSML Processing Differences

**Exact DSML replay** (`tool_memory_attach_to_messages`):
- All three protocols benefit from exact DSML replay for KV cache alignment.
- The `raw_dsml` field on `tool_calls` is populated by `tool_memory_attach_to_messages()` for all APIs.

**Canonical checkpoint rebuild:**
- **Chat/completions** (line 11200-11212): if `should_canonicalize_tool_checkpoint()` returns true (some tools had canonical rendering), calls `canonicalize_tool_checkpoint()` to rebuild the session. This is the full rebuild path.
- **Responses** (line 11200-11201): explicitly **skips** canonicalization because `previous_response_id` / live state binds the next turn. The live KV state is preserved as-is.
- **Anthropic** (line 11214): if `parsed_calls.len > 0` and not Responses, clears thinking_live state.

**Live state management after generation:**
- **Responses** (line 11168-11188): after successful generation with tool calls, calls `responses_live_remember()` which stores visible transcript + call IDs + token frontier. This is used by `responses_live_visible_prefix_prompt()` and `responses_live_continuation_prompt()` on the next request.
- **Anthropic** (line 11190-11198): calls `anthropic_live_remember()` which stores only call IDs + token frontier (no visible text). Used by `anthropic_live_continuation_prompt()`.
- **Chat/completions** (line 11221-11222): calls `thinking_live_clear()` — no live state preservation.

---

## 5. Verification: Grep-Confirmed Function Line Numbers

```bash
# === Server Framework ===
grep -n "listen_on\|accept\|client_main\|read_http_request\|worker_main\|enqueue\|dequeue" ds4_server.c | head -20
# 11641: listen_on
# 11550: client_main
# 11435: read_http_request
# 11382: worker_main
# 11354: enqueue
# 11367: dequeue

# === Route Dispatch ===
grep -n '"/v1/chat/completions\|"/v1/messages\|"/v1/responses\|"/v1/completions' ds4_server.c
# 11583: "/v1/messages"
# 11586: "/v1/chat/completions"
# 11589: "/v1/responses"

# === Parse Functions ===
grep -n "^static bool parse_chat_request\|^static bool parse_anthropic_request\|^static bool parse_responses_request" ds4_server.c
# 2607: parse_chat_request
# 2777: parse_anthropic_request
# 3666: parse_responses_request

# === KV Cache Lifecycle ===
grep -n "^static int kv_cache_try_load_text\|^static int kv_cache_find_text_prefix\|^static void kv_cache_maybe_store_continued\|^static void kv_cache_store_current\|^static bool kv_cache_store_live_prefix_text\|^static void kv_cache_evict\|^static void kv_cache_close\|^static bool kv_cache_open" ds4_server.c
# 9464: kv_cache_try_load_text
# 9440: kv_cache_find_text_prefix
# 9429: kv_cache_maybe_store_continued
# 9367: kv_cache_store_current
# 9180: kv_cache_store_live_prefix_text
# 8849: kv_cache_evict
# 8930: kv_cache_close
# 8901: kv_cache_open

# === SHA1 ===
grep -n "^static void sha1_bytes_hex" ds4_server.c
# 8382: sha1_bytes_hex

# === Tool Memory ===
grep -n "^static void tool_memory_remember\|^static void tool_memory_attach_to_messages\|^static bool tool_memory_has_id\|^static void assign_tool_call_ids\|^static void random_tool_id" ds4_server.c
# 8024: tool_memory_remember
# 8050: tool_memory_attach_to_messages
# 8005: tool_memory_has_id
# 8114: assign_tool_call_ids
# 483: random_tool_id

# === Response Formatting ===
grep -n "^static bool sse_headers\|^static bool sse_chunk\|^static bool sse_done\|^static bool final_response\|^static bool anthropic_final_response\|^static bool responses_final_response" ds4_server.c
# 4783: sse_headers
# 4796: sse_chunk
# 4873: sse_done
# 6677: final_response
# 6790: anthropic_final_response
# 6603: responses_final_response

# === Live State Management ===
grep -n "^static void responses_live_remember\|^static void anthropic_live_remember\|^static void responses_live_clear\|^static void anthropic_live_clear\|^static void thinking_live_clear\|^static int responses_live_visible_prefix_prompt\|^static int responses_live_continuation_prompt\|^static int anthropic_live_continuation_prompt\|^static int live_text_prefix_prompt\|^static int thinking_live_visible_prefix_prompt" ds4_server.c
# 7916: responses_live_remember
# 7933: anthropic_live_remember
# 7945: responses_live_clear
# 7952: anthropic_live_clear
# 7898: thinking_live_clear
# 9685: responses_live_visible_prefix_prompt
# 9626: responses_live_continuation_prompt
# 9652: anthropic_live_continuation_prompt
# 9593: live_text_prefix_prompt
# 9727: thinking_live_visible_prefix_prompt

# === Generate Job ===
grep -n "^static void generate_job" ds4_server.c
# 10505: generate_job

# === Render & Tokenize ===
grep -n "^static char \*render_chat_prompt_text" ds4_server.c
# 2258: render_chat_prompt_text
```

---

## 6. Summary: Architecture Diagram

```
main() :11988
  ├─ listen_on() :11641                   -- socket, bind, listen
  ├─ worker_main() :11382                 -- single inference thread (dequeue loop)
  │    └─ generate_job() :10505
  │         ├─ Cache lookup (7 tiers)     -- live text/visible/disk
  │         ├─ kv_cache_store_current("evict")  -- persist old session before load
  │         ├─ kv_cache_try_load() disk   -- restore from .kv file
  │         ├─ ds4_session_sync() prefill -- eval all tokens
  │         ├─ kv_cache_store_live_prefix("cold")  -- store prefix checkpoint
  │         ├─ kv_cache_maybe_store_continued()    -- store interval checkpoint
  │         ├─ Decode loop                -- sample → eval → stream
  │         │    ├─ kv_cache_maybe_store_continued()  -- per-step continued store
  │         │    ├─ SSE streaming         -- api-specific format
  │         │    └─ DSML tracker          -- tool call detection
  │         ├─ tool_memory_remember()     -- store exact DSML for replay
  │         ├─ Live state remember        -- per API (responses/anthropic)
  │         └─ SSE finish / final response
  └─ accept loop :12067
       └─ client_main() :11550            -- per connection, detached thread
            ├─ read_http_request() :11435
            ├─ Route dispatch             -- method+path match
            ├─ parse_*_request()
            │    ├─ JSON parse
            │    ├─ kv_cache_restore_tool_memory_for_messages()  -- disk tool maps
            │    ├─ tool_memory_attach_to_messages()             -- exact DSML replay
            │    ├─ render_chat_prompt_text()
            │    └─ ds4_tokenize_rendered_chat()
            ├─ enqueue() + wait
            └─ signal done → close
```

The KV cache filesystem layout:
```
<kv-disk-dir>/
  ├── <sha1(rendered_text)>.kv     -- 1 file per checkpoint
  ├── .kv_cache_dir                 -- touch marker
  └── (entries reloaded from dir scan on each operate)
```
