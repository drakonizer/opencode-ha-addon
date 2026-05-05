# Changelog

## [1.1.0] - 2026-05-05

### Added
- **Ollama provider support** — run local LLMs with no API key or cloud dependency
- New config options: `ollama_host` (server URL) and `ollama_keep_alive` (model memory retention)
- Automatic model map generation from `model`/`small_model` options
- Validation: add-on exits with clear error if `ollama_host` is missing when provider is `ollama`
- Recommended models table in README (qwen3:8b, qwen3:14b, llama3.2:3b, deepseek-coder-v2:16b)

## [1.0.0] - 2026-05-01

### Added
- Initial release
- OpenCode web UI accessible via HA sidebar (ingress)
- Multi-provider support (Anthropic, OpenAI, Google, and more)
- Automatic hass-mcp integration using Supervisor token
- Dynamic ingress path discovery (no hardcoded paths)
- nginx reverse proxy with sub_filter path rewriting
- Automatic git initialization in /config
- amd64 and aarch64 architecture support
