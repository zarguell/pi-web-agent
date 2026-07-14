# PI WEB Agent

Run [Pi Coding Agent](https://pi.dev/) with a persistent browser UI via [PI WEB](https://pi-web.dev/), packaged as a single Docker image.

```
zarguell/pi-web-agent:latest
```

## Quick start

```bash
# Create a directory for persistent Pi state
mkdir -p ~/pi-data

# Run the container
docker run -d \
  --name pi-web-agent \
  -p 8504:8504 \
  -v ~/pi-data:/data \
  zarguell/pi-web-agent:latest
```

Open **http://localhost:8504** in your browser.

> The container listens on `0.0.0.0:8504` inside the network. **No built-in auth.** Put an authenticated reverse proxy in front for anything beyond localhost (see [Security](#security)).

## What this is

PI WEB is a web UI for [Pi Coding Agent](https://pi.dev/) by [@jmfederico](https://github.com/jmfederico/pi-web). It provides:

- **Persistent sessions** — agents keep running after you close the browser tab
- **Real workspaces** — agents work inside actual repositories on the filesystem
- **Multi-device control** — laptop, phone, tablet, all pointing at the same agent runtime
- **Session history, file browser, terminal access** — all from the browser

This image bundles both Pi Coding Agent and PI WEB together so you get a working agent environment from a single `docker run`.

## Why this image exists

The official PI WEB [Docker setup](https://github.com/jmfederico/pi-web/tree/main/docker) is comprehensive but opinionated — openSUSE base, systemd, docker-compose, hostexec, fleet management. It's designed for the project's maintainers and contributors.

This image is the **minimal, portable alternative**:

| | Official Docker | This image |
|---|---|---|
| Base | openSUSE Tumbleweed | Debian Bookworm-slim |
| Size | ~1 GB+ | ~250 MB |
| Dependencies | Docker Compose, build pipeline | Single `docker run` |
| Use case | Development / full-featured server | Quick deploy / CI / homelab |
| Architecture | amd64 + arm64 | amd64 + arm64 |

## Usage

### Mount a workspace

Mount a host directory as a workspace so Pi can edit real files:

```bash
docker run -d \
  --name pi-web-agent \
  -p 8504:8504 \
  -v ~/pi-data:/data \
  -v /path/to/your/project:/workspace/my-project \
  zarguell/pi-web-agent:latest
```

Then add `/workspace/my-project` as a project in the PI WEB UI.

### Mount Pi config

Pi stores its config, models, sessions and auth in `~/.pi/agent`. Mount a host directory there to persist sessions across container restarts:

```bash
docker run -d \
  --name pi-web-agent \
  -p 8504:8504 \
  -v ~/pi-data:/data \
  -v ~/.pi:/home/node/.pi \
  zarguell/pi-web-agent:latest
```

### Set API keys

Set your LLM provider key as an environment variable. Pi reads standard variables:

```bash
docker run -d \
  --name pi-web-agent \
  -p 8504:8504 \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -e OPENAI_API_KEY=sk-... \
  zarguell/pi-web-agent:latest
```

### Custom config

PI WEB config lives at `~/.config/pi-web/config.json`. Mount or bake your own:

```json
{
  "host": "0.0.0.0",
  "port": 8504,
  "agent": {
    "command": "pi"
  },
  "spawnSessions": true
}
```

## Volumes

| Path | Purpose |
|---|---|
| `/data` | PI WEB persistent state (projects, settings, session logs) |
| `/home/node/.pi` | Pi coding agent config, auth, sessions, models |
| `/workspace` | Default working directory for agent workspaces |
| `/home/node/.config/pi-web` | PI WEB config file location |

## Ports

| Port | Service |
|---|---|
| `8504` | PI WEB HTTP UI and API |

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `PI_WEB_HOST` | `0.0.0.0` | Bind address (set to `0.0.0.0` for Docker) |
| `PI_WEB_PORT` | `8504` | Web UI port |
| `PI_WEB_SESSIOND_SOCKET` | `/tmp/pi-web-sessiond.sock` | Session daemon Unix socket |
| `PI_WEB_SPAWN_SESSIONS` | `true` | Allow agents to spawn sub-sessions |
| `ANTHROPIC_API_KEY` | — | Anthropic API key (used by Pi) |
| `OPENAI_API_KEY` | — | OpenAI API key |

## Security

**This image provides no authentication.** pi-web is designed for trusted networks. Follow its guidance:

- **Do not** expose port 8504 directly to the public internet
- Use an **SSH tunnel**: `ssh -L 8504:127.0.0.1:8504 user@server`
- Or put an **authenticated reverse proxy** (nginx, Caddy, Traefik) in front
- Or run on a **VPN / private network** (Tailscale, NetBird, WireGuard)

The `PI_WEB_HOST=0.0.0.0` default is intentional: Docker networking is the boundary. Expose the port only to trusted networks via `docker run -p 127.0.0.1:8504:8504` for local-only access.

## Building

```bash
docker build -t pi-web-agent .
```

Versions are pinned via build args:

```bash
docker build \
  --build-arg PI_CODING_AGENT_VERSION=0.80.7 \
  --build-arg PI_WEB_VERSION=1.202607.0 \
  -t pi-web-agent .
```

## Architecture

```
┌─────────────────────────────────┐
│         Docker Container        │
│                                 │
│  pi-web-sessiond  (daemon)      │
│       │                        │
│  pi-web-server    (web UI)      │
│       │                        │
│  pi-coding-agent  (CLI)        │
│       │                        │
│  /workspace  ←── mount your    │
│                 project here   │
└─────────────────────────────────┘
         │ port 8504
         ▼
    Browser ←── authenticated
               reverse proxy
               (optional)
```

## Rationale

### Why npm packages instead of a base image?

Pi Coding Agent is distributed as an [npm package](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) — there is no official Docker image. Building from a pinned npm version is the canonical installation path and gives us:

1. **Reproducible builds** — specific versions with npm integrity hashes
2. **Renovate / Dependabot compatibility** — version bumps via standard PRs
3. **Faster updates** — new Pi releases are npm publishes, not image rebuilds

### Why `0.0.0.0`?

PI WEB's default `127.0.0.1` binding is correct for desktop use but wrong for containers. Docker containers are isolated network namespaces; `0.0.0.0` lets Docker's port mapping (`-p`) control access. The real security boundary is the Docker network and any reverse proxy in front of it.

## Images

Pre-built multi-arch images are published on Docker Hub:

- **`zarguell/pi-web-agent:latest`** — weekly automated builds (Sun 16:50 UTC)
- Architectures: `linux/amd64`, `linux/arm64`

## License

MIT
