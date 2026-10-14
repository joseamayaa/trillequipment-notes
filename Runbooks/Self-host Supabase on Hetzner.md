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

## Notes
- Self-hosted Studio has one project (the default). There is no GitHub integration; schema gets in via SQL migrations ([[Apply the schema to self-hosted Supabase]]).
- Plain HTTP on 8000 is for testing only; keys and the Studio password travel unencrypted. HTTPS comes with Caddy (next session).
- Keep `.env` out of git and back up its values (a lost `JWT_SECRET` means regenerating all keys and updating the app).

Related: [[Update the Hetzner test server]], [[2026-10-14 Self-hosting Supabase on Hetzner]]
