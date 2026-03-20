# Hermes Agent Home Assistant Add-on

[NousResearch Hermes Agent](https://github.com/NousResearch/hermes-agent) packaged as a Home Assistant add-on. Persistent AI agent with memory, self-improving skills, multi-platform messaging, and a plugin architecture for custom tools.

## Features

- **Persistent memory** -- SQLite FTS5 long-term memory that survives restarts
- **Self-improving skills** -- agent learns and creates new capabilities over time
- **Multi-platform messaging** -- Telegram, Discord, Home Assistant, and more via the gateway
- **Plugin architecture** -- custom tools, commands, and hooks without forking
- **Self-modifiable source** -- editable install lets the agent read and modify its own code
- **Web terminal** -- full CLI access via ttyd through the HA sidebar
- **Full persistence** -- source code, venv, Homebrew, npm, Go, and all agent data survive addon updates

## Installation

1. Add this repository to Home Assistant: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Paste the repository URL and click **Add**
3. Find **Hermes Agent** in the store and click **Install**
4. Start the addon and open **Hermes Agent** from the sidebar
5. Click **Open Terminal** and run `hermes setup` to configure your model and API keys

## Configuration

Addon-level options are configured in the HA UI (Settings > Add-ons > Hermes Agent > Configuration):

| Option | Default | Description |
|--------|---------|-------------|
| `install_source` | `git` | `git` (editable, self-modifiable) or `pypi` (simple) |
| `git_url` | NousResearch repo | Git repository URL |
| `git_ref` | `main` | Branch, tag, or commit |
| `git_token` | | Token for private repos |
| `auto_update` | `false` | Pull latest on restart |
| `timezone` | `Europe/Berlin` | Container timezone |
| `force_ipv4_dns` | `true` | Prefer IPv4 DNS resolution |
| `enable_terminal` | `true` | Enable web terminal |
| `terminal_port` | `7681` | Direct ttyd port |
| `env_vars` | `[]` | Extra environment variables (API keys, etc.) |

Hermes-internal configuration (model, platforms, memory, tools) is managed via the terminal:

```bash
hermes setup          # Interactive first-time setup
hermes config edit    # Edit config directly
```

## Architecture

Three services in a Debian Bookworm container:

1. **Hermes Gateway** -- persistent AI agent daemon, connects to messaging platforms
2. **ttyd** -- web terminal for CLI access (bound to localhost, exposed via ingress)
3. **nginx** -- ingress reverse proxy routing HA panel traffic

### Persistent Storage

Everything under `/config/` survives addon updates and is included in HA backups:

```
/config/
├── hermes-agent/          # Git clone (agent-modifiable source code)
├── .hermes/               # HERMES_HOME
│   ├── config.yaml        # Hermes config (model, platforms, tools)
│   ├── .env               # API keys
│   ├── SOUL.md            # Agent personality
│   ├── memories/          # Long-term memory (MEMORY.md, USER.md)
│   ├── skills/            # Auto-created + installed skills
│   ├── plugins/           # Custom tools and hooks
│   ├── workspace/         # Persistent docs, data, code
│   ├── state.db           # SQLite FTS5 state
│   └── venv/              # Python venv
├── .linuxbrew/            # Homebrew
├── .node_global/          # npm global packages
└── .go/                   # Go workspace
```

### Container Toolchain

Pre-installed at build time:

- Python 3.11+ (uv), Node.js 22, Go 1.22
- Chromium + Playwright deps, ffmpeg
- Homebrew (Linuxbrew), git, curl, jq, ripgrep, fd, rsync, vim

## Supported Architectures

- `amd64`
- `aarch64`

## License

This addon packaging is provided as-is. Hermes Agent itself is [MIT licensed](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
