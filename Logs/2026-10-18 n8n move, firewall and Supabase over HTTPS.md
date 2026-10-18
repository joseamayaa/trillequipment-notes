---
type: log
project: trillequipment
tags: [hetzner, docker, n8n, supabase, firewall, backups]
date: 2026-10-18
---

# 2026-10-18 n8n move, firewall and Supabase over HTTPS

Hub: [[Trill Equipment]] · Worked with Claude Code · Continues [[2026-10-15 Caddy cutover and server platform]] · Session started late 2026-10-17

## Done
- **Made the server reboot-safe.**
  - host nginx is `disabled` and `inactive`, so a reboot cannot grab 80/443 before Caddy.
  - Caddy's own compose file already declares `web` and `supabase_default` as external networks, so both survive a recreate (nothing to change).
  - The `trillequipment` compose file still had an old `caddy` service (published 80/443). A `trillequipment-caddy-1` container sat in `Created` (never started) plus empty `trillequipment_caddy_*` volumes. Replaced the file with just `site` on `web` (alias `site`) and `docker compose up -d --remove-orphans`; then removed the two unused volumes. The real Caddy data is in `caddy_caddy_data`. *Not verified by output:* the `ps` / `web` network / `curl` check after the recreate was not pasted; the volume removal succeeding implies the orphan container was gone.
- **Moved n8n into `/opt/x2g-infrastructure/n8n`** (it lived in `/opt/n8n`).
  - Took a fresh tarball of `n8n_n8n_data` first: `backups/n8n/n8n_data-pre-move-2026-10-17.tgz` (63 MB).
  - `docker compose down` (no `-v`), `cp -a` the folder, new compose file has `name: n8n` (so the volume names cannot change), `./backups:/backups` instead of the absolute path, and `web` added to the `n8n` service plus declared external.
  - Verified: compose labels point at `/opt/x2g-infrastructure/n8n` (both containers), the mount is still `n8n_n8n_data`, no new volumes, `n8n` is on `n8n_n8n-network` + `web`, `https://n8n.xidolabs.cc` returns 200, nothing mounts from `/opt/n8n` any more.
  - Hidden files: a plain `ls` hid `.env`; `ls -la` shows it. `cp -a` and `diff -r` handle dotfiles; shell globs (`*`) do not.
- **Firewall check** (details in [[Hetzner server layout]]): pooler 5432/6543 and n8n 5678 are bound to `127.0.0.1`. Docker publishes bypass UFW, so what counts is what is bound and a test from outside. From the PC only 22, 80, 443 answer. Removed the stale `5678/tcp` UFW rules (IPv4 + IPv6).
- **Supabase URLs now HTTPS.** In the stack's `.env`: `SUPABASE_PUBLIC_URL=https://trill-api.xidolabs.cc`, `API_EXTERNAL_URL=https://trill-api.xidolabs.cc/auth/v1`, `SITE_URL=https://trill.xidolabs.cc` (two of the three had a mistyped IP before). `docker compose up -d` recreated only auth, storage and edge-functions. `/auth/v1/health` returns 200 with the anon key on both `https://trill-api...` and the old `:8000`; without the key it is 401 (the gateway wants `apikey`). → [[Self-host Supabase on Hetzner]]
- **Closed port 8000 on the host.** Line 91 of the stack's `docker-compose.yml` is now `127.0.0.1:${API_GW_HTTP_PORT:-${KONG_HTTP_PORT:-8000}}:8000/tcp`; only the gateway container was recreated; `ss` shows `127.0.0.1:8000`; port test from the PC: 8000 `False`. Caddy reaches the gateway over `supabase_default`, so `trill-api` is unaffected.
- **Reviewed the Trill project (read-only, nothing changed or committed).** The two migrations (`20261007000000_initial_schema.sql`, `20261007000100_catalog_data.sql`) look sound, with RLS on all 6 tables, but **were not applied**. Findings in Notes.
- **Decided to scope the work to the platform** (not Trill itself) → [[2026-10-18 Shared Supabase and pulled backups]]
- **Wrote the daily backup script** (not run yet) → [[Daily server backups]]

