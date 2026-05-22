# Reasonix Audit

## Supported Claims

- **Claim:** "Reasonix is a TypeScript agent loop framework built around cache-first design"
  **Evidence:** `package.json` line 4: `"description": "DeepSeek-native coding agent: cache-first loop, flash-first cost control, tool-call repair."` confirms this positioning. The `CacheFirstLoop` class at `src/loop.ts:104` is the main loop class. `README.md` line 41: "A DeepSeek-native AI coding agent for your terminal." `README.md` lines 52-53: cache stability is called "an invariant the loop is designed around." The `REASONIX.md` line 3 also says "DeepSeek-native coding agent, cache-first loop." All of TypeScript, ESM modules per `package.json` line 6. **This claim is fully supported and can be cited exactly.**

- **Claim:** "Three-zone context partition: IMMUTABLE PREFIX (system prompt + tool specs), APPEND-ONLY LOG (messages + tool results, monotonic append), VOLATILE SCRATCH (per-turn reset)"
  **Evidence:** `docs/ARCHITECTURE.md` lines 24-35 contain the exact ASCII diagram with three labeled zones and the invariants listed below. The implementation is at `src/memory/runtime.ts` with three exported classes: `ImmutablePrefix` (line 10, has system prompt, toolSpecs, fewShots), `AppendOnlyLog` (line 91, has append() method with monotonic invariant, compactInPlace as the one exception path), and `VolatileScratch` (line 123, has reset() method clearing reasoning/planState/notes per turn). These are imported into and used by the main loop at `src/loop.ts` line 43: `import { AppendOnlyLog, type ImmutablePrefix, VolatileScratch } from "./memory/runtime.js"`. The loop calls `this.scratch.reset()` at line 574 on each step. **Fully supported, matches architecture document, code exists.**

- **Claim:** "Tool-call repair pipeline: Flatten, Scavenge, Truncation, Storm"
  **Evidence:** `src/repair/` directory contains exactly four files: `flatten.ts`, `scavenge.ts`, `truncation.ts`, `storm.ts`. The `src/repair/index.ts` line 1 says `/** Pass order: scavenge -> truncation -> storm. Schema flatten runs at loop construction, not per-turn. */` The order within `ToolCallRepair.process()` at lines 53-124 is: 1) scavenge lines 71-84, 2) truncation lines 87-108, 3) storm lines 112-121. Flatten runs at tool registration time (`analyzeSchema` in `flatten.ts` line 11). `storm.ts` line 14 exports `StormBreaker` class. **The exact names flatten/scavenge/truncation/storm are all correct as source-code identifiers. The only nuance: flatten runs at construction, not per-turn, and the process order is scavenge -> truncation -> storm. Fully supported.**

- **Claim:** '"Coupling to one backend is the feature, not a limitation" — from README'
  **Evidence:** `README.md` line 268, under the "Non-goals" section: "**Multi-provider flexibility.** DeepSeek-only on purpose. Coupling to one backend is the feature, not a limitation." **This is an exact quote and can be cited as README.md:268. Fully supported.**

