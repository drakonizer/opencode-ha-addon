# OpenCode Home Assistant Add-on

Run [OpenCode](https://opencode.ai) as a Home Assistant add-on — an AI coding assistant in your HA sidebar with full access to your entities, automations, and configuration files.

![Home Assistant](https://img.shields.io/badge/Home%20Assistant-add--on-blue?logo=homeassistant)
![License](https://img.shields.io/badge/license-MIT-green)
![Arch](https://img.shields.io/badge/arch-amd64%20%7C%20aarch64-orange)

## What This Does

- **OpenCode in your sidebar** — full AI coding assistant accessible from anywhere you can reach HA (local, Nabu Casa, DuckDNS, etc.)
- **No port forwarding** — uses HA's native ingress system
- **Automatic HA integration** — [hass-mcp](https://github.com/voska/hass-mcp) is pre-configured with the Supervisor token, so the AI can control entities, read states, and manage your smart home out of the box
- **Multi-provider** — works with Anthropic, OpenAI, Google, Ollama (local), and any other provider OpenCode supports
- **Git-tracked config** — automatically initializes git in your `/config` directory

## Quick Start

### 1. Add the repository

In Home Assistant:

1. Go to **Settings > Add-ons > Add-on Store**
2. Click the **...** menu (top right) > **Repositories**
3. Add: `https://github.com/drakonizer/opencode-ha-addon`
4. Click **Close**, then find **OpenCode** in the store and click **Install**

### 2. Configure

Go to the **Configuration** tab and set your provider and API key:

```yaml
provider: anthropic
api_key: sk-ant-api03-...
```

Other providers:

```yaml
# OpenAI
provider: openai
api_key: sk-...

# Google
provider: google
api_key: AIza...

# Ollama (local — no API key needed)
provider: ollama
ollama_host: "http://homeassistant:11434"
model: "qwen3:8b"
```

You can optionally override the model:

```yaml
model: anthropic/claude-sonnet-4-20250514
small_model: anthropic/claude-sonnet-4-20250514
```

### 3. Start

Click **Start**. OpenCode will appear in your HA sidebar.

### 4. Open a project

1. Click **OpenCode** in the sidebar
2. Click the hamburger menu > **Open project**
3. Type `/config` and select it
4. Start chatting — the AI can read your HA config files and control your entities

## How It Works

```
Browser --> HA Ingress Proxy --> nginx (:8099) --> OpenCode (:19876)
                                                        |
                                                        v
                                                    hass-mcp
                                                        |
                                                        v
                                                HA Supervisor API
```

1. HA's ingress proxy forwards requests to nginx inside the add-on container
2. nginx rewrites OpenCode's absolute asset paths to work under the ingress subpath
3. OpenCode serves its web UI and handles AI conversations
4. hass-mcp gives the AI access to HA entities and services via the Supervisor token

The ingress path is **discovered dynamically** at startup from the Supervisor API — nothing is hardcoded.

### Supervisor Token

The `SUPERVISOR_TOKEN` is a short-lived JWT automatically injected by Home Assistant into every add-on container. The add-on never stores or generates it — it's provided as an environment variable at runtime and rotates on each restart. This is the standard authentication mechanism for all HA add-ons.

## Configuration Reference

| Option | Required | Default | Description |
|---|---|---|---|
| `provider` | Yes | `anthropic` | LLM provider (`anthropic`, `openai`, `google`, `ollama`) |
| `api_key` | Varies | — | API key for cloud providers (not needed for Ollama) |
| `model` | No | Auto | Model ID override (auto-detected from provider if empty) |
| `small_model` | No | Auto | Fast model for simple tasks |
| `ollama_host` | If Ollama | — | Ollama server URL (e.g. `http://homeassistant:11434`) |
| `ollama_keep_alive` | No | — | How long Ollama keeps models in memory (e.g. `5m`, `24h`, `-1` for forever) |
| `github_token` | No | — | GitHub Personal Access Token (enables GitHub MCP for repo management) |

### Default Models

| Provider | Default Model |
|---|---|
| Anthropic | `claude-sonnet-4-20250514` |
| OpenAI | `gpt-4o` |
| Google | `gemini-2.0-flash` |
| Ollama | `qwen3:8b` |

OpenCode supports additional providers (Amazon Bedrock, Azure, etc.) — see [OpenCode docs](https://opencode.ai/docs) for advanced provider configuration.

### Ollama Setup

Ollama lets you run LLMs entirely locally — no API keys, no cloud, no costs. You need an Ollama server accessible from the add-on container.

**Common setups:**

| Ollama Location | `ollama_host` value |
|---|---|
| Same machine (host_network add-on or bare metal) | `http://homeassistant:11434` |
| Another machine on LAN | `http://192.168.x.x:11434` |
| Ollama HA add-on (internal port) | `http://<addon-slug>:11434` |

**Recommended models for HA automation:**

| Model | VRAM | Good for |
|---|---|---|
| `qwen3:8b` | ~6 GB | Best balance of speed + capability |
| `qwen3:14b` | ~10 GB | Stronger reasoning |
| `llama3.2:3b` | ~3 GB | Fast, lightweight tasks |
| `deepseek-coder-v2:16b` | ~12 GB | Code-heavy automation |

**Example config (same machine):**

```yaml
provider: ollama
ollama_host: "http://homeassistant:11434"
model: "qwen3:8b"
small_model: "qwen3:8b"
ollama_keep_alive: "5m"
```

## Architecture Support

| Architecture | Platform |
|---|---|
| `amd64` | Intel/AMD x86_64 (NUC, Proxmox, generic x86) |
| `aarch64` | ARM64 (Raspberry Pi 4/5, ODROID) |

## Ingress + nginx: The Hard Part

Getting a modern SPA to work behind HA's ingress proxy is non-trivial. Here's what the nginx layer handles:

- **Asset path rewriting** — OpenCode serves assets at `/assets/...` but ingress mounts the add-on at `/api/hassio_ingress/<token>/`. All `href` and `src` attributes in HTML/JS/CSS are rewritten via `sub_filter`.
- **Vite chunk preloading** — The bundler's dynamic import base path function is patched to prepend the ingress path.
- **API base URL** — OpenCode's client SDK derives its API URL from `location.origin`. This is rewritten to include the ingress subpath.
- **Web Workers** — Worker script references are rewritten to the correct path.
- **CSP stripping** — The Content-Security-Policy header is removed to allow inline scripts.
- **gzip handling** — Responses are decompressed before filtering (`gunzip on`).

These rules are specific to OpenCode's current SPA build and may need updating for major OpenCode version changes.

## Troubleshooting

### White screen after clicking OpenCode in sidebar

- Check add-on logs for errors (Settings > Add-ons > OpenCode > Log)
- Verify your API key is correct
- Try a hard refresh (Ctrl+Shift+R)

### "No folders found" when opening a project

This is expected on first launch. Type `/config` in the search box to navigate to your HA config directory.

### AI can't control entities

The hass-mcp server uses the Supervisor token which is automatically provided. If entity control isn't working:
1. Verify `hassio_api` and `homeassistant_api` are enabled in the add-on config
2. Restart the add-on to refresh the token

## Development

```bash
# Clone into your HA's local add-ons directory
cd /addons/local/
git clone https://github.com/drakonizer/opencode-ha-addon opencode

# In HA: Settings > Add-ons > Add-on Store > ... > Check for updates
# The local add-on will appear under "Local add-ons"
```

## Credits

- [OpenCode](https://opencode.ai) — the AI coding assistant
- [hass-mcp](https://github.com/voska/hass-mcp) — Home Assistant MCP server
- [uv](https://github.com/astral-sh/uv) — Python package manager (runs hass-mcp)

## License

MIT — see [LICENSE](LICENSE)

---

*This entire project was vibecoded — built through conversation with AI, from the nginx ingress hack to this README.*
