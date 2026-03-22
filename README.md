# Hermes Agent Home Assistant Add-on

[Hermes Agent](https://hermes-agent.nousresearch.com/) packaged as a [Home Assistant](https://home-assistant.io/) add-on/app. Persistent AI agent with memory, self-improving skills, multi-platform messaging, and a plugin architecture for custom tools.

> The self-improving AI agent built by [Nous Research](https://nousresearch.com/). Home Assistant Add-on by [Wolfram Ravenwolf](https://x.com/WolframRvnwlf).

## Features

- **Persistent memory** -- SQLite FTS5 long-term memory that survives restarts
- **Self-improving skills** -- agent learns and creates new capabilities over time
- **Multi-platform messaging** -- Telegram, Discord, WhatsApp, and more via the gateway
- **OpenAI-compatible API** -- connect any chat frontend (Open WebUI, SillyTavern, etc.) via `/v1/`
- **Plugin architecture** -- custom tools, commands, and hooks without forking
- **Self-modifiable source** -- editable install lets the agent read and modify its own code
- **Persistent web terminal** -- full CLI access via tmux-backed ttyd through the HA sidebar
- **HTTP + HTTPS** -- direct LAN access with auto-generated TLS certificates
- **Full persistence** -- source code, venv, Homebrew, npm, Go, and all agent data survive addon updates

## Installation

1. Add this repository to Home Assistant: **Settings > Apps > Install app > ⋮ > Repositories**
2. Paste the repository URL and click **Add**
3. Find **Hermes Agent** in the store and click **Install**
4. Start the addon and open **Hermes Agent** from the sidebar
5. The setup wizard runs automatically -- configure your model and API keys

## Configuration

Addon-level options are configured in the HA UI (Settings > Apps > Hermes Agent > Configuration):

| Option                | Default                           | Description                                                                     |
| --------------------- | --------------------------------- | ------------------------------------------------------------------------------- |
| `git_url`             | NousResearch repo                 | Git repository URL (clear to reset to default)                                  |
| `git_ref`             | *(empty)*                         | Branch, tag, or commit (empty = repo's default branch)                          |
| `git_token`           |                                   | Token for private repos + exported as `GITHUB_TOKEN` for gh CLI                 |
| `auto_update`         | `false`                           | Pull latest changes on restart (preserves local modifications)                  |
| `hass_url`            | `http://homeassistant.local:8123` | Home Assistant URL for API access                                               |
| `homeassistant_token` |                                   | Long-lived access token for HA API integration                                  |
| `hermes_home`         | `.hermes`                         | Agent profile directory (relative to ~). Change to switch profiles (e.g. "amy") |
| `prefer_ipv4_dns`     | `true`                            | Prioritize IPv4 over IPv6 for DNS resolution                                    |
| `env_vars`            | `[]`                              | Additional environment variables (API keys, etc.)                               |

Hermes-internal configuration (model, platforms, memory, tools) is managed via the terminal:

```bash
hermes setup          # Interactive first-time setup
hermes config edit    # Edit config directly
hermes doctor         # Diagnostics and dependency check
hermes gateway setup  # Configure messaging platforms
```

## Access

The addon provides multiple access paths:

| Path                       | Description                                                              |
| -------------------------- | ------------------------------------------------------------------------ |
| **HA Sidebar**             | Landing page with embedded terminal, mode switching, status              |
| `/hermes/`                 | Hermes Agent (login shell -- starts hermes, crash drops to shell)        |
| `/terminal/`               | Shell terminal (non-login shell -- plain shell, hermes not auto-started) |
| `/cert/ca.crt`             | CA certificate download (for trusting self-signed HTTPS)                 |

**OpenAI-compatible API** (connect Open WebUI, SillyTavern, etc.):

| Endpoint                      | Method | Description                                       |
| ----------------------------- | ------ | ------------------------------------------------- |
| `/v1/chat/completions`        | POST   | Chat Completions (stateless)                      |
| `/v1/responses`               | POST   | Responses API (stateful via previous_response_id) |
| `/v1/responses/{response_id}` | GET    | Retrieve a stored response                        |
| `/v1/responses/{response_id}` | DELETE | Delete a stored response                          |
| `/v1/models`                  | GET    | List available models                             |
| `/health`                     | GET    | Health check                                      |

**Ports:**

| Port     | Description                                          |
| -------- | ---------------------------------------------------- |
| **8080** | HTTP access (all paths above)                        |
| **8443** | HTTPS access (same paths, TLS with self-signed cert) |

Both ports are configurable in the HA addon network settings. Use HTTPS (8443) for secure access. The HTTP port (8080) is intended for TLS-terminating reverse proxies (Cloudflare, NPM, Caddy, etc.).

### TLS Certificates

On first start, self-signed certificates are auto-generated in `~/.certs/`. To use your own:

1. Stop the addon
2. Replace `~/.certs/server.crt` and `~/.certs/server.key` with your own
3. Optionally replace `~/.certs/ca.crt` if you have a custom CA
4. Start the addon

The addon will use existing certificates and never overwrite them.

## Architecture

Three services in a Debian Bookworm container:

1. **Hermes Gateway** (`hermes gateway run`) -- persistent AI agent daemon with OpenAI-compatible API server and messaging platform connectors
2. **ttyd** (x2) -- web terminals backed by persistent tmux sessions (`hermes` + `terminal`)
3. **nginx** -- HTTP, HTTPS, and HA ingress proxy routing to terminal + API

### Shell Environment

Login shells start Hermes automatically. Non-login shells provide a plain shell with all paths configured.

| File                | Persistent? | Purpose                                         |
| ------------------- | ----------- | ----------------------------------------------- |
| `~/.hermes_profile` | Regenerated | Env vars, PATH, tokens (from addon config)      |
| `~/.bashrc`         | Yes         | Sources .hermes_profile + .env, prompt, aliases |
| `~/.profile`        | Yes         | Sources .bashrc, starts hermes (login shells)   |
| `~/.bash_aliases`   | Yes (user)  | Custom aliases and functions                    |
| `~/.tmux.conf`      | Yes         | Terminal config (mouse scroll, history)         |

### Persistent Storage

`~` is `/config/` (addon-isolated via `addon_config`). Everything survives addon updates and is included in HA backups:

```
~ (/config/)
├── .certs/                # TLS certificates (auto-generated or custom)
├── .go/                   # Go workspace
├── .hermes/               # HERMES_HOME (matches official installer layout)
│   ├── hermes-agent/      # Git clone (source code, agent-modifiable)
│   │   └── venv/          # Python venv (editable install)
│   ├── logs/              # Gateway logs
│   ├── memories/          # Long-term memory (MEMORY.md, USER.md)
│   ├── sessions/          # Conversation state
│   ├── skills/            # Auto-created + installed skills
│   ├── .env               # API keys (chmod 600)
│   ├── SOUL.md            # Agent personality
│   ├── config.yaml        # Hermes config (model, platforms, tools)
│   └── state.db           # SQLite FTS5 state
├── .linuxbrew/            # Homebrew
├── .npm-global/           # npm global packages
├── .bash_aliases          # Custom aliases and functions (optional, user-created)
├── .bashrc                # Shell config
├── .hermes_install        # Install marker
├── .hermes_profile        # Env vars + PATH (regenerated)
├── .profile               # Login shell config (starts hermes)
└── .tmux.conf             # tmux config
```

### Container Toolchain

Pre-installed at build time:

- **Languages**: Go 1.22, Node.js 22, Python 3.11
- **Browser**: Chromium, agent-browser
- **Dev tools**: bat, fd-find, gh (GitHub CLI), git, jq, nano, ripgrep, tree, vim
- **Media**: ffmpeg
- **Networking**: curl, dnsutils, netcat, openssh-client, wget
- **Package managers**: go, Homebrew (Linuxbrew), npm, uv
- **System**: bash-completion, command-not-found, rsync, sqlite3, tmux, unzip/zip

## SSH Access

Connect to the Hermes session (login shell -- starts hermes):

```bash
ssh -tp <port> root@<ha-host> "docker exec -it addon_<slug>_hermes_agent tmux -u new -A -s hermes /usr/bin/bash -l"
```

Connect to the terminal session (plain shell):

```bash
ssh -tp <port> root@<ha-host> "docker exec -it addon_<slug>_hermes_agent tmux -u new -A -s terminal /usr/bin/bash"
```

## Supported Architectures

- `amd64`
- `aarch64`

## License

This addon is [MIT licensed](LICENSE). Hermes Agent itself is also [MIT licensed](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
