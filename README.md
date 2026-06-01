# Mission Control Agent

Public sandbox agent for Mission Control. It runs on a local Docker sandbox, Railway, or any VM and exposes a bearer-protected WebSocket control plane for PTY, file, git, and SSH setup RPC.

## Install on a VM

With npm / npx:

```sh
npx @agentsystemlabs/mission-control-agent setup
cd mission-control-agent
docker compose up -d --build
```

Or with the shell installer:

```sh
curl -fsSL https://raw.githubusercontent.com/AgentSystemLabs/mission-control-agent/main/scripts/install.sh | bash
```

The setup command prints the generated `MC_AGENT_API_KEY`. Paste that key into Mission Control when creating a **Remote VM** sandbox.

For same-machine testing or SSH tunnels, use:

```txt
ws://localhost:9333
```

For public VM/Railway access, terminate TLS and use `https://` / `wss://`.

## Run Directly

```sh
export MC_AGENT_API_KEY="$(openssl rand -hex 32)"
export MC_WORKSPACE_ROOT=/workspace
mission-control-agent
```

## Railway

Use the **[Railway deploy bundle](./deploy/railway/README.md)** for the recommended setup (persistent volume, health check, template checklist).

Quick version:

1. Deploy from GitHub with **Root Directory** empty (repo root) — not `docker/remote-agent`.
2. Set **Config-as-Code Path** to `deploy/railway/railway.json` (or use root `railway.toml`).
3. Attach a Railway **volume** at **`/home/workspace`** — this must match the container `HOME` (persists `~/.ssh`, Claude/Codex/Cursor auth, and git clones via `/workspace` symlink).

   Do not mount only `/workspace`; agent auth lives under `/home/workspace`.
4. Set `MC_AGENT_API_KEY` (`openssl rand -hex 32`). Do not set `MC_AGENT_PORT`.
5. Generate a public domain and use `wss://…` in Mission Control.

Without a volume, SSH keys and agent CLI login state are lost on every redeploy.

## Coolify (private, via Tailscale)

Use the **[Coolify deploy bundle](./deploy/coolify/README.md)** to run the agent on a
[Coolify](https://coolify.io) server with a persistent volume and **no public domain** —
exposed over your tailnet with `tailscale serve`.

Quick version:

1. **+ New** → **Public Repository** → this repo → **Build Pack: Docker Compose**.
2. **Compose Location** → `/deploy/coolify/docker-compose.yaml`. Deploy.
3. Copy the auto-generated `MC_AGENT_API_KEY` from **Environment Variables**; confirm the
   `mc-agent-home` volume is at **`/home/workspace`**.
4. On the host: `tailscale serve --bg --https=8443 127.0.0.1:9333` (use `8443`, not `443` —
   Checkmate already owns 443).
5. In Mission Control, create a **Remote VM** sandbox with `wss://<host>.<tailnet>.ts.net:8443`.

## Local Mission Control Docker Image

Mission Control's private desktop app builds its local sandbox image from this package's Dockerfile:

```txt
docker/sandbox-base/Dockerfile
```

That keeps local Docker sandboxes and remote VM sandboxes on the same published agent runtime.

## Environment

| Var | Default | Purpose |
| --- | --- | --- |
| `MC_AGENT_PORT` | `PORT` or `9333` | WS + health port |
| `PORT` | empty | Railway-provided port fallback |
| `MC_AGENT_BIND_HOST` | `0.0.0.0` | Bind address; use `::` for dual-stack private networks |
| `MC_WORKSPACE_ROOT` | `/workspace` | Confinement root for spawns + RPC |
| `MC_AGENT_API_KEY` | empty | Required bearer secret for remote deployments |
| `MC_PAIRING_TOKEN` | empty | Local Docker compatibility fallback for the same bearer secret |
| `MC_HOOK_API_HOST` | `host.docker.internal` | Host the in-container hooks POST to |

## Security

Treat a public agent URL as a privileged shell exposed to the internet. Anyone with the API key can run commands, read/write workspace files, clone repos, and use credentials stored in the remote agent home directory.

Recommended posture:

- Use a long random `MC_AGENT_API_KEY` and rotate it if it may have leaked.
- Use `wss://` for public access. Mission Control rejects plaintext `ws://` for public hostnames.
- Prefer SSH tunnels, WireGuard/Tailscale, VPN/VPC, or private networking over a public domain.
- Prefer generated sandbox keys on shared or public remote VMs; copy-host mode can upload your host private keys to the remote over the agent connection.
- `/health` is intentionally unauthenticated and returns only liveness/version.

Railway private networking is reachable only from services in the same Railway project/environment. A desktop app on your laptop needs a tunnel or gateway to reach `*.railway.internal`.

## Development

```sh
npm install
npm run typecheck
npm test
npm run build
```
