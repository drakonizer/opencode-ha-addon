# OpenCode HA Add-on Documentation

## Overview

This add-on runs [OpenCode](https://opencode.ai) inside Home Assistant, accessible directly from the sidebar. It includes automatic integration with Home Assistant via the [hass-mcp](https://github.com/seanblanchfield/hass-mcp) MCP server, giving the AI assistant full access to your HA entities, automations, and configuration.

## Configuration

### Provider

Set `provider` to one of: `anthropic`, `openai`, `google`, or `ollama`.

OpenCode supports additional providers — see [OpenCode docs](https://opencode.ai/docs) for advanced configuration.

### Options

| Option | Required | Description |
|---|---|---|
| `provider` | Yes | LLM provider (`anthropic`, `openai`, `google`, `ollama`) |
| `api_key` | Varies | API key for cloud providers (not needed for Ollama) |
| `model` | No | Model ID (auto-detected from provider if empty) |
| `small_model` | No | Fast model for simple tasks (auto-detected if empty) |
| `ollama_host` | If Ollama | URL of the Ollama server (e.g. `http://homeassistant:11434`) |
| `ollama_keep_alive` | No | How long Ollama keeps models loaded in VRAM (e.g. `5m`, `24h`, `-1` for forever) |
| `github_token` | No | GitHub Personal Access Token — enables the GitHub MCP server so the AI can create repos, manage PRs, and push code |

### Example: Anthropic

```yaml
provider: anthropic
api_key: sk-ant-api03-...
```

### Example: OpenAI

```yaml
provider: openai
api_key: sk-...
model: openai/gpt-4o
```

### Example: Ollama (local, no API key)

```yaml
provider: ollama
ollama_host: "http://homeassistant:11434"
model: "qwen3:8b"
small_model: "qwen3:8b"
ollama_keep_alive: "5m"
```

Ollama requires a running Ollama server accessible from the add-on container. Common `ollama_host` values:

- `http://homeassistant:11434` — Ollama on same machine (most common)
- `http://192.168.x.x:11434` — Ollama on another machine on your LAN
- `http://<addon-slug>:11434` — Ollama running as another HA add-on

The `ollama_keep_alive` option controls how long models stay loaded in GPU/RAM after a request. Set to `-1` to keep loaded indefinitely, or use values like `5m`, `1h`, `24h`. Defaults to Ollama's built-in setting (5 minutes) if not specified.

## How It Works

1. **nginx reverse proxy** on port 8099 (the ingress port) forwards requests to OpenCode on 127.0.0.1:19876
2. **sub_filter rules** rewrite OpenCode's absolute paths (`/assets/...`) to work under HA's ingress subpath
3. **hass-mcp** is auto-configured using the Supervisor token — no manual HA token setup needed
4. **Git tracking** is automatically initialized in `/config` (your HA config directory)
5. The **ingress path** is discovered dynamically from the Supervisor API at startup — no hardcoded paths

## Working Directory

OpenCode opens with `/config` as its working directory — this is your Home Assistant configuration folder. From here the AI can read and edit:

- `configuration.yaml`
- `automations.yaml`
- `scripts.yaml`
- `scenes.yaml`
- Custom components, dashboards, and more

## First Launch

1. Click **OpenCode** in the HA sidebar
2. Click the hamburger menu (top-left) > **Open project**
3. Type `/config` in the search box and select it
4. Start a conversation — the AI has full access to your HA config files and entities

## Network

This add-on uses **HA ingress** — no port forwarding required. It is accessible from anywhere you can reach your HA instance (local network, Nabu Casa, DuckDNS + SSL, etc.).

## Support

Report issues at: https://github.com/drakonizer/opencode-ha-addon/issues
