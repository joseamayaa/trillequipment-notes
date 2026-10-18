---
type: runbook
project: trillequipment
tags: [supabase, hetzner, docker, self-hosting]
updated: 2026-10-14
---

# Self-host Supabase on Hetzner

Run the Supabase stack (Docker Compose) on the Hetzner test server so the site has a backend for testing. Hub: [[Trill Equipment]]

No secrets belong in this note. Real values live only in the stack's `.env` on the server (and a password manager).

## Before you start
- SSH access to the Hetzner server, Docker + Docker Compose installed.
- The `docker/` folder from the `supabase/supabase` repo cloned on the server (it holds `docker-compose.yml` and `.env.example`).
- The server's public IP (called `<server-ip>` below).

## Steps
1. Create the env file (the destination `.env` is created by `cp`; the template is `.env.example`, dot first):
   ```
   cd <path>/supabase/docker
   cp .env.example .env
   ```
2. Generate the secrets. Answer **yes** when it asks to update `.env`:
   ```
   sh ./utils/generate-keys.sh
   ```
   It sets `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY` and the other secrets. Never mix keys from two runs; if `JWT_SECRET` changes, regenerate both keys.
3. Edit `.env` (`nano .env`; save with Ctrl+O, Enter, Ctrl+X) and set the URLs. Only replace `localhost`; keep the template's path suffixes:
   ```
   SUPABASE_PUBLIC_URL=http://<server-ip>:8000
   API_EXTERNAL_URL=http://<server-ip>:8000/auth/v1
   SITE_URL=http://<frontend-url>
   ```
   `SITE_URL` is where auth redirects users: the **frontend**, not the API. Also check `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD` (Studio login) and `POSTGRES_PASSWORD` (no special characters like `@` or `/`). Run `grep -n localhost .env` for leftovers.
