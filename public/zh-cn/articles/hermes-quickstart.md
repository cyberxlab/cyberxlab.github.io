# Hermes Agent 快速入门

本指南带你从零开始搭建一个经得起实际使用的 Hermes Agent 环境。安装、选择模型供应商、验证聊天功能可用，并在遇到问题时知道该怎么修。

## 最快路径

选择最符合你目标的路径：

| 目标 | 第一步 | 第二步 |
| --- | --- | --- |
| 我只想在本机跑起来 | `hermes setup` | 运行一次真实聊天并验证响应 |
| 我已经知道用哪个供应商 | `hermes model` | 保存配置后开始聊天 |
| 我想要机器人或常驻服务 | CLI 可用后再 `hermes gateway setup` | 连接 Telegram、Discord、Slack 等平台 |
| 我想用本地或自托管模型 | `hermes model` → 自定义端点 | 验证端点地址、模型名和上下文长度 |
| 我想要多供应商容错 | 先 `hermes model` | 基础聊天稳定后再加路由和容错 |

**经验法则：** 如果 Hermes 连一次正常聊天都跑不通，不要急着加更多功能。先让一次干净的对话跑起来，然后再叠网关、定时任务、技能、语音或路由。

---

## 1. 安装 Hermes Agent

### 使用 Hermes Desktop 安装器（推荐）

前往 [Hermes Agent 官网](https://hermes-agent.nousresearch.com/)下载桌面安装器，运行即可同时安装命令行和桌面应用。

### 不使用 Hermes Desktop

仅安装命令行：

#### Linux / macOS / WSL2 / Android (Termux)

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

#### Windows (原生)

在 PowerShell 中执行：

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

安装完成后重新加载 shell：

```bash
source ~/.bashrc   # 或 source ~/.zshrc
```

## 2. 选择模型供应商

这是最关键的一步。用 `hermes model` 交互式选择：

```bash
hermes model
```

最简单的路径——**Nous Portal**：一个订阅覆盖 300+ 模型，还附带工具网关（网页搜索、图片生成、TTS、云端浏览器）。全新安装时：

```bash
hermes setup --portal
```

### 常用供应商

| 供应商 | 说明 | 配置方式 |
| --- | --- | --- |
| **Nous Portal** | 订阅制，零配置 | `hermes model` OAuth 登录 |
| **OpenAI Codex** | ChatGPT OAuth，使用 Codex 模型 | `hermes model` 设备码认证 |
| **Anthropic** | Claude 模型（Max 订阅或 API Key） | `hermes model` → OAuth 或 API Key |
| **OpenRouter** | 多供应商路由 | 输入 API Key |
| **Fireworks AI** | OpenAI 兼容 API | `FIREWORKS_API_KEY` |
| **Z.AI** | GLM / 智谱模型 | `GLM_API_KEY` / `ZAI_API_KEY` |
| **DeepSeek** | DeepSeek 直连 API | `DEEPSEEK_API_KEY` |
| **Hugging Face** | 20+ 开源模型统一路由 | `HF_TOKEN` |
| **AWS Bedrock** | Claude、Nova、Llama | IAM 角色或 `aws configure` |
| **Azure Foundry** | Azure AI Foundry 模型 | `AZURE_FOUNDRY_API_KEY` + base URL |
| **Google AI Studio** | Gemini 模型 | `GOOGLE_API_KEY` |
| **xAI** | Grok 模型 | `XAI_API_KEY` |
| **自定义端点** | vLLM、SGLang、Ollama 或任何 OpenAI 兼容 API | 设置 base URL + API Key |

> Hermes 要求模型至少有 **64,000 tokens** 的上下文窗口。大多数托管模型都满足此要求。

### 配置存储位置

- **密钥和令牌** → `~/.hermes/.env`
- **非密钥设置** → `~/.hermes/config.yaml`

```bash
hermes config set model anthropic/claude-opus-4.6
hermes config set terminal.backend docker
hermes config set OPENROUTER_API_KEY sk-or-...
```

## 3. 运行第一次聊天

```bash
hermes            # 经典 CLI
hermes --tui      # 现代 TUI（推荐）
```

你会看到欢迎横幅，显示模型、可用工具和技能。用一个具体且容易验证的提示：

```text
用 5 个要点总结这个仓库，告诉我主入口文件是什么。
```

**成功的标志：**

- 横幅显示你选择的模型/供应商
- Hermes 正常回复，无报错
- 需要时能调用工具（终端、文件读取、网页搜索）
- 对话能正常持续多轮

## 4. 验证会话功能

```bash
hermes --continue    # 恢复最近会话
hermes -c            # 简写形式
```

## 5. 尝试关键功能

### 终端使用

```text
❯ 我的磁盘使用情况如何？列出最大的 5 个目录。
```

### 斜杠命令

输入 `/` 查看命令自动补全列表：

| 命令 | 作用 |
| --- | --- |
| `/help` | 显示所有可用命令 |
| `/tools` | 列出可用工具 |
| `/model` | 交互式切换模型 |
| `/save` | 保存对话 |

### 多行输入

按 `Alt+Enter`、`Ctrl+J` 或 `Shift+Enter` 换行。

### 中断 Agent

输入新消息并按 Enter——会中断当前任务。`Ctrl+C` 同样有效。

## 6. 叠加下一层功能

基础聊天稳定后再加：

### 机器人或消息网关

```bash
hermes gateway setup
```

连接 Telegram、Discord、Slack、WhatsApp、Signal、Email、Home Assistant 或 Microsoft Teams。

### 沙箱终端

```bash
hermes config set terminal.backend docker    # Docker 隔离
hermes config set terminal.backend ssh       # 远程服务器
```

### 语音模式

```bash
cd ~/.hermes/hermes-agent
uv pip install -e ".[voice]"
```

然后在 CLI 中输入 `/voice on`。按 `Ctrl+B` 录音。

### 技能系统

```bash
hermes skills browse                      # 浏览所有可用技能
hermes skills search kubernetes           # 按关键词搜索
hermes skills install openai/skills/k8s   # 安装一个
```

### MCP 服务器

```yaml
# 添加到 ~/.hermes/config.yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_xxx"
```

---

## 常见故障排查

| 症状 | 可能原因 | 修复方法 |
| --- | --- | --- |
| Hermes 打开了但回复为空或异常 | 供应商认证或模型选择有误 | 重新运行 `hermes model` |
| 自定义端点"能用"但返回乱码 | base URL 或模型名不对 | 先用其他客户端验证端点 |
| 网关启动了但没人能发消息 | Bot token 或白名单设置不完整 | 重新运行 `hermes gateway setup` |
| `--continue` 找不到旧会话 | 切换了 profile 或会话从未保存 | 检查 `hermes sessions list` |
| 模型不可用或回退行为异常 | 路由设置过于激进 | 基础供应商稳定前关闭路由 |

## 恢复工具箱

1. `hermes doctor`
2. `hermes model`
3. `hermes setup`
4. `hermes sessions list`
5. `hermes --continue`
6. `hermes gateway status`

## 快速参考

| 命令 | 说明 |
| --- | --- |
| `hermes` | 开始聊天 |
| `hermes model` | 选择 LLM 供应商和模型 |
| `hermes tools` | 配置各平台工具权限 |
| `hermes setup` | 完整设置向导 |
| `hermes doctor` | 诊断问题 |
| `hermes update` | 更新到最新版 |
| `hermes gateway` | 启动消息网关 |
| `hermes --continue` | 恢复上次会话 |

---

*来源：[Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart)*

