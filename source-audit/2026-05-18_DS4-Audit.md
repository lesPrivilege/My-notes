# ds4 Audit

Audit of the local clone at `/Users/lesprivilege/Projects/ds4/` against
a submission document listing key claims.  All citations reference files and
line numbers from this repo (commit: as of the repo snapshot, no upstream
changes factored in).

---

## Supported Claims

- **Claim:** ds4 is a self-contained C inference engine (not a wrapper, not a GGUF
  runner).

  **Evidence:** `README.md` lines 3-6: "DwarfStar 4 is a small native inference
  engine specific for DeepSeek V4 Flash. It is intentionally narrow: not a generic
  GGUF runner, not a wrapper around another runtime: it is completely
  self-contained."  Also `AGENT.md` line 3-4: "`ds4.c` is a DeepSeek V4 Flash
  specific inference engine. It is not a generic GGUF runner."  The Makefile
  (`/Users/lesprivilege/Projects/ds4/Makefile`) links only against system libraries
  (Foundation, Metal, pthreads, or cudart/cublas) -- no GGML shared library,
  no third-party inference runtime.  The gguf-tools quantizer has a
  deliberately narrow implementation of only the needed quantization types and
  does not link GGML either (`gguf-tools/README.md` lines 24-25: "The quantizer is
  plain C and does not link GGML").

  **Suggested wording:** no change needed; this is solid.

- **Claim:** KV cache as first-class disk citizen with SHA1-keyed .kv files for
  cross-session persistence.

  **Evidence:**
  - `README.md` line 40: "**The KV cache is actually a first-class disk citizen**."
  - `README.md` lines 576-753: Full subsystem documentation including file layout,
    header format (48-byte "KVC" header), four save reasons (cold/continued/
    evict/shutdown), and the optional tool-id map section.
  - `ds4_server.c` lines 8151-8191: Block comment describing the KV cache design.
  - `ds4_server.c` lines 8282-8389: Complete SHA1 implementation (sha1_ctx,
    sha1_init, sha1_update, sha1_final, sha1_bytes_hex) -- every function.
  - `ds4_server.c` line 9263: `sha1_bytes_hex(text, text_len, sha);` generates
    the cache key from rendered text bytes.
  - `ds4_server.c` line 8442: `kv_path_for_sha(kc, sha)` constructs file path
    `<sha>.kv` from the 40-hex-char SHA1.
  - `ds4_server.c` line 9240-9264: Full store path showing hash generation and
    path construction.
  - `ds4_server.c` lines 9445-9462: `kv_cache_find_text_prefix()` which iterates
    cache directory entries and matches by SHA1.
  - `ds4.h` lines 191-198: Public API for disk KV payload helpers
    (`ds4_session_save_payload`, `ds4_session_load_payload`,
    `ds4_session_save_snapshot`, `ds4_session_load_snapshot`).
  - The tool-id map inside .kv files preserves exact DSML blocks across server
    restarts (`ds4_server.c` lines 8187-8190, 8500-8700).

  **Suggested wording:** "KV cache as first-class disk citizen with SHA1-keyed .kv
  files" is accurate. Add that files use `read`/`write` I/O (not mmap) to avoid
  piling VM mappings atop the already-large GGUF mapping.

- **Claim:** Supports OpenAI /v1/chat/completions, /v1/responses, AND Anthropic
  /v1/messages simultaneously.

  **Evidence:**
  - `README.md` lines 308-315: Lists all supported endpoints.
  - `ds4_server.c` lines 11568-11598: Exact route dispatch:
    - `GET /v1/models` (line 11568)
    - `GET /v1/models/deepseek-v4-flash` (line 11573)
    - `POST /v1/messages` (line 11583) -- Anthropic-style
    - `POST /v1/chat/completions` (line 11586) -- OpenAI-style
    - `POST /v1/responses` (line 11589) -- OpenAI Responses / Codex CLI
    - `POST /v1/completions` (line 11592) -- OpenAI completions
  - `misc/ANTHROPIC_LIVE_CONTINUATION.md`: Design doc for Anthropic tool_use
    live continuation.
  - `misc/RESPONSE_API.md`: Full design doc for /v1/responses live continuation,
    including multi-client Codex/Pi/curl session switching protocol.

  **Suggested wording:** All three main protocols are implemented. The README
  also lists `/v1/completions` as a bonus endpoint.

- **Claim:** Exact DSML byte-level tool-call replay (exact bytes stored and
  replayed, not re-normalized).

  **Evidence:**
  - `README.md` lines 357-388: Tool call handling description.
  - `ds4_server.c` lines 7671-7683: "Tool Call Text Memory" block comment:
    "remember the exact DSML block sampled by the model under a random id."
  - `ds4_server.c` lines 8005-8008: `tool_memory_has_id()` -- checks if a tool
    call id is known.
  - `ds4_server.c` lines 8024-8033: `tool_memory_remember()` -- stores the
    complete raw DSML block (`calls->raw_dsml`) under the tool call id.
  - `ds4_server.c` lines 8050-8099: `tool_memory_attach_to_messages()` -- looks
    up tool ids in history and attaches `raw_dsml` when exact match is found;
    falls back to canonical JSON-to-DSML rendering when not.
  - `ds4_server.c` lines 2214-2218: `append_dsml_tool_calls_text()` -- first
    checks `calls->raw_dsml` and replays it verbatim when set:
    `if (calls->raw_dsml && calls->raw_dsml[0]) { buf_puts(b, calls->raw_dsml); return; }`
  - `ds4_server.c` line 4493-4494: `calls->raw_dsml = xstrndup(...)` captures
    the raw DSML text from model output during parse.
  - `ds4_server.c` line 7643: `bool disable_exact_dsml_tool_replay;` -- the
    feature can be explicitly disabled (but is on by default).
  - `ds4_server.c` line 7685: `DS4_TOOL_MEMORY_DEFAULT_MAX_IDS = 100000` --
    bounded in-memory map.
  - Tool-id maps are also saved inside .kv cache files for restart survival
    (`ds4_server.c` lines 8650-8720).

  **Suggested wording:** The claim is accurate. The phrase "exact bytes stored
  and replayed, not re-normalized" matches the implementation: `raw_dsml` is the
  complete verbatim DSML block the model sampled. Note that canonical
  JSON-to-DSML rendering exists as a backup when the tool id is unknown or the
  feature is disabled.

- **Claim:** 2-bit asymmetric quantization: only routed MoE experts quantized,
  shared experts/projections/routing untouched.

  **Evidence:**
  - `README.md` lines 94-99: "The 2 bit quants use a very asymmetrical
    quantization: only the routed MoE experts are quantized, up/gate at
    `IQ2_XXS`, down at `Q2_K`. They are the majority of all the model space:
    the other components (shared experts, projections, routing) are left
    untouched to guarantee quality."
  - `gguf-tools/README.md` lines 69-86: Q2 generation command showing the
    mix: `--experts iq2_xxs --routed-w2 q2_k --attention-proj q8_0 --shared q8_0
    --output q8_0`.
  - `gguf-tools/deepseek4-quantize.c` lines 984-989: `quant_policy` structure
    with separate fields for `routed_w1`, `routed_w2`, `routed_w3`,
    `attention_proj`, `shared`, `embedding`, `output`, `dense`.
  - `gguf-tools/deepseek4-quantize.c` lines 1025-1050: `policy_type()` function
    that dispatches each tensor to its designated quantization type based on
    tensor name pattern (routed expert vs shared expert vs attention projection).
  - GGUF filenames confirm the mix, e.g.
    `IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix.gguf`.

  **Suggested wording:** The architecture exists as described. However, "left
  untouched" needs qualification: the non-expert components (attention proj,
  shared experts, output) use Q8_0 (8-bit block quantization), not full FP16.
  They are "untouched" only relative to the 2-bit aggression on routed experts.
  See Partially Supported below for the nuance.