## Commits / PRs
- Notes only; nothing committed in this vault or in the project.

## Next
Platform setup only; Trill-specific work is parked (see Notes).
- [ ] **Run the backup script**: put `backup.sh` in place, run it by hand, check the 7 files, add the cron entry, pull one backup to the PC, then do a restore test into a temporary Postgres → [[Daily server backups]]
- [ ] Caddy `import /etc/caddy/sites/*.caddy` + `new-project.sh` / `remove-project.sh` + templates. One snippet file per project, `caddy validate` before reload, compose on `web`, no published ports. Supabase path: a schema per project (see [[2026-10-18 Shared Supabase and pulled backups]])
- [ ] Studio hardening: own hostname behind Cloudflare Access; API hostname answering only `/rest`, `/auth`, `/storage`, `/realtime`, `/functions`
- [ ] **Confirm what is still unchecked:**
  - log in to `https://n8n.xidolabs.cc` and open your workflows. The data itself is confirmed: `docker exec n8n ls -la /home/node/.n8n` shows the original `database.sqlite` (496 MB, last written Apr 26) with a live `-wal` file, and config/event-log files dating back to Feb 2026, so n8n did not start empty. Only the browser look is left.
  - the `trillequipment` recreate: `docker ps -a | grep -i -E 'caddy|trill'` shows only `caddy` and `trillequipment-site-1`, the site is on `web`, `curl -sI https://trill.xidolabs.cc` returns 200 (output never seen)
- [x] Confirmed by the user at the end of the session: Studio login works over HTTPS after the URL change; the Hetzner firewall rules for 5678 and 8000 and UFW's 8000 rule are deleted
- [ ] `mv /opt/n8n /opt/n8n.old-delete-me`, then delete it with the other leftovers **on or after 2026-10-22** (`~/migrations.old-delete-me`, `~/supabase.old-delete-me`)
- [ ] Put the Supabase `.env` secrets in a password manager; once not needed, delete `docker-compose.yml.bak-2026-10-17` and `.env.bak-2026-10-17` in `/opt/x2g-infrastructure/supabase/docker` (the `.env` copy holds every secret)
- [ ] Look inside `/opt/website-builder/projects` (mode 777, public listing) for anything that should not be public

## Notes
- **Parked (Trill-specific, not needed for the platform):** apply the schema, add the missing equipment types, rewire the form, build `test-2`. If it is revived, know this: the site has 14 equipment pages and 14 options in the reserve dropdown, but the catalog migration seeds only 8 types (missing `backhoe-loader`, `mini-excavator`, `telehandler`, `compact-roller`, `light-tower`, `generator`), so `submit_reservation_request` would raise `Unknown equipment` for those. The form currently posts to a Google Apps Script mailer (`assets/site.js`), has no `config.js` or `supabase-js`, sends display names where the function wants slugs, treats `company` and the dates as optional while the database does not. Add the six types in a **new** migration instead of editing the applied one. Local hashes (sha256): `851f3b31...fa3ea` schema, `b6628fc7...cff4ed` catalog data.
- The project's `Dockerfile` and `.dockerignore` are already committed (working tree clean on `Joh's-Branch`); there is no `docker-compose.yml` in the project, only on the server.
- n8n runs on SQLite (`DB_TYPE=sqlite`); the `n8n-postgres` container looks unused (its dump was 20 KB). Not confirmed; if true, back up the `n8n_n8n_data` volume, not the Postgres dump, and the unused service can be dropped later.
- A `200` from a URL only proves the web server answered; for n8n the real check is that the workflows are there.
- The decisions folder on disk is `Decisions Decisions Decisions`, not `Decisions` (git status shows the rename uncommitted); links by note name still resolve.
