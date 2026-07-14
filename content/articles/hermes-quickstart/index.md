---
title: "Hermes Agent Quickstart"
description: "Get from zero to a working Hermes Agent setup that survives real use — install, choose a provider, verify a working chat, and know what to do when something breaks."
date: 2026-07-14
draft: false
---

This guide gets you from zero to a working Hermes setup that survives real use. Install, choose a provider, verify a working chat, and know exactly what to do when something breaks.

## The Fastest Path

Pick the row that matches your goal:

| Goal | Do this first | Then do this |
| --- | --- | --- |
| I just want Hermes working on my machine | `hermes setup` | Run a real chat and verify it responds |
| I already know my provider | `hermes model` | Save the config, then start chatting |
| I want a bot or always-on setup | `hermes gateway setup` after CLI works | Connect Telegram, Discord, Slack, or another platform |
| I want a local or self-hosted model | `hermes model` → custom endpoint | Verify the endpoint, model name, and context length |
| I want multi-provider fallback | `hermes model` first | Add routing and fallback only after the base chat works |

**Rule of thumb:** if Hermes cannot complete a normal chat, do not add more features yet. Get one clean conversation working first, then layer on gateway, cron, skills, voice, or routing.

---

## 1. Install Hermes Agent

### With the Hermes Desktop installer (recommended)

To easily install the command-line and desktop applications, [download the Hermes Desktop installer](https://hermes-agent.nousresearch.com/) from the official website and run it.

### Without Hermes Desktop

For a command-line only install:

#### Linux / macOS / WSL2 / Android (Termux)

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

#### Windows (native)

Run in PowerShell:

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

After it finishes, reload your shell:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

## 2. Choose a Provider

The single most important setup step. Use `hermes model` to walk through the choice interactively:

```bash
hermes model
```

Easiest path — **Nous Portal**: one subscription covers 300+ models plus the Tool Gateway (web search, image generation, TTS, cloud browser). On a fresh install:

```bash
hermes setup --portal
```

### Good Defaults

| Provider | What it is | How to set up |
| --- | --- | --- |
| **Nous Portal** | Subscription-based, zero-config | OAuth login via `hermes model` |
| **OpenAI Codex** | ChatGPT OAuth, uses Codex models | Device code auth via `hermes model` |
| **Anthropic** | Claude models (Max plan or API key) | `hermes model` → OAuth or API key |
| **OpenRouter** | Multi-provider routing | Enter API key |
| **Fireworks AI** | Direct OpenAI-compatible API | `FIREWORKS_API_KEY` |
| **Z.AI** | GLM / Zhipu models | `GLM_API_KEY` / `ZAI_API_KEY` |
| **DeepSeek** | Direct DeepSeek API | `DEEPSEEK_API_KEY` |
| **Hugging Face** | 20+ open models via unified router | `HF_TOKEN` |
| **AWS Bedrock** | Claude, Nova, Llama via Converse API | IAM role or `aws configure` |
| **Azure Foundry** | Azure AI Foundry-hosted models | `AZURE_FOUNDRY_API_KEY` + base URL |
| **Google AI Studio** | Gemini models | `GOOGLE_API_KEY` |
| **xAI** | Grok models | `XAI_API_KEY` |
| **Custom Endpoint** | vLLM, SGLang, Ollama, or any OpenAI-compatible API | Set base URL + API key |

> Hermes requires a model with at least **64,000 tokens** of context. Most hosted models meet this easily.

### How Settings Are Stored

- **Secrets and tokens** → `~/.hermes/.env`
- **Non-secret settings** → `~/.hermes/config.yaml`

```bash
hermes config set model anthropic/claude-opus-4.6
hermes config set terminal.backend docker
hermes config set OPENROUTER_API_KEY sk-or-...
```

## 3. Run Your First Chat

```bash
hermes            # classic CLI
hermes --tui      # modern TUI (recommended)
```

You'll see a welcome banner with your model, available tools, and skills. Use a prompt that's specific and easy to verify:

```text
Summarize this repo in 5 bullets and tell me what the main entrypoint is.
```

**What success looks like:**

- The banner shows your chosen model/provider
- Hermes replies without error
- It can use a tool if needed (terminal, file read, web search)
- The conversation continues normally for more than one turn

## 4. Verify Sessions Work

```bash
hermes --continue    # Resume the most recent session
hermes -c            # Short form
```

## 5. Try Key Features

### Terminal access

```text
❯ What's my disk usage? Show the top 5 largest directories.
```

### Slash commands

Type `/` to see an autocomplete dropdown:

| Command | What it does |
| --- | --- |
| `/help` | Show all available commands |
| `/tools` | List available tools |
| `/model` | Switch models interactively |
| `/save` | Save the conversation |

### Multi-line input

Press `Alt+Enter`, `Ctrl+J`, or `Shift+Enter` to add a new line.

### Interrupt the agent

Type a new message and press Enter — it interrupts the current task. `Ctrl+C` also works.

## 6. Add the Next Layer

Only after the base chat works:

### Bot or messaging gateway

```bash
hermes gateway setup
```

Connect Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant, or Microsoft Teams.

### Sandboxed terminal

```bash
hermes config set terminal.backend docker    # Docker isolation
hermes config set terminal.backend ssh       # Remote server
```

### Voice mode

```bash
cd ~/.hermes/hermes-agent
uv pip install -e ".[voice]"
```

Then in the CLI: `/voice on`. Press `Ctrl+B` to record.

### Skills

```bash
hermes skills browse                      # list everything available
hermes skills search kubernetes           # find skills by keyword
hermes skills install openai/skills/k8s   # install one
```

### MCP servers

```yaml
# Add to ~/.hermes/config.yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_xxx"
```

---

## Common Failure Modes

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Empty or broken replies | Provider auth or model selection wrong | Run `hermes model` again |
| Custom endpoint returns garbage | Wrong base URL or model name | Verify in a separate client first |
| Gateway starts but nobody can message | Bot token or allowlist incomplete | Re-run `hermes gateway setup` |
| `--continue` can't find session | Switched profiles or never saved | Check `hermes sessions list` |
| Model unavailable or odd fallback | Routing settings too aggressive | Keep routing off until base is stable |

## Recovery Toolkit

1. `hermes doctor`
2. `hermes model`
3. `hermes setup`
4. `hermes sessions list`
5. `hermes --continue`
6. `hermes gateway status`

## Quick Reference

| Command | Description |
| --- | --- |
| `hermes` | Start chatting |
| `hermes model` | Choose your LLM provider and model |
| `hermes tools` | Configure which tools are enabled per platform |
| `hermes setup` | Full setup wizard |
| `hermes doctor` | Diagnose issues |
| `hermes update` | Update to latest version |
| `hermes gateway` | Start the messaging gateway |
| `hermes --continue` | Resume last session |

---

*Source: [Hermes Agent Documentation](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart)*
