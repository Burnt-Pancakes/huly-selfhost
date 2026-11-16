# Huly on Unraid (rootless, no setup script)

Two files run the whole stack: `docker-compose.yml` and `.env`. There is no `setup.sh`, `nginx.sh` or generated config. Included: Love and LiveKit (calls), HulyPulse (live presence), Cloudflare Tunnel, and a playit.gg agent for call media.

Adapted from the upstream `compose.yml`. It has not been tested end to end, so verify each step.

**The `.env` file explains every variable inside itself.** Read it top to bottom; lines marked `<== EDIT` must be changed.

## How it fits together

- **No LAN exposure.** No host ports are published. The `huly_net` network is internal (no internet, no host access), and the Cloudflare tunnel agent reaches `nginx:80` by name. Docker picks the subnet itself; nothing is hardcoded.
- **Egress network.** Only `cloudflared`, `playit`, `elastic` (first-boot plugin download), `mail` (SMTP) and `notification` (browser push) also join the outbound-only `huly_egress` network.
- **Object storage.** The service is named `minio` so every hard-coded path keeps working, but the image is `pgsty/silo`, a maintained community fork of MinIO (the upstream MinIO community repo is archived).
- **Calls.** Web traffic uses Cloudflare Tunnel. LiveKit media uses a playit.gg UDP and TCP tunnel whose local address is `livekit:<port>` (the container name).

## Requirements

- Docker Compose v2.23 or newer (the nginx and LiveKit configs are inline).
- 4 vCPUs and 16 GB RAM recommended (upstream minimum: 2 vCPUs, 8 GB).
- A Cloudflare account, a domain, and a tunnel token.
- A playit.gg account for calls from outside your LAN, or use LiveKit Cloud instead.

## 1. Download the two files

Create an Unraid share named `huly`, ideally cache-only. Databases are slow on the `/mnt/user` FUSE layer, so the paths use `/mnt/cache/huly`.

```bash
mkdir -p /mnt/cache/huly/{elastic,files,redpanda,cockroach/data,cockroach/certs}
cd /mnt/cache/huly
curl -fsSL https://raw.githubusercontent.com/Burnt-Pancakes/huly-selfhost/refs/heads/main/unraid/docker-compose.yml -o docker-compose.yml
curl -fsSL https://raw.githubusercontent.com/Burnt-Pancakes/huly-selfhost/refs/heads/main/unraid/.env -o .env
```

`.env` is a hidden file; use `ls -a` to see it. Compose reads a file called `.env` in the same folder automatically, so no extra flags are needed.

## 2. Fix ownership (rootless)

Each data folder must belong to the UID its container runs as. Upstream documents UID 1000 for most services and 101 for Redpanda. Add your subuid offset if you run rootless Docker or Podman.

```bash
chown -R 1000:1000 /mnt/cache/huly/{elastic,files,cockroach}
chown -R 101:101 /mnt/cache/huly/redpanda
```

If a container logs "permission denied", fix the owner of that folder.

## 3. Fill in `.env`

Generate all the random secrets in one go. This block rewrites the matching lines in `.env`:

```bash
cd /mnt/cache/huly
CRPW=$(openssl rand -hex 32)
sed -i \
  -e "s|^SECRET=.*|SECRET=$(openssl rand -hex 32)|" \
  -e "s|^CR_USER_PASSWORD=.*|CR_USER_PASSWORD=$CRPW|" \
  -e "s|^CR_DB_URL=.*|CR_DB_URL=postgres://selfhost:$CRPW@cockroach:26257/defaultdb|" \
  -e "s|^REDPANDA_ADMIN_PWD=.*|REDPANDA_ADMIN_PWD=$(openssl rand -hex 32)|" \
  -e "s|^LIVEKIT_API_KEY=.*|LIVEKIT_API_KEY=API$(openssl rand -hex 8)|" \
  -e "s|^LIVEKIT_API_SECRET=.*|LIVEKIT_API_SECRET=$(openssl rand -hex 32)|" .env
```

Notes:

- `CR_USER_PASSWORD` must also appear inside `CR_DB_URL`; the block above keeps them identical. If you set them by hand, make them match.
- `LIVEKIT_API_KEY` is only a name, not a password. Any text works; the block uses `API` plus random hex so it is always read as text. `LIVEKIT_API_SECRET` is the password for that key.
- Create the web-push key pair (`PUSH_PUBLIC_KEY`, `PUSH_PRIVATE_KEY`) on the Unraid terminal, then paste the two values into `.env`:

```bash
docker run --rm node:20-alpine npx -y web-push generate-vapid-keys
```

Then edit the remaining `<== EDIT` lines by hand: `HOST_ADDRESS`, `CF_TUNNEL_TOKEN`, `MAIL_SOURCE`, the `SMTP_*` values and `LIVEKIT_WS`.

### Variables at a glance

Full explanations are in `.env` next to each value.

