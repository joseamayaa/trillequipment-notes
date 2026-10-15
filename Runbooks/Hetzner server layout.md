---
type: reference
project: trillequipment
tags: [hetzner, docker, caddy, layout]
updated: 2026-10-15
---

# Hetzner server layout

What lives where on the Hetzner server, as of 2026-10-15. The server is kept as a general multi-project host (Docker + Caddy). Hub: [[Trill Equipment]]

## Folders
```
/opt/x2g-infrastructure/
├── caddy/             Caddyfile, docker-compose.yaml, logs/   (active)
├── backups/           builder/, n8n/  (see [[Back up n8n and builder]])
├── migrations/        empty, for SQL files
├── supabase/          empty; the running stack is still in its original folder
└── trillequipment/   test project (contents not recorded)

Outside that tree:
/opt/n8n/                    n8n compose project (+ /opt/n8n/backups, n8n's own)
/opt/website-builder/projects   generated sites; n8n writes here; mode 777
/var/www/website-builder/form   builder pages (index.html, refine.html)
~/trillequipment                the test site's docker-compose.yaml (original location)
```
Deciding whether to move `/opt/n8n`, `/opt/website-builder`, `/var/www/website-builder` and the Supabase folder into `/opt/x2g-infrastructure/` is still open.

## Containers
- **caddy**: ports 80/443, networks `web` + `supabase_default`.
- **n8n** (`n8nio/n8n`, bound to `127.0.0.1:5678`) + **n8n-postgres**; volumes `n8n_n8n_data`, `n8n_postgres_data`; also mounts `/opt/website-builder` and `/opt/n8n/backups`.
- **trillequipment-site-1** (`jjohamaya/trillequipment:test-1`), no published port, network alias `site` on `web`.
- **Supabase stack** (`supabase-*`, gateway is **Envoy**: `supabase-envoy`/`api-gw`).

## Networks
- `web` (created 2026-10-14): caddy, n8n, trillequipment-site-1. `n8n` and the site were attached with `docker network connect`; make it permanent in their compose files.
- `supabase_default`: Supabase stack + caddy.

## Ports
| Port | What | Public? |
|---|---|---|
| 22 | SSH | yes |
| 80 / 443 | Caddy | yes |
| 8000 | Supabase gateway (Envoy), plain HTTP | **still open; close it** |
| 5432 / 6543 | `supabase-pooler` publishes on 0.0.0.0 | **verify the firewall blocks them** |
| 5678 | n8n | localhost only |

## DNS (Cloudflare, zone `xidolabs.cc`)
- Wildcard `*.xidolabs.cc` → server, **proxied**; explicit `n8n` and `builder` records.
- SSL/TLS: Full (strict).
- Hostnames in use: `n8n.`, `builder.`, `trill.`, `trill-api.`

## host nginx
Stopped on 2026-10-14/15 (not removed). Its configs are in `/etc/nginx/sites-available/`, and certbot certificates in `/etc/letsencrypt/`. It was the first thing on 80/443 and served `builder` and `n8n`.
**Check `systemctl is-enabled nginx`.** If enabled, a reboot starts nginx first and Caddy then cannot bind 80/443.

Related: [[Caddy reverse proxy on Hetzner]], [[Self-host Supabase on Hetzner]], [[2026-10-15 Caddy cutover and server platform]]
