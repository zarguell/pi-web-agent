# PI WEB Agent

Run [Pi Coding Agent](https://pi.dev/) with a persistent browser UI via [PI WEB](https://pi-web.dev/), packaged as a single Docker image.

> **⚠️ SECURITY WARNING — read before deploying**
> This container runs as **root** and **binds to `0.0.0.0`** by design. There is **no built-in authentication**. The container boundary IS the security boundary — anyone who reaches port 8504 has root-equivalent code execution inside it.
>
> **Do not** expose port 8504 to the public internet. Use an SSH tunnel, authenticated reverse proxy, or private network (Tailscale/WireGuard). The container assumes a trusted network perimeter — it does not enforce one.
>
> See the [Security section](#-security) for the full tradeoff and deployment guidance.

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

Pi stores its config, models, sessions and auth in `~/.pi/agent`. Mount a host directory there to persist sessions across container restarts (note: container runs as root, so Pi uses `/root/.pi`):

```bash
docker run -d \
  --name pi-web-agent \
  -p 8504:8504 \
  -v ~/pi-data:/data \
  -v ~/.pi:/root/.pi \
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
| `/home/node/.pi` | Pi coding agent config, auth, sessions, models *(mount at `/root/.pi` when running as root)* |
| `/workspace` | Default working directory for agent workspaces |
| `/home/node/.config/pi-web` | PI WEB config file location *(mount at `/root/.config/pi-web` when running as root)* |

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

## 🔒 Security

### The short version

**Do not expose port 8504 to the public internet.** This container is designed for trusted networks only. Always put an authenticated reverse proxy, SSH tunnel, or VPN in front of it.

### Why root?

This container runs as **root** so the AI agent can install system packages (apt), language runtimes, and tools on demand — which is the whole reason to use a containerized agent instead of a static environment.

**Is that dangerous?** Only inside the container. Docker provides the real security boundary:

- **seccomp profiles** restrict syscalls
- **Capability drops** limit what root can do (e.g., no `SYS_ADMIN`)
- **Read-only root filesystem** can be enabled (see below)
- **Host mounts** control what files the agent can touch

The agent (an LLM) can already write arbitrary shell commands, Python scripts, and npm packages — all of which execute arbitrary code inside the container. Denying `apt-get` while allowing `pip install` or `npm install` (which run maintainer scripts during install) is **security theater**. Every package manager is a vector for arbitrary code execution. Containers that block root but allow npm/pip give users a false sense of security while the real attack surface is identical.

**Running as root is honest about the boundary.** The container is the sandbox. Anyone who reaches port 8504 has root-equivalent access inside it — the only question is how far that access reaches.

### Deployment options (choose one)

| Approach | How | Security level |
|---|---|---|
| **Local only** | `docker run -p 127.0.0.1:8504:8504` | Safe for single-user dev machines |
| **SSH tunnel** | `ssh -L 8504:127.0.0.1:8504 user@server` | Safe for remote access |
| **Authenticated reverse proxy** | nginx/Caddy/Traefik with HTTP Basic Auth, OAuth, or Cloudflare Access in front | Production-safe |
| **VPN / private network** | Tailscale, NetBird, WireGuard — bind to the VPN IP | Production-safe |
| **Public internet** | `docker run -p 8504:8504` (0.0.0.0) | **🚫 Do not do this** |

### Additional hardening

If you want a tighter boundary, you can add these at deployment time without changing the image:

```bash
docker run -d \
  --name pi-web-agent \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add DAC_OVERRIDE \
  --cap-add CHOWN \
  --cap-add SETUID \
  --cap-add SETGID \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /root \
  -p 127.0.0.1:8504:8504 \
  zarguell/pi-web-agent:latest
```

This drops all capabilities except the minimum needed for package managers and user-space operation, prevents privilege escalation, and makes the rootfs read-only.

**But understand:** with the agent controlling `npm install`, `pip install`, and any shell command, root inside a hardened container is still powerful. The container boundary + network perimeter are what actually protect you.

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
