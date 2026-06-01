# Deploy Mission Control Agent on Coolify (private, via Tailscale)

This bundle runs the remote agent on a [Coolify](https://coolify.io) server **without
a public domain**. Coolify builds the image, runs the container, and manages a
persistent volume; the agent is published only on the host loopback
(`127.0.0.1:9333`) and reached over your tailnet with **`tailscale serve`** — the
same pattern used for Checkmate on this host.

Persistent storage (the `mc-agent-home` volume at `/home/workspace`) keeps:

- `~/.ssh` — copied or generated Git SSH keys
- `~/.claude`, `~/.codex`, `~/.cursor` — agent CLI auth and config
- `~/workspace` — git clones (via the `/workspace` symlink)

Without the volume, all of that is wiped on every redeploy.

## 1. Deploy in Coolify

1. **+ New** → **Resource** → your Server/Project → **Public Repository**.
2. Repository: `https://github.com/Shironex/mission-control-agent`, branch `main`.
3. **Build Pack** → **Docker Compose**.
4. **Docker Compose Location** → `/deploy/coolify/docker-compose.yaml`.

   The compose uses `build.context: ../..` so the build context is the repo root
   (where the Dockerfile's `COPY package.json src ...` live). This is what
   deploys successfully. If a build can't find `package.json`, your Coolify Base
   Directory is the repo root `/` — in that case change the compose to `context: .`.
5. **Domains** → leave **empty** (no public domain — this is the private variant).
6. **Deploy.** On first deploy Coolify auto-generates `MC_AGENT_API_KEY` (a 64-char
   secret) and lists it under **Environment Variables**.
7. **Storages** → confirm the `mc-agent-home` volume is mounted at **`/home/workspace`**.
8. Open **Environment Variables**, reveal **`MC_AGENT_API_KEY`**, and copy it.

No domain is assigned — there's no `SERVICE_FQDN_*` var, so Coolify never touches
the public proxy.

## 2. Expose it on the tailnet (host shell)

Checkmate already owns `https://<host>.<tailnet>.ts.net` on **port 443**. Only one
service can own `/` on 443, and the agent's WebSocket + `/health` live at root `/`
(no subpath), so give the agent its **own HTTPS port** — `8443`:

```sh
# On the Coolify host (the machine in your tailnet):
tailscale serve --bg --https=8443 127.0.0.1:9333
tailscale serve status
# https://host.tailnet.ts.net:443   -> 127.0.0.1:<checkmate>   (existing)
# https://host.tailnet.ts.net:8443  -> 127.0.0.1:9333          (this agent)
```

`8443` is arbitrary — any free HTTPS serve port other than `443` works. Tailscale
proxies WebSocket upgrades, so `wss://` works through `tailscale serve`.

To remove it later: `tailscale serve --https=8443 off`.

## 3. Connect from Mission Control

In Mission Control → **Sandboxes** → create a **Remote VM** sandbox:

- URL: `wss://<host>.<tailnet>.ts.net:8443`
- API key: the value copied in step 7.

(`tailscale serve` gives a real TLS cert via MagicDNS, so this is `wss://`. If you
ever skip `tailscale serve` and hit the tailnet IP directly, the traffic is still
WireGuard-encrypted and Mission Control accepts plaintext `ws://100.x.y.z:9333`
for private addresses — but you'd need to bind the port to the Tailscale IP
instead of loopback in the compose.)

## Verify

```sh
# from any tailnet device:
curl -s "https://<host>.<tailnet>.ts.net:8443/health"
# {"ok":true,"version":"0.2.1"}
```

## Why no port conflict with Checkmate

| Layer | Checkmate | Agent | Conflict? |
| --- | --- | --- | --- |
| Docker host publish | `127.0.0.1:<checkmate>` | `127.0.0.1:9333` | No — different ports |
| `tailscale serve` HTTPS | `:443` | `:8443` | No — different serve ports |

The only thing you must not do is reuse `443` for the agent's `tailscale serve`.

## Security

Even on a private tailnet, anyone who can reach the URL **and** has
`MC_AGENT_API_KEY` gets a privileged shell. Keep the key secret, rotate it if it
may have leaked, and keep ACLs tight on the tailnet. `/health` is unauthenticated
by design and returns only liveness/version. See the root
[README](../../README.md#security).

## Notes

- **Build context** is `../..` (the repo root), so the Dockerfile's
  `COPY package.json src ...` resolve even though this compose lives under
  `deploy/coolify/`. If a build can't find `package.json`, see step 4 about the
  Coolify Base Directory.
- **Public-domain variant:** to expose this through Coolify's proxy instead of
  Tailscale, drop the `ports:` block and add `- SERVICE_FQDN_AGENT_9333` to
  `environment:`; Coolify then allocates a domain and routes Traefik to 9333.
- **Separate clones volume:** add a second named volume at `/workspace`; the
  entrypoint detects it in `/proc/mounts` and keeps it independent of the home volume.
