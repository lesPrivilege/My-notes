---
name: env-audit
description: >-
  Audit the local environment: Claude Code token usage, session storage, user additions to Claude Code, or installed software. Trigger: "查用量", "token usage", "session audit", "会话审计", "claude-audit", "claude diff", "system-audit", "系统软件画像", "installed what tools".
---

# Skill: env-audit

Report facts — names, versions, counts. No scoring or recommendations unless asked. Never delete anything without user sign-off.

## Token usage

`~/Scripts/claude-usage` (flags: `--days N`, `--since DATE`, `--json`). Data source: `~/.claude/projects/*/*.jsonl` `message.usage`. Cache hit rate = cache_read / total. `<synthetic>` entries carry no tokens. Local machine only.

## Sessions

- `~/.claude/sessions/*.json` — active sessions
- `~/.claude/history.jsonl` — one JSON object per user message; group by `conversation_id`

## Claude Code additions (baseline = factory install)

Scan and report:

- `~/.claude/skills/` — symlink to `~/.pi/agent/skills` (git repo `lesPrivilege/My-skills`)
- `~/.claude/settings.json`, `~/.claude/settings.local.json`, `~/.claude/CLAUDE.md`
- `~/.pi/agent/AGENTS.md`
- `~/Scripts/` files referenced by skills

## Installed software

`brew list` / `brew leaves` / `pip list` / `npm list -g --depth=0` / `gem list` / `ls /Applications` / `echo $PATH` / `launchctl list`. Organized by layer.
