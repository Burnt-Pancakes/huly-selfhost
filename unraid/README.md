# Huly on Unraid (rootless, no setup script)

This folder runs the full Huly stack from two files: `huly.yaml` and `huly.env`. No `setup.sh`, `nginx.sh` or generated config is needed. Included services: Love, LiveKit, HulyPulse, Cloudflare Tunnel.

The stack was adapted from the upstream `compose.yml`. It has not been tested end to end, so verify each step below.

## Requirements

- Docker Compose v2.23 or newer (nginx and LiveKit configs use inline `configs.content`)
- 4 vCPUs and 16 GB RAM recommended (upstream minimum: 2 vCPUs, 8 GB)
- A Cloudflare account with a domain and a tunnel token
- For calls from outside your LAN: a playit.gg tunnel for UDP 7882 and TCP 7881, or LiveKit Cloud

## 1. Create a dedicated share

Create an Unraid share named `huly`, preferably cache-only (no mover). Databases perform poorly on the `/mnt/user` FUSE layer, so the env file points at `/mnt/cache/huly`.

```bash
mkdir -p /mnt/cache/huly/{elastic,files,redpanda,cockroach/data,cockroach/certs}
cp huly.yaml /mnt/cache/huly/
cp huly.env.example /mnt/cache/huly/huly.env
```

## 2. Fix ownership for rootless

Each data directory must be owned by the UID its container runs as. Upstream documents UID 1000 for most services and 101 for Redpanda. Add your subuid offset if you run rootless Docker or Podman.

```bash
chown -R 1000:1000 /mnt/cache/huly/{elastic,files,cockroach}
chown -R 101:101 /mnt/cache/huly/redpanda
```

If a container logs permission errors, adjust that directory.

## 3. Configure huly.env

Generate secrets:

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
| `SECURE` | `true` for HTTPS/WSS (Cloudflare Tunnel); empty for plain HTTP |
| `HTTP_BIND` / `HTTP_PORT` | Host bind address and port for nginx. Avoid 80 and 443 on Unraid |
| `CF_TUNNEL_TOKEN` | Cloudflare tunnel token |
| `LIVEKIT_HOST` | Public `wss://` URL of LiveKit signalling |
| `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` | LiveKit credentials shared by `love` and `livekit` |
| `LIVEKIT_NODE_IP` | Public IP of the playit tunnel, advertised to clients for media |
| `SMTP_*` | Mail. Leave blank only if you remove the `mail` service |

## 4. Cloudflare Tunnel routes

The `cloudflared` container joins the stack network, so it reaches services by name:

| Public hostname | Service |
|---|---|
| `huly.example.com` | `http://nginx:80` |
| `livekit.example.com` | `http://livekit:7880` |

Cloudflare Tunnel carries HTTP and WebSocket traffic only. It cannot carry WebRTC media (UDP).

## 5. Calls: LiveKit media path

Signalling goes through the tunnel. Media needs UDP 7882 and TCP 7881 reachable from the internet. Options:

1. **playit.gg**: create UDP and TCP tunnels to the Unraid host. LiveKit advertises its own port, so the public port must match `udp_port` and `tcp_port` in the `livekit_cfg` block. Verify in the playit dashboard that you can choose the public port. Set `LIVEKIT_NODE_IP` to the tunnel's public IP.
2. **LiveKit Cloud**: remove the `livekit` service and `livekit_cfg`, then set `LIVEKIT_HOST`, `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET` to your cloud project values. This is the setup upstream documents.

Test from a phone on cellular data.

## 6. Start

```bash
cd /mnt/cache/huly
docker compose --env-file huly.env -f huly.yaml config   # look for blank variables
docker compose --env-file huly.env -f huly.yaml up -d
docker compose --env-file huly.env -f huly.yaml logs -f account transactor
```

In the Unraid Compose Manager plugin, set the stack's compose file to `huly.yaml` and its env file to `huly.env`. Compose only auto-loads a file named `.env`.

First start takes about a minute. Elasticsearch installs the `ingest-attachment` plugin on boot and needs internet access.

## 7. Disable public sign-up (optional)

Create your first account, then add to both `account` and `front`:

```yaml
      - DISABLE_SIGNUP=true
```

## Differences from upstream compose.yml

- `HTTP_BIND` defaults to `0.0.0.0` so an empty value does not produce an invalid port mapping.
- nginx config is embedded via `configs`; `_love`, `_pulse` and `_print` routes are enabled.
- `love`, `livekit` and `cloudflared` are added; `hulypulse`, `mail` and `notification` are enabled.
- No ports are published except nginx, 7881/tcp and 7882/udp.
- Unused `COCKROACH_SECRET` and `REDPANDA_SECRET` are removed.
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
- **Calls connect but no audio/video**: UDP path or `LIVEKIT_NODE_IP` is wrong.
- **`mail` or `notification` restarting**: SMTP host or VAPID keys are empty.