- **Claim:** Author acknowledges llama.cpp/GGML inspiration but project is
  self-contained.

  **Evidence:**
  - `README.md` lines 44-55: "Acknowledgements to llama.cpp and GGML" section.
  - `README.md` lines 46-48: "ds4.c does not link against GGML, but it exists
    thanks to the path opened by the llama.cpp project and the kernels,
    quantization formats, GGUF ecosystem, and hard-won engineering knowledge
    developed there."
  - `README.md` lines 51-53: "Some source-level pieces are retained or adapted
    here under the MIT license: GGUF quant layouts and tables, CPU quant/dot
    logic, and certain kernels."
  - `LICENSE` file: Contains both "Copyright (c) 2026 The ds4.c authors" AND
    "Copyright (c) 2023-2026 The ggml authors."
  - `gguf-tools/quants.c` line 9-11: "derived from the MIT-licensed GGML/llama.cpp
    quantizers."
  - `ds4.c` line 13003: references "llama.cpp's legacy imatrix `.dat` format."

  **Suggested wording:** Accurately reflects the repo's own positioning.

- **Claim:** DeepSeek V4 Flash-specific with MOE architecture optimization.

  **Evidence:**
  - `README.md` line 3: "specific for DeepSeek V4 Flash."
  - `README.md` lines 91-93: "This implementation only works with the DeepSeek V4
    Flash GGUFs published for this project."
  - The asymmetrical 2-bit quantization targets routed MoE experts specifically
    (`gguf-tools/deepseek4-quantize.c` lines 1031-1036).
  - The engine does not load arbitrary GGUFs: no general graph runner.

  **Suggested wording:** Accurate.

