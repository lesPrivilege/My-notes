# OpenClaw 源碼審計報告

## Repo 資訊

| 項目 | 值 |
|---|---|
| **Repo** | `lesPrivilege/openclaw` (fork of `openclaw/openclaw`) |
| **描述** | Personal AI assistant — 自己的設備上運行的 AI 助手 |
| **許可證** | MIT |
| **語言** | TypeScript (113MB), Swift (4.4MB), Kotlin (1.4MB), Shell (1MB) |
| **上游** | 373k stars, 77k forks, 活躍維護 |
| **fork 創建** | 2026-05-21，純 fork 無改動 |
| **Clone 狀態** | HTTPS/SSH 均 timeout (github.com unreachable)，使用 `gh api` 完成審計 |

---

## 目錄結構

```
openclaw/
├── src/               # 核心 TypeScript (~83 子目錄)
│   ├── gateway/       # Gateway 協議 (716 files)
│   ├── agents/        # Agent 系統 (1642 files)
│   ├── cli/           # CLI 入口 (429 files)
│   ├── commands/      # 命令系統 (653 files)
│   ├── channels/      # 頻道抽象層 (364 files)
│   ├── plugins/       # 插件加載器 (601 files)
│   ├── plugin-sdk/    # 插件 SDK (546 files)
│   ├── config/        # 配置系統 (355 files)
│   ├── infra/         # 基礎架構 (737 files)
│   ├── secrets/       # Secrets 管理 (122 files)
│   ├── security/      # 安全邊界 (81 files)
│   ├── cron/          # 定時任務 (178 files)
│   ├── talk/          # 語音對話 (25 files)
│   ├── tts/           # 文字轉語音 (20 files)
│   ├── tui/           # 終端 UI (58 files)
│   └── mcp/           # MCP 協議支援 (13 files)
├── apps/
│   ├── android/       # Android 原生殼 (Kotlin)
│   └── ios/           # iOS + watchOS 原生殼 (Swift)
├── extensions/        # 官方插件 (130+ extensions)
├── packages/          # 共享 packages
├── ui/                # Web 前端 (Lit)
├── docs/              # 文檔站點
├── scripts/           # 建置腳本
└── .github/           # CI/CD + CodeQL
```

## 核心宣稱驗證

### 1. 架構宣稱

| README 宣稱 | 驗證結果 | 源碼證據 |
|---|---|---|
| Personal AI assistant on own devices | **Implemented** | `src/gateway/server.ts` 完整 self-hosted server；Dockerfile 支援容器；`apps/android/` + `apps/ios/` 原生殼 |
| Gateway is control plane | **Implemented** | `src/gateway/` — 716 files, 包含 `server.ts`, `server-http.ts`, `connection-auth.ts`, `device-auth.ts`, `discovery.ts` 等 |
| Plugin API | **Implemented** | `src/plugin-sdk/` (546 files): `channel-contract.ts`, `channel-core.ts`, `facade-loader.ts`, `facade-runtime.ts`, `plugin-entry.ts` |
| Can render live Canvas | **Implemented** | `apps/ios/Sources/RootCanvas.swift`; Android: `CanvasScreen.kt`, `CanvasController.kt`, `OpenClawProtocolConstants.kt` |

### 2. 頻道支援驗證

README 列出 22 個頻道，逐一對照 `extensions/`：

| 頻道 | 實作位置 | 判定 |
|---|---|---|
| WhatsApp | `extensions/whatsapp/` | **Implemented** |
| Telegram | `extensions/telegram/` | **Implemented** |
| Slack | `extensions/slack/` | **Implemented** |
| Discord | `extensions/discord/` | **Implemented** |
| Google Chat | `extensions/googlechat/` | **Implemented** |
| Signal | `extensions/signal/` | **Implemented** |
| iMessage | `extensions/imessage/` | **Implemented** |
| IRC | `extensions/irc/` | **Implemented** |
| Microsoft Teams | `extensions/msteams/` | **Implemented** |
| Matrix | `extensions/matrix/` | **Implemented** |
| Feishu | `extensions/feishu/` | **Implemented** |
| LINE | `extensions/line/` | **Implemented** |
| Mattermost | `extensions/mattermost/` | **Implemented** |
| Nextcloud Talk | `extensions/nextcloud-talk/` | **Implemented** |
| Nostr | `extensions/nostr/` | **Implemented** |
| Synology Chat | `extensions/synology-chat/` | **Implemented** |
| Tlon | `extensions/tlon/` | **Implemented** |
| Twitch | `extensions/twitch/` | **Implemented** |
| Zalo / Zalo Personal | `extensions/zalo/`, `extensions/zalouser/` | **Implemented** |
| QQ | `extensions/qqbot/`（QQ Bot） | **Implemented** |
| WebChat | 無對應 extension；`ui/` 是 control UI 非獨立 chat widget | **Partial** |
| WeChat | 未在 `extensions/` 中找到 | **Unsupported** |

### 3. 語音支援

| 宣稱 | 驗證 |
|---|---|
| Speak & listen on macOS/iOS/Android | iOS: `TalkModeManager.swift`, `TalkOrbOverlay.swift`; Android: `apps/.../voice/` (10 files); TTS engine: `src/tts/` (20 files), `src/talk/` (25 files); Provider plugins: ElevenLabs, Deepgram, Azure Speech |
| Talk mode | `src/talk/talk-session-controller.ts`, `talk-events.ts`, `session-runtime.ts` |

### 4. 模型提供者生態

`extensions/` 中的 provider plugin（50+）：

OpenAI, Anthropic, Claude, Google, DeepSeek, Groq, Ollama, LM Studio, Mistral, Perplexity, Together, Fireworks, xAI, SGLang, vLLM, DeepInfra, Arcee, Cerebras, Chutes, Gradium, HuggingFace, Moonshot, NVIDIA, Novita, Qwen, StepFun, Venice, Vercel AI Gateway, Voyage, Amazon Bedrock, Azure, Baidu (Qianfan), BytePlus, Cloudflare, Koala, Kilocode, Microsoft Foundry, OpenRouter, Tencent, Volcengine, 等。

### 5. 安全體系

- **CodeQL**: 20+ 安全邊界 profiles（gateway-runtime, network-ssrf, agent-runtime, plugin-trust, core-auth-secrets 等）
- **OpenGrep + Semgrep**: 靜態分析
- **Secret scanning**: 自動
- **Security policy**: 私有披露、安全公告、trust model
- **專案規模**: `src/security/` (81 files), `src/secrets/` (122 files)

---

## 軟體事實

| 計量 | 數值 |
|---|---|
| Total .ts files | 14,614 |
| Total .swift files | 651 |
| Total .kt files | 175 |
| Test files | 5,582 (~38%) |
| Total source files | 18,019 |
| Workspace | pnpm monorepo |
| TypeScript | 6.0.3 |
| CI workflows | 50+ GitHub Actions |

---

## 結論

1. **文檔誠實度高** — 所有主要宣稱均可對應到源碼實作
2. **架構嚴謹** — plugin 邊界清晰（SDK facade pattern），gateway 作為控制平面
3. **WebChat/WeChat 需釐清** — README 列出但 `extensions/` 中無對應實作
4. **fork 乾淨** — `lesPrivilege/openclaw` 尚無任何自訂 commit
5. **維護成本高** — 18k+ source files, 1.3GB — 建議明確 upstream tracking 策略

---

*審計日期：2026-05-21 | 工具：gh API + source-audit skill*