| Variable | Purpose |
|---|---|
| `HOST_ADDRESS` | Public hostname, no `https://`, no trailing slash |
| `SECURE` | `true` for HTTPS/WSS (always, with Cloudflare Tunnel) |
| `CF_TUNNEL_TOKEN` | Cloudflare Tunnel token |
| `SECRET` | Shared secret the services use to sign tokens |
| `CR_USER_PASSWORD`, `CR_DB_URL` | Database password and connection string (must match) |
| `REDPANDA_ADMIN_PWD` | Message queue admin password |
| `VOLUME_*_PATH` | Where each service keeps data on your share |
| `SMTP_*`, `MAIL_SOURCE` | Outgoing mail |
| `PUSH_PUBLIC_KEY`, `PUSH_PRIVATE_KEY` | Web push key pair |
| `LIVEKIT_HOST` | Internal LiveKit URL used by `love` (leave as is) |
| `LIVEKIT_WS` | Public `wss://` URL browsers use |
| `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | LiveKit key name and its password |
| `PLAYIT_SECRET_KEY`, `PLAYIT_IMAGE` | playit.gg agent secret and image |
| `LIVEKIT_UDP_PORT`, `LIVEKIT_TCP_PORT` | Public ports assigned by playit |
| `LIVEKIT_NODE_IP` | Public IP of the playit tunnel |

## 4. Cloudflare Tunnel routes

In the tunnel's Public Hostnames:

| Public hostname | Service |
|---|---|
| `huly.example.com` | `http://nginx:80` |
| `livekit.example.com` | `http://livekit:7880` |

Cloudflare Tunnel carries HTTP and WebSocket traffic only. It cannot carry call media (UDP). Protect both hostnames with your access rule (for example a client-certificate policy) before you start `cloudflared`.

## 5. Calls: playit.gg and LiveKit

LiveKit tells browsers which port it listens on and has no separate external-port setting, so the container must listen on the port playit assigns. The tunnel therefore comes first:

1. Create a Docker agent in the playit dashboard and put its secret in `PLAYIT_SECRET_KEY`.
2. Start only the agent: `docker compose up -d playit`.
3. In the dashboard create one UDP tunnel and one TCP tunnel to that agent. Note each assigned public port.
4. Put those numbers in `LIVEKIT_UDP_PORT` and `LIVEKIT_TCP_PORT`.
5. Edit each tunnel's local address to `livekit:<the same port number>`.
6. Put the tunnel's public IP in `LIVEKIT_NODE_IP` (`dig +short <tunnel hostname>`).
7. Start the rest in step 6.

If you cannot match ports, use LiveKit Cloud: remove the `livekit` service and the `livekit_cfg` block from `docker-compose.yml`, then set `LIVEKIT_HOST`, `LIVEKIT_WS`, `LIVEKIT_API_KEY` and `LIVEKIT_API_SECRET` to your cloud project's values.

Test from a phone on cellular data.

## 6. Start

```bash
cd /mnt/cache/huly
docker compose config        # look for blank variables or warnings
docker compose up -d
docker compose logs -f account transactor
```

In the Unraid Compose Manager plugin, point the stack at the `/mnt/cache/huly` folder so it finds both `docker-compose.yml` and `.env`.

First start takes about a minute; Elasticsearch downloads its `ingest-attachment` plugin and needs the `egress` network. Because no ports are published, test through your Cloudflare hostname.

## 7. Disable public sign-up (optional)

Create your own account first, then add this line under `environment:` of both `account` and `front` in `docker-compose.yml`, and run `docker compose up -d`:

```yaml
      - DISABLE_SIGNUP=true
```

## Differences from upstream compose.yml

- Image for `minio` is `pgsty/silo`; the service name stays `minio`.
- nginx config is embedded via `configs`; `_love`, `_pulse` and `_print` routes are enabled.
- `love`, `livekit`, `playit` and `cloudflared` are added; `hulypulse`, `mail` and `notification` are enabled.
- No host ports are published; `huly_net` is internal and Docker assigns its subnet.
- Unused `COCKROACH_SECRET`, `REDPANDA_SECRET`, `HTTP_BIND` and `HTTP_PORT` are removed.
- Routes for calendar, gmail, telegram, github, export and aibot are not proxied.

## Upgrading

Read `MIGRATION.md` in the repo root and take a backup first. Change `HULY_VERSION` and `DESKTOP_CHANNEL` in `.env`, then:

```bash
cd /mnt/cache/huly
docker compose pull
docker compose up -d
```

## Troubleshooting

- **Sign-up fails:** `docker compose logs -f account`.
- **502 from nginx:** services are still starting, or the resolver `127.0.0.11` is wrong. That is Docker's embedded DNS; Podman needs a different one.
- **File uploads fail:** `docker compose logs minio`.
- **Calls connect but no audio or video:** the playit local address, the matching ports, or `LIVEKIT_NODE_IP` is wrong.
- **playit cannot resolve `livekit`:** check `docker compose logs playit`. If the agent only accepts IP addresses, give LiveKit a fixed address: add `ipv4_address: <free IP>` under `livekit` and an `ipam` subnet under `huly_net` that does not overlap `docker network ls` output, then use that IP as the local address.
- **`Pool overlaps with other one`:** a leftover network from an earlier attempt. Run `docker compose down` and `docker network rm huly_net huly_egress huly_huly_net`, then start again.
- **`mail` or `notification` keeps restarting:** SMTP host or push keys are empty.
- **Blank variable warnings from `docker compose config`:** a required line in `.env` is empty or misspelled.