- **Claim:** Author mentions alpha quality and assistance from GPT 5.5.

  **Evidence:**
  - `README.md` line 39: "This software is developed with **strong assistance from
    GPT 5.5** and with humans leading the ideas, testing, and debugging. We say
    this openly because it shaped how the project was built."
  - `README.md` line 41: "alpha quality code."
  - `README.md` lines 59-61: "The code and GGUF files are to be considered of
    **alpha quality** because inference and model serving is a complicated matter
    and all this exists only for a few days."
  - `README.md` line 38: "If you are not happy with AI-developed code, this
    software is not for you."

  **Suggested wording:** Accurate. This transparency is unusual and worth
  highlighting as it signals the author's candidness about the project maturity.

---

## Partially Supported / Needs Weakening

- **Claim:** "only routed MoE experts quantized, shared experts/projections/routing
  **untouched**"

  **Evidence:** The README says "left untouched" (`README.md` line 99). However,
  the GGUF file names show `AProjQ8-SExpQ8-OutQ8`, which means attention
  projections use Q8_0, shared experts use Q8_0, and output uses Q8_0. The
  quantizer code at `gguf-tools/deepseek4-quantize.c` line 1045-1048 confirms:
  ```c
  if (is_shared_expert(name) && p->shared != DS4Q_TYPE_COUNT) return p->shared;
  ```
  where `--shared q8_0` is the default for published GGUFs. So these components
  ARE quantized, just to a much higher precision (8-bit) than the routed experts
  (2-bit). The official model weights are FP8 for non-expert parts, so converting
  to Q8_0 is a quantization step, not "untouched."

  **Problem:** If a reader takes "untouched" literally (meaning full float or
  the original FP8), this claim is inaccurate. In context of the submission, it
  could be called misleading.

  **Safer wording:** "Non-routed components (shared experts, attention projections,
  output projection) are quantized at a much higher precision (Q8_0 / 8-bit)
  rather than the aggressive 2-bit applied to routed MoE experts. Only the routed
  MoE expert tensors receive 2-bit asymmetric quantization (gate/up: IQ2_XXS,
  down: Q2_K)."

