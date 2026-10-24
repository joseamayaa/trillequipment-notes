---
type: reference
project: trillequipment
tags: [hetzner, docker, caddy, layout]
updated: 2026-10-23
---

# Hetzner server layout

What lives where on the Hetzner server, as of 2026-10-23. The server is kept as a general multi-project host (Docker + Caddy + a shared Supabase). Hub: [[Trill Equipment]]

## Folders
```
/opt/x2g-infrastructure/
├── bin/               new-project.sh, remove-project.sh, new-schema.sh, apply-migrations.sh, remove-schema.sh   (see [[Project template scripts]])
├── caddy/             Caddyfile, docker-compose.yaml, logs/, sites/ (one <name>.caddy per project)   (active)
├── n8n/               docker-compose.yml (name: n8n), .env (hidden), backups/   (moved here 2026-10-18)
├── backups/           builder/, n8n/, supabase/ (manual dumps, env.last, schema dumps); daily/ + backup.sh (see [[Daily server backups]])
├── migrations/        project SQL files (merged from ~/migrations)
├── supabase/docker/   the running Supabase stack: docker-compose.yml, .env, volumes/ (moved here 2026-10-15)
├── trillequipment/    test site docker-compose.yaml (site on the `web` network only)
├── <project>/         one folder per project made by new-project.sh (compose with service `app`)
└── .removed/          folders of projects removed by remove-project.sh (delete by hand when sure)

Outside that tree:
/opt/website-builder/projects   generated sites; n8n writes here; mode 777
/var/www/website-builder/form   builder pages (index.html, refine.html)
```
Moving `/opt/website-builder` and `/var/www/website-builder` into `/opt/x2g-infrastructure/` is still open. `.env` files are hidden: use `ls -la`.

## Cleanup
Done 2026-10-23: the old copies `~/migrations.old-delete-me`, `/root/supabase` (the 3.3 GB upstream clone) and `/opt/n8n` were deleted after checks (disk 25 GB → 22 GB used). Still lying around (delete by hand when sure):
- [ ] `/opt/x2g-infrastructure/.removed/demo-*` (test projects)
- [ ] `/opt/x2g-infrastructure/caddy/Caddyfile.bak-2026-10-23` and `docker-compose.yaml.bak-2026-10-23`
- [ ] In `/opt/x2g-infrastructure/supabase/docker`: `.env.bak-2026-10-17` (holds every secret) and `docker-compose.yml.bak-2026-10-17`
- [ ] `/opt/x2g-infrastructure/backups/supabase/env.last` (rolling copy of the Supabase `.env` kept by the schema scripts, mode 600)
- Remove with `rm -rf` only after reading the path; `df -h /` shows the space reclaimed.

Moving a compose folder keeps the project name only if the folder name (or the compose `name:`) stays the same. The Supabase and n8n compose files set `name:` (`supabase`, `n8n`), so their volumes and networks kept their names and Caddy kept working. Without a pinned name the volumes would be recreated empty.

## Containers
- **caddy**: ports 80/443, networks `web` + `supabase_default` (both declared external in its compose file); mounts `./Caddyfile`, `./sites` (read-only), `./logs`, the builder folders, and the volumes `caddy_caddy_data` / `caddy_caddy_config`.
- **n8n** (`n8nio/n8n`, bound to `127.0.0.1:5678`) + **n8n-postgres**, from `/opt/x2g-infrastructure/n8n`; volumes `n8n_n8n_data`, `n8n_postgres_data`; networks `n8n_n8n-network` + `web`; mounts `./backups` and `/opt/website-builder`. n8n is configured with `DB_TYPE=sqlite` and its database is about 496 MB (original data, confirmed 2026-10-18), so `n8n-postgres` looks unused (to confirm).
- **trillequipment-site-1** (`jjohamaya/trillequipment:test-1`), no published port, network alias `site` on `web` (declared in its compose file).
- **Projects made by `new-project.sh`**: `<name>-app-1`, alias `<name>` on `web`, no published ports.
- **Supabase stack** (`supabase-*`, gateway is **Envoy**: `supabase-envoy`/`api-gw`), run from `/opt/x2g-infrastructure/supabase/docker`. Studio is at `https://trill-api.xidolabs.cc/` behind Envoy's Basic-auth login (`DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` in that `.env`). Never proxy straight to `supabase-studio:3000`: Studio has no login of its own. Calls to `/auth/v1/*` need the `apikey` header (401 without it). The API exposes the schemas in `PGRST_DB_SCHEMAS` (default `public,graphql_public`, plus one per project), read by `supabase-studio` and `supabase-rest`.
- Idle but kept: `supabase-edge-functions`, `supabase-imgproxy`, `supabase-storage`, `realtime-dev.supabase-realtime`, `supabase-pooler`. Optional; can be stopped with `docker compose stop <service>` to save memory.

## Networks
- `web`: caddy, n8n, trillequipment-site-1, every project container. Declared `external: true` in each compose file, so it survives a recreate.
- `supabase_default`: Supabase stack + caddy.
- `n8n_n8n-network`: n8n and n8n-postgres only.

## Ports
| Port | What | Public? |
|---|---|---|
| 22 | SSH | yes |
| 80 / 443 | Caddy | yes |
| 8000 | Supabase gateway (Envoy); published as `127.0.0.1:8000` since 2026-10-18 (line 91 of its `docker-compose.yml`) | no; firewall rules deleted |
| 5432 / 6543 | `supabase-pooler`, bound to `127.0.0.1` | no. DB access from the PC goes through an SSH tunnel |
| 5678 | n8n, bound to `127.0.0.1` | no; firewall rules deleted |

Docker publishes ports through its own iptables rules, which **bypass UFW**: `ufw status` does not show what is exposed. Check what is bound (`sudo ss -tlnp`) and test from outside. Only the Hetzner Cloud Firewall filters before Docker. From the PC on 2026-10-18: 22, 80, 443 answered; 5432, 6543, 5678, 8000, 3000 did not. Test:
```powershell
foreach ($p in 22,80,443,5432,6543,8000,5678,3000) {
  $r = Test-NetConnection <server-ip> -Port $p -WarningAction SilentlyContinue
  "{0}: {1}" -f $p, $r.TcpTestSucceeded
}
```
Firewall rules now: SSH 22, HTTP 80, HTTPS 443 in both the Hetzner Cloud Firewall and UFW (default deny incoming). The 5678 and 8000 rules were deleted from both (confirmed 2026-10-18).

## DNS (Cloudflare, zone `xidolabs.cc`)
- Wildcard `*.xidolabs.cc` → server, **proxied**; explicit `n8n` and `builder` records.
- SSL/TLS: Full (strict). A new hostname can return `525` for a few seconds until Caddy has its certificate.
- Hostnames in use: `n8n.`, `builder.`, `trill.`, `trill-api.`, plus one per project from `new-project.sh`.

## host nginx
Stopped on 2026-10-14/15 and **disabled** (`systemctl is-enabled nginx` → `disabled`, confirmed 2026-10-18), so a reboot cannot grab 80/443 before Caddy. Not removed: its configs are in `/etc/nginx/sites-available/`, and certbot certificates in `/etc/letsencrypt/`. Rollback: `docker compose stop caddy && sudo systemctl start nginx`. Disabling certbot's renewal timer is still open.

Related: [[Caddy reverse proxy on Hetzner]], [[Self-host Supabase on Hetzner]], [[Daily server backups]], [[Add a project to the server]], [[2026-10-15 Caddy cutover and server platform]], [[2026-10-18 n8n move, firewall and Supabase over HTTPS]], [[2026-10-23 Project template scripts and cleanup]]