4. Firewall: allow inbound TCP **22** and **8000** only (Hetzner Cloud firewall, and `ufw` if used).
5. Start it:
   ```
   docker compose pull
   docker compose up -d
   docker compose ps
   ```
   Every service should be `healthy`. After changing `.env` later, use `docker compose down` then `up -d` (plain `restart` can keep old values; don't add `-v`, it deletes data).

## Check it worked
- On the server: `curl -i http://localhost:8000/auth/v1/health -H "apikey: <ANON_KEY>"` returns `200 OK`.
- From my PC: open `http://<server-ip>:8000`, log in to Studio with the dashboard credentials. (On Windows PowerShell use `curl.exe`, not `curl`.)

## Troubleshooting
- **`403 RBAC: access denied` on `/rest/v1/`** with the anon key: not a fault. This version's gateway is **Envoy** (`supabase-envoy`, service `api-gw`), not Kong, and the bare `/rest/v1/` root appears restricted (probably to the secret key). Test `/auth/v1/health` or a real table instead.
- **`401` (12 bytes)** → missing or invalid `apikey`.
- `POSTGRES_PASSWORD` only applies on first DB initialization; changing it later means wiping the DB volume.
- Postgres ports: the pooler (`supabase-pooler`) publishes **5432 and 6543 on 0.0.0.0**. Keep them closed in the firewall; use an SSH tunnel for DB access (`ssh -L 5432:localhost:5432 <user>@<server-ip>`, user `postgres.<POOLER_TENANT_ID>`).

## Move the stack to another folder
Done 2026-10-15 (now at `/opt/x2g-infrastructure/supabase/docker`).
```
SRC=$(docker inspect supabase-db --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}')
grep -n "^name:" "$SRC/docker-compose.yml"          # name: supabase keeps container/network names
docker exec supabase-db pg_dumpall -U postgres | gzip > /opt/x2g-infrastructure/backups/supabase/supabase-dump-$(date +%F).sql.gz
cp "$SRC/.env" /opt/x2g-infrastructure/backups/supabase/env.bak && chmod 600 /opt/x2g-infrastructure/backups/supabase/env.bak
cd "$SRC" && docker compose down          # never -v; "supabase_default has active endpoints" is Caddy and is fine
mv "$SRC" /opt/x2g-infrastructure/supabase/docker
cd /opt/x2g-infrastructure/supabase/docker && docker compose up -d && docker compose ps
```
The database lives in the bind-mounted `volumes/` folder, so move the whole folder (including the hidden `.env`). Rollback: `down`, move it back, `up -d`. `trill-api` returns 502 while it is down.

## Pooler ports
`supavisor` publishes 5432/6543. Bind them to localhost (`- 127.0.0.1:${POSTGRES_PORT}:5432` and `- 127.0.0.1:${POOLER_PROXY_PORT_TRANSACTION}:6543`), then `docker compose up -d supavisor`. Reach the database from the PC with `ssh -L 5432:localhost:5432 root@<server-ip>`.

## Switch to HTTPS URLs and close port 8000
Done 2026-10-18, once Caddy served `https://trill-api.xidolabs.cc`.
1. Back up, then change only the three URL lines (`sed` leaves the rest of the file alone). Keep the `/auth/v1` suffix on `API_EXTERNAL_URL`:
   ```
   cd /opt/x2g-infrastructure/supabase/docker
   cp -a .env .env.bak-<date> && chmod 600 .env .env.bak-<date>
   sed -i \
     -e 's|^SUPABASE_PUBLIC_URL=.*|SUPABASE_PUBLIC_URL=https://trill-api.xidolabs.cc|' \
     -e 's|^API_EXTERNAL_URL=.*|API_EXTERNAL_URL=https://trill-api.xidolabs.cc/auth/v1|' \
     -e 's|^SITE_URL=.*|SITE_URL=https://trill.xidolabs.cc|' .env
   grep -E '^(SUPABASE_PUBLIC_URL|API_EXTERNAL_URL|SITE_URL|ADDITIONAL_REDIRECT_URLS)=' .env
   docker compose config -q && docker compose up -d
   ```
   `up -d` recreates only the services whose values changed (auth, storage, edge-functions); no `-v`. Rollback: restore the `.bak` and `up -d`.
2. Check with the anon key read from the file (never paste it): `ANON=$(grep '^ANON_KEY=' .env | cut -d= -f2-)`, then `curl -s -o /dev/null -w '%{http_code}\n' -H "apikey: $ANON" https://trill-api.xidolabs.cc/auth/v1/health` returns `200`; `unset ANON`.
3. Bind the gateway port to localhost: in the gateway service, change the `ports:` line to `127.0.0.1:${API_GW_HTTP_PORT:-${KONG_HTTP_PORT:-8000}}:8000/tcp` (line 91 here; back up the compose file first), then `docker compose config -q && docker compose up -d`. Only the gateway is recreated, so `trill-api` is down for a few seconds. Check `sudo ss -tlnp | grep :8000` shows only `127.0.0.1`. Other `8000` mentions in the file (`http://api-gw:8000`, the healthcheck) are the container's own port and stay.
4. Delete the `8000/tcp` rules in UFW and the Hetzner Cloud Firewall. Publishing on localhost is more reliable than a firewall rule, because Docker bypasses UFW.

## Notes
- Self-hosted Studio has one project (the default). There is no GitHub integration; schema gets in via SQL migrations ([[Apply the schema to self-hosted Supabase]]).
- Plain HTTP on 8000 was for the first test only; Studio and the API are now served over HTTPS by Caddy and the port is bound to localhost (section above).
- Future projects share this stack with a schema each ([[2026-10-18 Shared Supabase and pulled backups]]); they share the anon and service-role keys.
- Keep `.env` out of git and back up its values (a lost `JWT_SECRET` means regenerating all keys and updating the app).

Related: [[Update the Hetzner test server]], [[2026-10-14 Self-hosting Supabase on Hetzner]]