- **Claim:** "96GB RAM M3 Max: prefill 58.52 t/s, generation 26.68 t/s"

  **Evidence:** The README speed table at line 157 says:
  ```
  | MacBook Pro M3 Max, 128 GB | q2 | short | 58.52 t/s | 26.68 t/s |
  ```
  The submission document says "96GB RAM" but the README says **128 GB**. The
  README does not contain any M3 Max row with 96GB. The README line 33 mentions
  "96/128GB of memory" in the general intro, and line 415 mentions users
  running with 96GB, but the benchmark row itself is 128 GB. There is also no
  `m3_max.csv` file in `speed-bench/` -- only `m2_ultra.csv` and `m4_max.csv`
  exist with CSV data. The M3 Max numbers appear only in the README table
  and the `m3_max_ts.svg` graph. The README explicitly says line 150:
  "These are **single-run** Metal CLI numbers."

  **Problem:** Two issues: (a) wrong RAM figure -- 128 GB not 96 GB; (b) these
  are single-run numbers from the README, not a reproducible benchmark CSV.
  Submitting them as hard benchmarks without the "single-run" qualifier is
  misleading.

  **Safer wording:** "On a MacBook Pro M3 Max with 128 GB RAM, the README
  reports single-run q2 performance of 58.52 t/s prefill and 26.68 t/s
  generation (short prompt, `--ctx 32768`, greedy decoding, `-n 256`). These
  are single-run numbers and should not be treated as reproducible benchmarks
  without independent verification."

- **Claim:** "DSML" as "DeepSeek Markup Language"

  **Evidence:** The acronym `DSML` is used pervasively throughout the codebase
  (`README.md` lines 320, 359, 363-370, 380; `ds4_server.c` lines 1999-2011,
  2214-2234, 4493-4494, 7671-7683, etc.). The format is documented at
  https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/encoding/README.md
  (referenced from `README.md` line 359). However, "DeepSeek Markup Language"
  as a full expansion of the DSML acronym **does not appear anywhere** in the
  codebase, README, sub-READMEs, or model card. The acronym is used without
  expansion in all documentation and code.

  **Problem:** If the submission document introduces "DeepSeek Markup Language"
  as the official name, this should be sourced from the upstream DeepSeek
  encoding documentation, not from this repo. The repo only uses the acronym
  "DSML" without expanding it.

  **Safer wording:** "DSML (the DeepSeek markup format, referenced as `DSML`
  throughout the codebase and documented at [HuggingFace encoding README])"
  -- and cite the upstream source, not this repo.

---

## Unsupported / Remove

- **Claim that there are no CSV benchmarks backing the M3 Max numbers.**

  **Search performed:** Looked for `m3_max.csv` in `/Users/lesprivilege/Projects/ds4/speed-bench/`. Found
  `m2_ultra.csv`, `m4_max.csv`, and `m3_max_ts.svg` but no `m3_max.csv`.

  **Reason:** The README speed table claims single-run numbers for M3 Max but no
  corresponding CSV file exists in the speed-bench directory. The `m2_ultra.csv`
  and `m4_max.csv` files prove CSV-format benchmarks exist for other machines.
  The absence of `m3_max.csv` means these numbers cannot be independently
  verified from the repo. This does not make them false, but the submission
  should acknowledge they are single-run README-reported figures.

  **Recommendation:** Add a sentence: "The M3 Max numbers are single-run figures
  reported in the README, not a repeatable benchmark CSV." If the submission
  needs stronger evidence, ask the author to run `ds4-bench` and contribute the
  CSV.

- **The submission says M3 Max with "96GB RAM" but the README says 128 GB.**

  **Search performed:** README line 157: "MacBook Pro M3 Max, **128 GB**".
  Also checked README line 13: "Starting from MacBooks with 96GB of RAM" --
  that's a general system requirement, not the benchmark spec.

  **Reason:** This is a factual error. The README speed table clearly states
  128 GB for the M3 Max. The 96 GB figure appears only as a general minimum
  requirement. Must be corrected to 128 GB.

---

## New Observations Worth Adding

