# Huly on Unraid (rootless, no setup script)

This folder runs the full Huly stack from two files: `huly.yaml` and `huly.env`. No `setup.sh`, `nginx.sh` or generated config is needed. Included services: Love, LiveKit, HulyPulse, Cloudflare Tunnel, playit.gg agent.

The stack was adapted from the upstream `compose.yml`. It has not been tested end to end, so verify each step below.

## Design

- **No LAN exposure.** No host ports are published. `huly_net` is an internal Docker network (no internet, no host access). The Cloudflare tunnel agent reaches `nginx:80` by name.
- **Egress network.** Only `cloudflared`, `playit`, `elastic` (first-boot plugin download), `mail` (SMTP) and `notification` (browser push) are also on the outbound-only `egress` network.
- **Object storage.** The `silo` service is `pgsty/silo`, a maintained community fork of MinIO (upstream MinIO's community repo is archived). A network alias `minio` is kept. `STORAGE_CONFIG` uses `minio|silo?...`: the `minio|` prefix is the adapter type; the host after it is the service name.
- **Media.** Web traffic uses Cloudflare Tunnel. LiveKit media uses a playit.gg UDP and TCP tunnel to the `livekit` container at the fixed address `172.30.0.10`.

## Requirements

- Docker Compose v2.23 or newer (inline `configs.content`)
- 4 vCPUs and 16 GB RAM recommended (upstream minimum: 2 vCPUs, 8 GB)
- A Cloudflare account with a domain and a tunnel token
- A playit.gg account for calls from outside your LAN, or LiveKit Cloud instead

## 1. Create a dedicated share

Create an Unraid share named `huly`, preferably cache-only (no mover). Databases perform poorly on the `/mnt/user` FUSE layer, so the env file points at `/mnt/cache/huly`.

```bash
mkdir -p /mnt/cache/huly/{elastic,files,redpanda,cockroach/data,cockroach/certs}

curl -fsSL https://raw.githubusercontent.com/Burnt-Pancakes/huly-selfhost/refs/heads/main/unraid/docker-compose.yml -o /mnt/cache/huly/docker-compose.yml

curl -fsSL https://raw.githubusercontent.com/Burnt-Pancakes/huly-selfhost/refs/heads/main/unraid/.env -o /mnt/cache/huly/.env

```

## 2. Fix ownership for rootless

Each data directory must be owned by the UID its container runs as. Upstream documents UID 1000 for most services and 101 for Redpanda. Add your subuid offset if you run rootless Docker or Podman.

```bash
chown -R 1000:1000 /mnt/cache/huly/{elastic,files,cockroach}
chown -R 101:101 /mnt/cache/huly/redpanda
```

If a container logs permission errors, adjust that directory.

## 3. Configure huly.env

```bash
openssl rand -hex 32   # SECRET, CR_USER_PASSWORD, REDPANDA_ADMIN_PWD, LIVEKIT_API_SECRET
npx web-push generate-vapid-keys   # PUSH_PUBLIC_KEY / PUSH_PRIVATE_KEY
```

`CR_USER_PASSWORD` must be repeated inside `CR_DB_URL`:

```env
CR_USER_PASSWORD=<hex>
CR_DB_URL=postgres://selfhost:<hex>@cockroach:26257/defaultdb
```

| Variable | Purpose |
|---|---|
| `HOST_ADDRESS` | Public hostname, no protocol or trailing slash |
| `SECURE` | `true` for HTTPS/WSS (Cloudflare Tunnel) |
| `CF_TUNNEL_TOKEN` | Cloudflare tunnel token |
| `LIVEKIT_HOST` | Internal LiveKit URL used by `love` (`http://livekit:7880`) |
| `LIVEKIT_WS` | Public `wss://` URL used by browsers |
| `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | LiveKit credentials shared by `love` and `livekit` |
| `PLAYIT_SECRET_KEY` | Secret of the playit Docker agent |
| `PLAYIT_IMAGE` | Agent image; the default name is unverified, copy it from the playit dashboard wizard |
| `LIVEKIT_UDP_PORT` / `LIVEKIT_TCP_PORT` | The public ports playit assigned to your tunnels; LiveKit listens on the same numbers |
| `LIVEKIT_NODE_IP` | Public IP of the playit tunnel |
| `SMTP_*` | Mail. Leave blank only if you remove the `mail` service |

## 4. Cloudflare Tunnel routes

| Public hostname | Service |
|---|---|
| `huly.example.com` | `http://nginx:80` |
| `livekit.example.com` | `http://livekit:7880` |

Cloudflare Tunnel carries HTTP and WebSocket traffic only. It cannot carry WebRTC media (UDP).

## 5. Calls: playit.gg and LiveKit

LiveKit advertises its own local port to clients and has no separate external-port setting, so the container must listen on the port playit assigns. That means the tunnel comes up first.

1. Create a Docker agent in the playit dashboard and set `PLAYIT_SECRET_KEY`.
2. Start only the agent: `docker compose --env-file huly.env -f huly.yaml up -d playit`.
3. In the dashboard create one UDP tunnel and one TCP tunnel. Note each assigned public port.
4. Set `LIVEKIT_UDP_PORT` and `LIVEKIT_TCP_PORT` to those numbers.
5. Edit each tunnel's origin to `172.30.0.10:<same port>`.
6. Set `LIVEKIT_NODE_IP` to the tunnel's public IP (`dig +short <tunnel hostname>`).
7. Start the rest of the stack.

If you cannot choose or match ports, use LiveKit Cloud instead: remove the `livekit` service and `livekit_cfg`, then set `LIVEKIT_HOST`, `LIVEKIT_WS`, `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET` to your cloud project values.

Test from a phone on cellular data.

## 6. Start

```bash
cd /mnt/cache/huly
docker compose --env-file huly.env -f huly.yaml config   # look for blank variables
docker compose --env-file huly.env -f huly.yaml up -d
docker compose --env-file huly.env -f huly.yaml logs -f account transactor
```

In the Unraid Compose Manager plugin, set the stack's compose file to `huly.yaml` and its env file to `huly.env`. Compose only auto-loads a file named `.env`.

First start takes about a minute. Elasticsearch installs the `ingest-attachment` plugin on boot and needs the `egress` network.

Because no ports are published, test through the Cloudflare hostname. For a temporary LAN test, add `ports: ["127.0.0.1:8087:80"]` to `nginx` and attach it to `egress` as well.

## 7. Disable public sign-up (optional)

Create your first account, then add to both `account` and `front`:

```yaml
      - DISABLE_SIGNUP=true
```

## Differences from upstream compose.yml

- Service `minio` renamed `silo` (alias `minio` kept); `STORAGE_CONFIG` host is `silo`.
- nginx config is embedded via `configs`; `_love`, `_pulse` and `_print` routes are enabled.
- `love`, `livekit`, `playit` and `cloudflared` are added; `hulypulse`, `mail` and `notification` are enabled.
- No host ports are published; `huly_net` is internal.
- Unused `COCKROACH_SECRET`, `REDPANDA_SECRET`, `HTTP_BIND` and `HTTP_PORT` are removed.
- Routes for calendar, gmail, telegram, github, export and aibot are not proxied.

## Upgrading

Read `MIGRATION.md` in the repo root first, take a backup, change `HULY_VERSION` and `DESKTOP_CHANNEL` in `huly.env`, then:

```bash
docker compose --env-file huly.env -f huly.yaml pull
docker compose --env-file huly.env -f huly.yaml up -d
```

## Troubleshooting

- **Sign-up fails**: `docker compose logs -f account`.
- **502 from nginx**: services are still starting, or the resolver `127.0.0.11` is wrong. That address is Docker's embedded DNS; Podman needs a different one.
- **File uploads fail**: check `docker compose logs silo` and that the `STORAGE_CONFIG` host (`silo`) resolves.
- **Calls connect but no audio/video**: the playit origin, the matching ports, or `LIVEKIT_NODE_IP` is wrong.
- **`mail` or `notification` restarting**: SMTP host or VAPID keys are empty.
