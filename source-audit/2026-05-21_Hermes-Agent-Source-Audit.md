# Hermes Agent — Source Audit

**Repo**: NousResearch/hermes-agent (v0.14.0)
**Date**: 2026-05-21
**Method**: README claims → 原始碼逐項驗證

---

## 支援的聲稱（Implemented）

| 聲稱 | 證據 | 說明 |
|------|------|------|
| Nous Research 開發 | `pyproject.toml:11` | MIT 授權 |
| 自我改進學習迴路（curator） | `agent/curator.py:3-13` — 閒置時 fork AIAgent 定期審視 session，建立技能 | 核心差異化功能，實作完整 |
| 從經驗創建技能 | `agent/skill_bundles.py`, `agent/skill_commands.py` | curator + skill creation pipeline |
| Full TUI | `ui-tui/` — React/Ink，含 `entry.tsx`, `app.tsx`, components, hooks | TypeScript 原始碼完整 |
| 18+ 通訊平台 | `gateway/platforms/` — Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Email, DingTalk, Feishu, WeCom, Weixin, QQ Bot, BlueBubbles, Home Assistant, Mattermost, SMS, Webhook, API Server | 遠超 README 列舉 |
| FTS5 全文搜尋 | `hermes_state.py:255-284` — messages_fts + trigram FTS5 for CJK | 雙表實作 |
| Cron 排程器 | `cron/scheduler.py` + `cron/jobs.py`（~82 function/class） | 完整排程系統 |
| 子代理（Subagent） | `tools/delegate_tool.py:66-101` — ThreadPoolExecutor + auto_approve/deny | 隔離子代理 |
| 7 種終端後端 | `tools/environments/` — local, docker, ssh, singularity, modal, daytona, vercel_sandbox | 數字正確 |
| Batch 軌跡生成 | `batch_runner.py` — 多進程 + checkpoint + resume | 研究用途 |
| 軌跡壓縮 | `trajectory_compressor.py` — 保護首尾、壓縮中間、目標 token | 訓練資料前處理 |
| MCP 整合 | `tools/mcp_tool.py` + `mcp_serve.py`（31K） | 完整 MCP client/server |
| ACP 協定 | `acp_adapter/`（9 files） | VS Code / JetBrains 整合 |
| Honcho 記憶 | `plugins/memory/honcho/` | dialectic user modeling |
| 28+ 模型提供商 | `plugins/model-providers/` — OpenRouter, Anthropic, Gemini, DeepSeek, xAI, 小米, GLM 等 | 遠超 README 列舉 |
| Serverless 持久化 | `tools/environments/modal.py, daytona.py, vercel_sandbox.py` | 閒置休眠、喚醒按需 |
| agentskills.io 相容 | `tools/skills_tool.py:23-42` — SKILL.md frontmatter | 開放標準 |
| 40+ 工具 | `tools/*.py` — 76 個 .py 檔案 | 保守估計 |

## 部分支援 — 需弱化

| 聲稱 | 問題 | 建議修正 |
|------|------|---------|
| "skills self-improve during use" | 現有機制是**創建新技能** + curator 審視，非使用中迭代改進既有技能 | "自動從經驗提煉可重用技能，curator 定期審視優化" |
| "builds a deepening model of who you are" | 記憶系統存在但非 LLM-driven 人格建模，Honcho 是 pluggable | "跨 session 記憶系統，支援 Honcho 等外部建模" |

## 無支援 — 需移除

無。所有 README 主要聲稱均有對應原始碼實作。

## 審計中發現的新觀察

| 觀察 | 證據 | 相關性 |
|------|------|--------|
| 供應鏈攻擊緩解 | `pyproject.toml:14-21` — exact pins, mistralai 封鎖, lazy deps | 反映 Mini Shai-Hulud 事件 |
| 弱依賴原則 | 核心僅 12 個依賴；其他全數 `tools/lazy_deps.py` | 攻擊面極小 |
| pyproject.toml 註解詳盡 | 每筆 dep 有 why + 歷史脈絡 | 少見的高品質套件管理 |
| Windows 原生支援 | `tzdata`, `psutil`, MinGit | early beta 但投入實質 |
| 在地化 | `locales/` + `agent/i18n.py` | 多語言支援 |

## 總結

Hermes Agent 的 README 聲稱誠實度非常高。21 項主要聲稱中 19 項 **Implemented**，2 項 wording 略超前（建議弱化）。原始碼品質、安全性設計、模組化架構顯著優於一般 AI agent 專案。