- **Claim:** "99.82% cache hit rate, 435M input tokens in one day, ~$12 vs ~$61 under generic framework — from repo benchmarks"
  **Evidence:** `benchmarks/real-world-cache/README.md` contains the exact numbers. Lines 10-14: input cache hit 435,033,856, cache miss 767,616, output 179,763, total 435,981,235. Line 18: `99.82%`. Lines 28-33: "$12.34" vs "$60.63" assuming v4-flash. **However**: the claim says "~$12 vs ~$61" which is approximate — the actual figures are $12.34 vs $60.63. The claim also says "under generic framework" but the benchmark compares 99.82% hit rate to a hypothetical "0% cache" baseline, not to an actual generic framework run. The README itself says "Same workload, 0% cache" which is a synthetic baseline, not a measured competitor. **The numbers exist and are real (from a user's DeepSeek dashboard, used with permission). Suggested wording: cite as "real user, single day (2026-05-01): 435M input tokens, 99.82% cache hit, ~$12.34 vs ~$60.63 hypothetical uncached baseline on v4-flash" and add the qualifier "the uncached figure is a counterfactual (0% cache hit), not a measured generic framework."**

- **Claim:** "Skills, memory, hooks, MCP, permissions, semantic index capabilities"
  **Evidence:** All verified in source code:
  - **Skills**: `src/skills.ts` contains `SkillStore` class (line 93), `applySkillsIndex` function (line 334), and 5 built-in skills (explore, research, review, security-review, test) at lines 510-556. Also `src/tools/skills.ts`.
  - **Memory**: `src/memory/user.ts` has `MemoryStore` class with types user/feedback/project/reference. `src/memory/project.ts` handles `REASONIX.md`. `src/tools/memory.ts` has `remember` / `recall_memory` tools.
  - **Hooks**: `src/hooks.ts` with full lifecycle: PreToolUse, PostToolUse, UserPromptSubmit, Stop. `class HookConfig`, `class ResolvedHook`, `runHooks()` function at line 318.
  - **MCP**: `src/mcp/client.ts` (`McpClient` class), `src/mcp/stdio.ts`, `src/mcp/sse.ts`, `src/mcp/streamable-http.ts`, `src/mcp/spec.ts`, `src/mcp/registry.ts`.
  - **Permissions**: `src/tools/shell.ts` has `BUILTIN_ALLOWLIST` and `isAllowed()`. `src/cli/ui/slash/handlers/permissions.ts` has `/permissions` slash command. `src/server/api/permissions.ts` has dashboard API.
  - **Semantic index**: `src/index/semantic/` directory with `builder.ts`, `chunker.ts`, `embedding.ts`, `store.ts`, `tool.ts`, `ollama-launcher.ts`. Supports both Ollama and OpenAI-compatible embedding providers (see `src/server/api/semantic.ts` lines 497-498 and `src/config.ts` line 25: `export type EmbeddingProvider = "ollama" | "openai-compat"`).
  **All six capabilities exist in source code with usable implementations. For Harness PM evaluation: MCP, hooks, and permissions are the most mature and valuable. Semantic index is valuable but requires external Ollama or an OpenAI-compatible embedding endpoint.**

- **Claim:** "Desktop client (prerelease) plus CLI"
  **Evidence:** `README.md` lines 124-133: "Desktop client (prerelease)" section. The desktop is a Tauri app at `desktop/` directory with `desktop/package.json` version `0.43.0`, React/Vite frontend at `desktop/src/`, Tauri backend at `desktop/src-tauri/`. README explicitly says "The desktop ships as a **prerelease**: the loop and protocol are the same as the CLI, but the UI is still being polished and the installers aren't code-signed yet." **Fully supported.**

- **Claim:** "Comparison table vs Claude Code / Cursor / Aider"
  **Evidence:** `README.md` lines 208-221, "How it compares" table. Columns: Reasonix, Claude Code, Cursor, Aider. Rows: Backend, License, Cost profile, DeepSeek prefix-cache, Embedded web dashboard, Configurable web search engine, Persistent per-workspace sessions, Plan mode / MCP / hooks / skills, Web search, Open community development. **The comparison table exists in the README. Fully supported.**

## Partially Supported / Needs Weakening

- **Claim:** "DeepSeek-native AI coding agent" / "agent loop framework"
  **Evidence:** `package.json` description says "coding agent." `README.md` says "AI coding agent for your terminal." The `CacheFirstLoop` class is used as a library entry point exported from `src/index.ts` line 6-7. However, the term "agent loop framework" suggests it's a general-purpose framework for building agents, when the actual positioning is more specific: a "DeepSeek-native AI coding agent." The `docs/ARCHITECTURE.md` line 5 says "Reasonix is **opinionated, not general**." The REASONIX.md line 3 says "DeepSeek-native coding agent" not framework. `package.json` keyword "agent" is present.
  **Problem:** The repo positions itself as a "coding agent" (an end-user product) more than an "agent loop framework" (a developer toolkit). It can be used as a framework (the `src/index.ts` library entry exports `CacheFirstLoop`, `ImmutablePrefix`, `AppendOnlyLog`, etc.), but calling it a "framework" without the "coding agent" qualifier overstates generality.
  **Safer wording:** "DeepSeek-native AI coding agent and agent-loop library," or "a TypeScript agent-loop framework purpose-built as a DeepSeek-first coding agent."

- **Claim:** "Agent loop framework" (without qualification)
  **Evidence:** The repo exports a library via `src/index.ts` with `CacheFirstLoop`, `ToolCallRepair`, `ImmutablePrefix` etc. that could be used as a framework. But the CLI (`reasonix code`) is clearly the primary surface.
  **Problem:** Calling it a "framework" without acknowledging it's first and foremost an end-user coding agent is misleading. The README's first line is "coding agent for your terminal."
  **Safer wording:** "agent-loop framework for building DeepSeek-native coding agents" or acknowledge it's primarily distributed as a CLI coding agent.

- **Claim:** "~$12 vs ~$61 under generic framework"
  **Evidence:** The actual benchmark README (`benchmarks/real-world-cache/README.md` line 33) shows "$12.34" vs "$60.63." The "$61" is a rounding of "$60.63." The baseline is called "Same workload, 0% cache" (a counterfactual scenario), not "under generic framework."
  **Problem:** Saying "under generic framework" implies a real competitor was measured, when the README states it is a 0% cache hypothetical. The tau-bench benchmarks DO compare Reasonix vs a "cache-hostile baseline" (`benchmarks/README.md` lines 8-14), but the real-world-cache study does not.
  **Safer wording:** "A real user's single day: 435M input tokens at 99.82% cache hit, costing ~$12.34 instead of the ~$60.63 the same workload would cost with 0% prefix cache."

- **Claim:** "DeepSeek-only" / "single-model commitment"
  **Evidence:** The ARCHITECTURE.md line 246 says "Support for non-DeepSeek backends (an OpenAI-compatible shim would work today via `--model` override, but is not tested)." The `DeepSeekClient` class at `src/client.ts` accepts a configurable `baseUrl` (line 113: `opts.baseUrl ?? process.env.DEEPSEEK_BASE_URL ?? "https://api.deepseek.com"`). The `--model` CLI flag at `src/cli/index.ts` line 155 accepts any model ID. The `ModelClient` port at `src/ports/model-client.ts` line 1 says "DeepSeek today; pluggable later."
  **Problem:** The client IS configurable. Saying "DeepSeek-only" without the "by design, untested on other backends" qualifier found in the architecture doc overstates the lock-in. The README's "Non-goals" section (line 268) says "DeepSeek-only on purpose" which is the project's position, but technically you CAN pass `--model gpt-4` -- it just won't have cache optimization.
  **Safer wording:** "DeepSeek-only by design (the architecture is tuned to DeepSeek's specific prefix-cache behavior; an OpenAI-compatible shim is technically possible via `--model` override but untested and unsupported)."

- **Claim:** "Cache-first loop" as "the official architectural mainline"
  **Evidence:** `docs/ARCHITECTURE.md` titles it "The four pillars" with Pillar 1 being "Cache-First Loop." The main loop class is `CacheFirstLoop` at `src/loop.ts`. However, the architecture document also gives Pillar 2 (tool-call repair) and Pillar 3 (cost control) equal billing.
  **Problem:** Calling it "the" mainline is fine since it's Pillar 1, but the three pillars are described as co-equal subsystems. The design directly addresses all three.
  **Safer wording:** "Pillar 1 of the three-pillar architecture. The other two pillars are Tool-Call Repair and Cost Control." (This is already how the submission describes it if it says "three pillars." If it singles out cache-first, that's fair since it IS Pillar 1.)

- **Claim:** "Flatten / Scavenge / Truncation / Storm — each targeting specific DeepSeek failure modes"
  **Evidence:** The exact names exist. The mapping to failure modes is in `docs/ARCHITECTURE.md` lines 72-88. However, the pass order in `src/repair/index.ts` line 1 says "scavenge -> truncation -> storm" with flatten running at construction. The claim says just "Flatten, Scavenge, Truncation, Storm" without specifying the runtime order.
  **Problem:** Minor - the claim doesn't specify ordering and flatten runs at tool registration, not per-turn. Not misleading but could be clarified.
  **Safer wording:** "Four-pass repair pipeline: flatten (at tool registration), scavenge (per-turn), truncation (per-turn), storm (per-turn)."

## Unsupported / Remove

- **Claim:** "435M input tokens in one day" comes from the repo's own benchmarks
  **Search performed:** Checked `benchmarks/real-world-cache/README.md` and `benchmarks/README.md`.
  **Reason:** The data is from a SINGLE real user's DeepSeek dashboard screenshot, used with permission, anonymized. The repo did NOT run its own benchmark to produce these numbers -- it is an external user's self-reported usage data. The README explicitly says "A real Reasonix user shared their DeepSeek dashboard." This is fine to cite as a case study, but it should NOT be claimed as "repo benchmarks" or internal measurement. The repo's OWN benchmark suite is the tau-bench harness under `benchmarks/tau-bench/`. If the submission claims these are "from repo benchmarks," that needs correction to "from a community case study shared with permission."

- **Claim:** "Cache-first loop" as the sole/main architectural focus (without mentioning Pillar 2 and 3)
  **Search performed:** Checked `docs/ARCHITECTURE.md` header "The four pillars" and the three pillar sections.
  **Reason:** The architecture document describes THREE co-equal pillars. Pillar 2 (tool-call repair) and Pillar 3 (cost control) are equally important structural elements. If the submission document presents cache-first as the only architectural feature, it is incomplete. Including all three is the correct framing.

## New Observations Worth Adding

- **Observation:** The ACP (Agent Client Protocol) implementation is a notable architectural element not mentioned in the submission document.
  **Evidence:** `src/acp/` directory with `dispatch.ts`, `gates.ts`, `protocol.ts`, `server.ts`. CLI subcommand at `src/cli/commands/acp.ts`. This allows Reasonix to operate as an ACP-compatible agent on stdio NDJSON JSON-RPC.
  **Why it matters for Harness PM:** ACP is the emerging standard for agent-to-agent communication. Reasonix shipping ACP support means it can participate in multi-agent orchestrations natively -- a significant differentiator vs Claude Code (which does not implement ACP).

- **Observation:** The QQ channel integration is unique and not found in comparable tools.
  **Evidence:** `src/qq/` directory with `access.ts`, `bot.ts`, `channel.ts`, `use-qq-channel.ts`. README lines 97-123 document QQ as a remote communication channel.
  **Why it matters for Harness PM:** This shows the team's awareness of the Chinese market and ability to integrate with non-Western messaging platforms -- valuable context for go-to-market strategy.

- **Observation:** The pricing code (`src/telemetry/stats.ts`) includes Claude Sonnet reference pricing for "equivalent cost" calculations, but this is deprecated and no longer surfaced in the TUI.
  **Evidence:** `src/telemetry/stats.ts` line 34: `export const CLAUDE_SONNET_PRICING = { input: 3.0, output: 15.0 }`. Lines 105-108: `/** @deprecated Claude reference; kept for benchmarks + replay compat, no longer surfaced in the TUI. */`
  **Why it matters for Harness PM:** The team previously showed Claude cost comparisons but deliberately removed them from the UI -- suggesting sensitivity about direct comparisons or a realization they were misleading. The code still computes them but doesn't display them.

- **Observation:** The `ModelClient` port interface at `src/ports/model-client.ts` is explicitly labeled as "DeepSeek today; pluggable later," which contradicts the README's strong "DeepSeek-only" position.
  **Evidence:** `src/ports/model-client.ts` line 1: `/** Port: streaming chat model. Adapters: DeepSeek today; pluggable later. */`
  **Why it matters for Harness PM:** This suggests the team has explicitly designed for future multi-provider support at the port/adapter level, even though the current product position is single-backend. This architectural hedge is important for product planning.

- **Observation:** The codebase has extensive i18n support (English and Chinese).
  **Evidence:** `src/i18n/` directory with `EN.ts`, `zh-CN.ts`, `index.ts`, `types.ts`. `desktop/src/i18n/` has `en.ts`, `zh-CN.ts`.
  **Why it matters for Harness PM:** The bilingual design (English + Simplified Chinese) is a deliberate market choice, not an afterthought. All UI strings, error messages, and documentation are available in both languages.

- **Observation:** The parallel tool dispatch system is carefully engineered with safety guarantees.
  **Evidence:** `docs/ARCHITECTURE.md` lines 46-67 describe the parallel dispatch system. `src/loop.ts` lines 1111-1198 implement it. Tools declare `parallelSafe?: boolean` (default false). The dispatcher groups consecutive parallel-safe calls, races them, but yields results in declared order regardless of settlement timing. Env vars `REASONIX_PARALLEL_MAX` (default 3, cap 16) and `REASONIX_TOOL_DISPATCH=serial` control it.
  **Why it matters for Harness PM:** This is a sophisticated feature that generic frameworks (e.g. LangChain) don't handle well with DeepSeek. It's a genuine technical differentiator worth highlighting.

## Final Rewrite Notes

1. **Fix the "framework vs agent" framing.** Reasonix is primarily positioned as a "DeepSeek-native AI coding agent" (its README positioning). Calling it an "agent loop framework" without also calling it a "coding agent" misrepresents its primary identity. Use "agent-loop framework purpose-built as a DeepSeek-first coding agent."

2. **Cite the three-zone context partition with exact file paths.** The `ImmutablePrefix`, `AppendOnlyLog`, and `VolatileScratch` classes are all at `src/memory/runtime.ts`. The architecture diagram is at `docs/ARCHITECTURE.md` lines 24-35. These are exact and citably correct.

3. **Remove "repo benchmarks" language for the 99.82% cache-hit numbers.** These come from a community case study (`benchmarks/real-world-cache/README.md`), not from the repo's own benchmark harness (`benchmarks/tau-bench/`). Cite as "community case study, 2026-05-01, shared with permission."

4. **Add the cost-control pillar.** The current submission apparently focuses on cache-first and tool-call repair but omits Pillar 3 (cost control), which is one of the three equal pillars in the architecture document. Include: tiered defaults (flash-first), turn-end auto-compaction, /pro single-turn arming, and failure-signal auto-escalation.

5. **Weaken the "DeepSeek-only" claim with the technical qualifier.** The README says "DeepSeek-only on purpose" and that's the official position. But the architecture doc (ARCHITECTURE.md:246) reveals an OpenAI-compatible shim "would work today via `--model` override" -- add this as a technical note.

6. **Add the ACP (Agent Client Protocol) implementation.** This is a significant differentiator not mentioned in the submission document. It positions Reasonix as a native citizen in multi-agent systems.

7. **Explicitly tag the desktop as prerelease.** The README labels it prerelease (not code-signed, UI being polished). The CLI remains the canonical surface. Ensure the submission reflects this accurately.

8. **Clarify the three-pillar architecture as co-equal.** Do not present cache-first as the sole architectural theme. All three pillars (Cache-First Loop, Tool-Call Repair, Cost Control) deserve proportional coverage as they are structured equally in the architecture document.
