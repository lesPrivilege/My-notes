# How Claude Code works in large codebases: Best practices and where to start

- **来源**: https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start
- **日期**: 2026-05-14
- **作者/机构**: Anthropic (Claude Blog)
- **类型**: article
- **置信度**: high
- **模式**: deep-review

## 核心问题

Large codebases (multi-million-line monorepos, decades-old legacy systems, distributed microservice architectures) break the assumptions that work for small projects. RAG-powered AI coding tools struggle because embedding pipelines can't keep up with active engineering — by the time you query an index, it reflects code that existed weeks ago. Claude Code uses agentic search instead (no index), but that shifts the burden onto *how well the codebase is set up* to guide navigation.

## 核心方案

The article argues for **the harness, not just the model** — a 5-layer extension system that determines real-world performance more than the model version:

1. **CLAUDE.md hierarchy** — root file for big-picture context, subdirectory files for local conventions. Ancestor loading (walks up directory tree) + descendant lazy loading (per-directory only on file access). Keeps context lean.

2. **Hooks as self-improvement** — not just policing, but reflective: stop hooks can propose CLAUDE.md updates after sessions; start hooks inject team-specific context dynamically.

3. **Skills for progressive disclosure** — specialized workflows (security review, docs update) stay off-screen until triggered. Path-scoped so teams only load what's relevant.

4. **Plugins for distribution** — bundle skills + hooks + MCP config into installable packages. Managed marketplace for org-wide rollout. Solves "good setups stay tribal."

5. **LSP integrations** — symbol-level precision (go to definition, find all references) instead of text pattern-matching that lands on wrong symbols in multi-language repos.

**Navigation model**: Traverses filesystem, reads files, uses grep, follows references — like a human engineer. No centralized index. Each instance works from live codebase. Tradeoff: needs good starting context.

## 证据

- Observations from "multi-million-line monorepos, decades-old legacy systems, distributed architectures" deployments
- RAG failure: index returns functions renamed weeks ago, no freshness signal
- LSP value: distinguishes identically named functions across languages
- Example: large retail org built analytics skill, distributed as plugin before org-wide rollout
- Example: enterprise software company deployed LSP org-wide before Claude Code rollout

(Experience-report based, not benchmarked — qualitative/anecdotal.)

## 风险与弱点

- ⚠️ No quantitative benchmarks — all claims observational
- ⚠️ Setup burden shifts to the team — agentic search avoids index maintenance but demands CLAUDE.md investment
- ⚠️ Hook-driven self-improvement requires careful prompt engineering; bad hooks degrade rather than improve
- ⚠️ LSP integration assumes working LSP — C++ monorepos with custom build systems often have partial coverage
- ⚠️ Plugin distribution presupposes org has standardized on Claude Code (outcome, not enabler)

## 待验证问题

- Does agentic search scale sublinearly, or hit practical limits on 100M+ line repos?
- How does CLAUDE.md depth affect context-window utilization in practice?
- Can LSP handle multi-language "go to definition" ambiguity robustly?
- What's the practical CLAUDE.md maintenance cost per engineer per week?
- Minimum team size / codebase complexity below which harness investment isn't worth it?

---

**Key insight**: Central thesis — *harness > model* — is a strong, counterintuitive claim from a model-making company. Shapes Claude Code's moat: switching cost isn't just learning a new CLI, it's rebuilding the entire harness stack. Five-layer model is useful as a maturity framework — most teams stop at layer 1 (CLAUDE.md) and don't realize what each subsequent layer unlocks.
