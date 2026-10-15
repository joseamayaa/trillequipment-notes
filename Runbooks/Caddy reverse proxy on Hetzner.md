---
type: runbook
project: trillequipment
tags: [caddy, docker, hetzner, cloudflare]
updated: 2026-10-15
---

# Caddy reverse proxy on Hetzner

Caddy (in Docker) owns ports 80/443 on the Hetzner server, gets HTTPS certificates automatically and routes each hostname to a container. Use this to add or change a site. Hub: [[Trill Equipment]]

Layout and containers: [[Hetzner server layout]]. Why Caddy: [[2026-10-15 Caddy replaces nginx]]

## How it is set up
- Folder: `/opt/x2g-infrastructure/caddy/` with `Caddyfile`, `docker-compose.yaml` and `logs/`.
- Caddy joins two Docker networks: `web` (shared, external) and `supabase_default` (Supabase stack, external).
- Routes (hostname → container): `n8n.xidolabs.cc` → `n8n:5678` (`flush_interval -1` for n8n's live push), `trill.xidolabs.cc` → `site:80`, `trill-api.xidolabs.cc` → `supabase-envoy:8000`, `builder.xidolabs.cc` → static files (`/srv/builder`, and `/srv/projects` with a directory listing under `/projects/`).
- Logs: `logs/caddy.log` (events, certificates), `logs/access.log` (one line per request). Startup errors such as `address already in use` only show in `docker compose logs caddy`.
- Cloudflare: wildcard `*.xidolabs.cc` A record, **proxied**, plus explicit `n8n` and `builder` records. SSL/TLS mode must be **Full (strict)**. Caddy gets its certificates through the proxy with the HTTP challenge on port 80.

## Add a new project (the five steps)
1. Make a folder `/opt/x2g-infrastructure/<project>/` with its `docker-compose.yaml`. **No published ports.** Put the container on the `web` network:
   ```yaml
   networks:
     web:
       external: true
   ```
   and add `networks: [web]` to the service.
2. Start it: `docker compose up -d`.
3. Add a block to `/opt/x2g-infrastructure/caddy/Caddyfile`:
   ```
   app.xidolabs.cc {
       import access_log
       reverse_proxy <container-name>:<port>
   }
   ```
4. Validate, then reload with no downtime:
   ```
   cd /opt/x2g-infrastructure/caddy
   docker run --rm -v "$PWD/Caddyfile:/etc/caddy/Caddyfile" -v "$PWD/logs:/var/log/caddy" caddy:2 caddy validate --config /etc/caddy/Caddyfile
   docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```
5. No DNS step: the wildcard already covers one-level subdomains. Check `https://app.xidolabs.cc`.

Changing a volume or port in the compose file recreates the container: `docker compose config --quiet && docker compose up -d` (a few seconds of downtime for every site).

## Cutover from nginx (done 2026-10-14/15)
```
cd /opt/x2g-infrastructure/caddy
docker compose pull && docker compose create --force-recreate
sudo systemctl stop nginx && docker compose start caddy
docker compose logs -f caddy      # wait for "certificate obtained successfully"
```
Rollback: `docker compose stop caddy && sudo systemctl start nginx` (nginx's config and certificates were left untouched).

## Check it worked
- `curl -I https://n8n.xidolabs.cc`, `https://trill.xidolabs.cc`, `https://builder.xidolabs.cc`
- `curl -i https://trill-api.xidolabs.cc/auth/v1/health -H "apikey: <ANON_KEY>"` returns 200
- n8n in the browser: run a workflow and watch execution progress update live.

## Troubleshooting
- `failed to bind host port 0.0.0.0:80 ... address already in use` → something else holds 80/443 (here it was host nginx; `sudo ss -tlnp | grep ":80 "` shows who).
- `networks must be a mapping` → bad indentation or a dash in the top-level `networks:`; run `docker compose config --quiet`.
- Cloudflare `526` or a certificate that is slow to arrive → check `caddy.log`; if issuance keeps failing, set that record to DNS-only (grey) briefly, let Caddy issue, then switch the proxy back on.
- `Caddyfile` must be a **file**; if it is missing when Compose starts, Docker creates an empty directory of that name.
- `trillequipment-site-1` and `n8n` were attached to `web` with `docker network connect`, which is **lost when either container is recreated**. Add `web` to their own compose files (see [[Hetzner server layout]]).
- Compose files must use spaces, not tabs; `cat -A docker-compose.yaml | head` shows tabs as `^I`.

Related: [[Self-host Supabase on Hetzner]], [[Update the Hetzner test server]], [[Back up n8n and builder]]
