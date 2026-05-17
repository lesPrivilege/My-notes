# CC Desktop Switch

- **来源**: https://github.com/lonr-6/cc-desktop-switch
- **日期**: 2026-04-24（创建）/ 2026-04-30（最近提交）
- **作者/机构**: lonr-6
- **类型**: repo
- **置信度**: high
- **模式**: deep-review

## 核心问题

Claude Desktop 官方客户端不提供 GUI 方式切换第三方 API 供應商。用户需要手动编辑配置文件或注册表，门槛高且容易出错。cc-desktop-switch 用一个轻量桌面应用解决这个问题：选供應商 → 填 Key → 一键写入 Claude Desktop 配置。

## 核心方案

1. **Anthropic 兼容接口直連（稳定路径）**：直接将供應商的 Anthropic 兼容 URL + API Key 写入 Claude Desktop 的 Windows Registry / macOS plist，关闭工具后 Claude Desktop 仍可独立使用。不依赖本机转发。

2. **OpenAI/new-api/反代格式适配（实验路径）**：通过 `api_adapters.py` 做 Anthropic ↔ OpenAI 格式互转（tool_use/tool_result/content block 转换），走本机 `:18080` 转发服务。

3. **内建供應商预设**：DeepSeek、Kimi、智谱 GLM、阿里云百炼、小米 MiMo 等，预填 API 地址、模型映射、1M 上下文选项、Max 思維开关。

4. **桌面 GUI + 系统托盘**：pywebview 渲染 HTML/CSS/JS 前端（Bootstrap 5.3），pystray 实现系统托盘常驻。浏览器 `127.0.0.1:18081` 作为备用入口。

5. **配置持久化**：`~/.cc-desktop-switch/config.json` 存储所有供應商配置、设置、活跃供應商状态。支持备份和回滚。

## 证据

- **Stars**: 125，**Forks**: 9（创建仅 6 天，增长较快）
- **技术栈**: Python 3.11+, FastAPI, httpx, uvicorn, pywebview, pystray, Pillow
- **版本**: v1.0.16，有 NSIS 安装包 + 便携版
- **内建预设**: DeepSeek（含 1M/Max 思維）、Kimi、智谱、百炼（含 Token Plan）、小米 MiMo 等
- **跨平台**: Windows（完整支持）、macOS（维护者单独同步）、Linux（仅后端/代理）

## 架构

```
main.py (入口)
├── backend/
│   ├── main.py          # FastAPI app, admin API 路由
│   ├── registry.py      # Windows Registry / macOS plist 写入
│   ├── config.py        # JSON 配置管理, 预设定义
│   ├── api_adapters.py  # Anthropic ↔ OpenAI 格式转换
│   ├── proxy.py         # 本机 :18080 转发服务
│   ├── model_alias.py   # 模型名别名映射
│   ├── provider_tools.py # 供應商工具/连通测试
│   ├── i18n.py          # 中英文国际化
│   ├── ccswitch_import.py # 从 farion1231/cc-switch 导入配置
│   └── update.py        # 自动更新检查
├── frontend/
│   ├── index.html       # 单页应用
│   ├── css/             # 样式
│   └── js/              # api.js, app.js, i18n.js
├── main.py              # 启动入口 (pywebview + tray)
└── build.bat / build.spec / installer.nsi  # 打包脚本
```

## 风险与弱点

- ⚠️ **项目极新**（6 天），尚无长期维护信号，bus factor = 1
- ⚠️ **Windows 构建未签名**，会触发"未知发布者"警告
- ⚠️ **实验兼容路径**（OpenAI/new-api/反代）功能不完整，tool_call 转换可能不覆盖所有 edge case
- ⚠️ **macOS 版本由外部维护者同步**，可能存在版本滞后
- ⚠️ **配置文件含明文 API Key**（`~/.cc-desktop-switch/config.json`），无加密保护
- ⚠️ **直接写入系统注册表/plist**，操作失败可能影响 Claude Desktop 配置
- ⚠️ **Claude Desktop 的第三方推理配置字段**（`inferenceProvider`, `inferenceGatewayBaseUrl` 等）属于非官方 API，可能随 Claude Desktop 更新而变化

## 待验证问题

- Claude Desktop 更新后，Registry/plist 配置字段是否仍然兼容？
- OpenAI 格式适配层对复杂 tool_use 场景（如并行工具调用、嵌套 content block）的覆盖率如何？
- 1M 上下文和 Max 思維在实际使用中是否稳定？
- macOS 版本的同步机制和质量如何？
- 本机转发服务（:18080）的性能和稳定性如何？高并发下是否可靠？
