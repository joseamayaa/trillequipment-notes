---
type: log
project: trillequipment
tags: [hetzner, caddy, supabase, backups, scripts, cleanup]
date: 2026-10-23
---

# 2026-10-23 Project template scripts and cleanup

Hub: [[Trill Equipment]] · Worked with Claude Code · Continues [[2026-10-18 n8n move, firewall and Supabase over HTTPS]]

## Done
- **Confirmed the open checks:** n8n workflows and the Trill site recreate were checked and reported good at the start of the session.
- **Backup script ran by hand** (2026-10-23 18:38 server time): 8 files, 59 MB, about 12 seconds. The n8n part was changed first to take a consistent copy with `sqlite3 .backup` (the database is 496 MB in WAL mode) → [[Daily server backups]]. The files made inside Docker containers came out world-readable (the `daily/` folder is mode 700), so a `chmod 600 "$OUT"/*` line was added to the script. The Supabase dump is only 53 KB (no project data in it yet).
- **Copy to the PC did not work:** `scp` to the PC kept asking for a password; the key setup was started but not finished (see Next). Parked.
- **Project template built and tested** → [[Add a project to the server]], [[Project template scripts]]
  - Caddy now imports one file per project from `caddy/sites/` (`import /etc/caddy/sites/*.caddy`, mount `./sites:/etc/caddy/sites:ro`).
  - `new-project.sh` / `remove-project.sh`: tested with `nginx:alpine` (`demo.xidolabs.cc`): created, returned 200, removed. The first attempt returned Cloudflare 525 and `caddy validate` still passed: Caddy had never been recreated, so it had no `sites` mount ("No files matching import glob pattern" in the output) and the route did not exist. After `docker compose up -d` in the Caddy folder it worked. The script now checks `docker exec caddy test -f /etc/caddy/sites/<name>.caddy` before validating.
  - `new-schema.sh` / `apply-migrations.sh` / `remove-schema.sh`: tested with a throwaway schema `demo_app`: reading with the schema header returned the row, the same table without it gave `PGRST205` (not in `public`), an anon insert without a policy gave `42501` (RLS), and the migration runner printed `apply`, then `skip`. `apply-migrations.sh` failed once with a syntax error at the loop's `done` (a `\` line continuation, probably damaged when pasted) and was rewritten; nothing was applied by the broken run. `remove-schema.sh` was run by the user (schema gone, `PGRST_DB_SCHEMAS` back to `public,graphql_public`); a second run printed the `does not exist` guard.
- **Cleanup (was due 2026-10-22), disk 25 GB → 22 GB used:** deleted `/root/migrations.old-delete-me` (`diff -rq` against `/opt/x2g-infrastructure/migrations` printed nothing), `/opt/n8n` (the only difference from the new folder was the compose edit made on purpose) and `/root/supabase` (the 3.3 GB upstream clone; it had never been renamed to `.old-delete-me`). Nothing running mounted any of them. The outputs of the final git and compose-label checks on `/root/supabase` were not pasted before it was deleted.
- Health check after all of it: `n8n`, `trill`, `builder` return 200 and `trill-api` returns 401 (Studio login).

## Commits / PRs
- Notes only. The scripts live on the server (full source in [[Project template scripts]]).

## Next
- [ ] **Add `bin/`, `caddy/sites/` and the project folders to the nightly backup** (`projects.tgz` covers only `migrations` and `trillequipment`; `caddy-config.tgz` only the Caddyfile and compose file)
- [ ] Confirm the cron entry exists: `cat /etc/cron.d/x2g-backup` (the line was given; not confirmed) and that `chmod 600` line is in `backup.sh` (`grep chmod /opt/x2g-infrastructure/backups/backup.sh`)
- [ ] **Get a backup copy off the server** (PC): `scp` still asks for a password. Fix the key (generate or find the key on the PC, append its **public** line to `/root/.ssh/authorized_keys` from the existing server shell, then pull) or use the server login; then do the **restore tests** (Supabase dump into a throwaway container of the same image, n8n database `pragma integrity_check`)
- [ ] Optional `--auth` (Basic auth) for `new-project.sh`, so test sites are not public by default
- [ ] Studio hardening: own hostname behind Cloudflare Access; API hostname answering only `/rest`, `/auth`, `/storage`, `/realtime`, `/functions`
- [ ] Small cleanups: `/opt/x2g-infrastructure/.removed/demo-*` folders, `Caddyfile.bak-2026-10-23` and `docker-compose.yaml.bak-2026-10-23` in `caddy/`, and in `supabase/docker/` the `.env.bak-2026-10-17` (holds every secret; delete once the values are in a password manager) and `docker-compose.yml.bak-2026-10-17`
- [ ] Note: `new-schema.sh` / `remove-schema.sh` keep one rolling copy of the Supabase `.env` at `backups/supabase/env.last` (mode 600): it holds every secret
- [ ] Look inside `/opt/website-builder/projects` (mode 777, public listing) for anything that should not be public
- [ ] Decide whether to move `/opt/website-builder` and `/var/www/website-builder` into `/opt/x2g-infrastructure/`; drop the unused `n8n-postgres` service if confirmed unused
- [ ] Make the new-project script retry the first request until the certificate arrives (it prints `525` right after `ok`, which looks like a failure)

## Notes
- A `200` from `caddy validate` is not proof a route exists: it validated while Caddy could not see the `sites` folder. The mount check now covers that case.
- Paste damage: a long heredoc pasted into a terminal can break a `\` line continuation. Run `bash -n <script>` after creating a script and look for `OK`.
- Docker publishes ports through its own iptables rules, which bypass UFW ([[Hetzner server layout]]).
- The migrations in the Trill repo write `public.` names explicitly, so `apply-migrations.sh` would still put them in `public`; they would need the prefix removed to use a project schema (Trill is parked).