- **Observation:** The `misc/` directory contains detailed design documents
  (`ANTHROPIC_LIVE_CONTINUATION.md`, `RESPONSE_API.md`) that provide unusual
  depth of design rationale, including QA logs, protocol edge cases tested,
  and even negative test results.

  **Evidence:**
  - `/Users/lesprivilege/Projects/ds4/misc/RESPONSE_API.md`: 448 lines of design notes
    including protocol model analysis, official OpenAI docs findings, live
    testing with Codex/pi/opencode clients, and interleaved session switching
    tests.
  - `/Users/lesprivilege/Projects/ds4/misc/ANTHROPIC_LIVE_CONTINUATION.md`: 100 lines
    covering design lessons, the Anthropic contract (tool_use.id binding), and
    a QA checklist.

  **Why it matters for Harness PM:** This is rare for any inference project
  (let alone an alpha) and shows the author is meticulously documenting
  protocol-level correctness. It strengthens the "serious engineering despite
  alpha quality" narrative.

- **Observation:** The SHA1 implementation is hand-rolled, not linked from
  OpenSSL or a system library.

  **Evidence:** `ds4_server.c` lines 8282-8389 implement SHA1 from scratch
  (sha1_ctx, sha1_transform, sha1_init, sha1_update, sha1_final, sha1_bytes_hex).
  There is a unit test `test_sha1_bytes_hex_matches_known_vector()` at line 14778
  that verifies against a known SHA1 vector.

  **Why it matters for Harness PM:** Zero external dependency for the cache
  key. The entire server, including the KV cache subsystem, is self-contained C
  with no external crypto library. This reinforces the "no wrapper, fully
  contained" positioning.

- **Observation:** The MTP (Multi-Token Prediction / speculative decoding) path
  exists but is explicitly marked as experimental.

  **Evidence:** `README.md` lines 130-133: "The current MTP/speculative decoding
  path is still experimental: it is correctness-gated and currently provides at
  most a slight speedup, not a meaningful generation-speed win." Line 132: "still
  experimental."

  **Why it matters:** If the submission mentions speculative decoding, it should
  include the "experimental/not yet meaningful" qualifier.

- **Observation:** The tests include a long-context security regression test
  (`tests/long_context_security_prompt.txt`) alongside the story fact-recall test,
  showing attention to security-relevant correctness.

  **Why it matters:** Useful for positioning as a serious coding-agent inference
  engine.

---

## Final Rewrite Notes

1. **Correct the RAM figure:** Change "MacBook Pro M3 Max 96GB RAM" to "MacBook
   Pro M3 Max 128 GB RAM" to match `README.md` line 157.

2. **Qualify the M3 Max numbers:** Add "single-run" and reference the README
   conditions (`--ctx 32768`, `--nothink`, greedy, `-n 256`). Do not present
   them as reproducible benchmarks unless you also run `ds4-bench`.

3. **Weaken "untouched" to "higher-precision quantized":** Replace "left
   untouched" with "quantized at 8-bit (Q8_0) rather than the 2-bit level
   applied to routed experts." Cite the GGUF file naming convention
   (`AProjQ8-SExpQ8-OutQ8`).

4. **Source the DSML expansion:** If using "DeepSeek Markup Language," cite the
   upstream HuggingFace encoding README
   (https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/encoding/README.md),
   not this repo, since the repo only uses the acronym.

5. **Add the transparency angle:** The README's open acknowledgment of GPT 5.5
   assistance (line 39) and alpha quality (lines 41, 59-61) is an honest
   engineering posture worth highlighting. The `misc/` design documents
   (`RESPONSE_API.md`, `ANTHROPIC_LIVE_CONTINUATION.md`) with their QA logs
   further evidence this.

6. **Keep the llama.cpp/GGML provenance narrative:** The repo handles this
   better than most projects -- both README acknowledgements and the LICENSE
   file retaining ggml copyright. Mention that ds4 does NOT link against GGML
   but derives quant layouts and CPU kernels from it.

7. **Mention the hand-rolled SHA1:** The KV cache key is computed by an
   embedded SHA1 implementation, not OpenSSL. This reinforces the
   "self-contained" claim.

8. **Add the endpoint completeness caveat:** The server supports five endpoints
   (chat/completions, responses, completions, messages, models) but is
   single-session -- requests serialize through one Metal/CUDA worker.
   Concurrent requests wait their turn. The README is clear about this; the
   submission should be too.
