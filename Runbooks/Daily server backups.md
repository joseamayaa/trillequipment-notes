---
type: runbook
project: trillequipment
tags: [backup, hetzner, docker, supabase, n8n, caddy]
updated: 2026-10-18
---

# Daily server backups

A script on the Hetzner server backs up Supabase, n8n and Caddy every night and keeps 7 days; I pull a copy to my PC. Hub: [[Trill Equipment]]

**Status (2026-10-23):** written 2026-10-18 and run by hand on 2026-10-23 (8 files, 59 MB, about 12 seconds, no errors). The cron entry (`/etc/cron.d/x2g-backup`, 03:15) and the `chmod 600` line were confirmed in place later that day, and the script gained `infra-extra.tgz` (9 files, still 59 MB; first scheduled run is the night of 2026-10-23, check `tail /var/log/x2g-backup.log`). The copy to the PC and the restore tests are **not done** (`scp` to the PC kept asking for a password; the SSH key is not set up). The n8n step was rewritten to a consistent `sqlite3 .backup` copy before that run. To make the files private, add `chmod 600 "$OUT"/*` just before the final `log "done` line (the files created inside Docker containers are world-readable otherwise; `daily/` is mode 700). The backup files contain secrets (`.env` files, the database dump): never in git, this vault or chat. Decision: [[2026-10-18 Shared Supabase and pulled backups]]

## What it saves
Into `/opt/x2g-infrastructure/backups/daily/<date>/`:
- `supabase-db.sql.gz`: `pg_dumpall` of `supabase-db` (the safe copy; do not copy the live `volumes/db`)
- `supabase-files.tgz`: the stack's `.env`, `docker-compose.yml`, and `volumes/storage` + `volumes/functions` if present
- `n8n-database.sqlite.gz` (consistent `sqlite3 .backup` copy of the n8n database: workflows and credentials), `n8n-files.tgz` (the rest of `n8n_n8n_data`: config with the encryption key, event logs, nodes, `binaryData`) and `n8n-config.tgz` (`.env`, compose)
- `caddy-data.tgz` (volume `caddy_caddy_data`: account key and certificates) and `caddy-config.tgz` (`Caddyfile`, compose)
- `projects.tgz`: `migrations/` and `trillequipment/`
- `infra-extra.tgz`: everything in `/opt/x2g-infrastructure` except `backups/`, `supabase/`, `.removed/`, `caddy/logs` and `n8n/backups`. That covers `bin/` (the scripts), `caddy/` (Caddyfile, compose, `sites/`), `n8n/` config, `migrations/` and every project folder made by `new-project.sh`, so new projects are included automatically. Docker volumes of projects are not covered (only n8n and Caddy have their own volume backups).

Older manual backups in `/opt/x2g-infrastructure/backups/{builder,n8n,supabase}` are not touched or pruned.

## Before you start
- Folder names the script assumes: `/opt/x2g-infrastructure/{supabase/docker,n8n,caddy,migrations,trillequipment}`; Caddy's compose file is `docker-compose.yaml`, n8n's is `docker-compose.yml`. Check with `ls`.
- Containers `supabase-db` running; volumes `n8n_n8n_data` and `caddy_caddy_data` exist (`docker volume ls`).
- Free disk space: `df -h /`.

## Steps
1. Create the script (the quoted `'EOF'` stops the shell expanding anything inside it):
   ```
   cat > /opt/x2g-infrastructure/backups/backup.sh <<'EOF'
   #!/usr/bin/env bash
   # Daily backup of the x2g test server. Keeps KEEP_DAYS days. Secrets inside: keep private.
   set -euo pipefail
   umask 077

   BASE=/opt/x2g-infrastructure
   OUT="$BASE/backups/daily/$(date +%F)"
   KEEP_DAYS=7
   mkdir -p "$OUT"

   log() { echo "$(date '+%F %T') $*"; }

   log "supabase db dump"
   docker exec supabase-db pg_dumpall -U postgres | gzip > "$OUT/supabase-db.sql.gz.tmp"
   gzip -t "$OUT/supabase-db.sql.gz.tmp"
   mv "$OUT/supabase-db.sql.gz.tmp" "$OUT/supabase-db.sql.gz"

   log "supabase config and files"
   tar czf "$OUT/supabase-files.tgz" -C "$BASE/supabase/docker" \
     .env docker-compose.yml \
     $(for d in volumes/storage volumes/functions; do [ -e "$BASE/supabase/docker/$d" ] && echo "$d"; done)

   log "n8n database (consistent SQLite copy while n8n runs)"
   docker run --rm -v n8n_n8n_data:/data -v "$OUT":/out alpine sh -c '
     apk add --no-cache sqlite >/dev/null &&
     sqlite3 /data/database.sqlite ".backup /out/n8n-database.sqlite" &&
     gzip -f /out/n8n-database.sqlite'

   log "n8n other data and config"
   docker run --rm -v n8n_n8n_data:/data:ro -v "$OUT":/out alpine sh -c '
     cd /data && tar czf /out/n8n-files.tgz $(ls -A | grep -v "^database\.sqlite")'
   tar czf "$OUT/n8n-config.tgz" -C "$BASE/n8n" .env docker-compose.yml

   log "caddy volume and config"
   docker run --rm -v caddy_caddy_data:/data:ro -v "$OUT":/out alpine \
     tar czf /out/caddy-data.tgz -C /data .
   tar czf "$OUT/caddy-config.tgz" -C "$BASE/caddy" Caddyfile docker-compose.yaml

   log "projects and migrations"
   tar czf "$OUT/projects.tgz" -C "$BASE" migrations trillequipment

   log "bin, caddy sites, project folders"
   tar czf "$OUT/infra-extra.tgz" -C "$BASE" --exclude=./backups --exclude=./supabase --exclude=./.removed --exclude=./caddy/logs --exclude=./n8n/backups .

   log "prune older than $KEEP_DAYS days"
   find "$BASE/backups/daily" -mindepth 1 -maxdepth 1 -type d -mtime +"$KEEP_DAYS" -exec rm -rf {} +

   chmod 600 "$OUT"/*
   log "done: $(du -sh "$OUT" | cut -f1) in $OUT"
   EOF
   ```
   The database is dumped to a `.tmp` file and checked with `gzip -t` before it is renamed; the volumes are tarred through a throwaway `alpine` container with the volume mounted read-only; pruning only touches day folders inside `daily/`.
2. Make it executable and the folders private:
   ```
   chmod 700 /opt/x2g-infrastructure/backups/backup.sh
   mkdir -p /opt/x2g-infrastructure/backups/daily && chmod 700 /opt/x2g-infrastructure/backups/daily
   ```
3. Run it by hand and look:
   ```
   sudo /opt/x2g-infrastructure/backups/backup.sh
   ls -lh /opt/x2g-infrastructure/backups/daily/*/
   ```
4. Schedule it (03:15 server time):
   ```
   echo '15 3 * * * root /opt/x2g-infrastructure/backups/backup.sh >> /var/log/x2g-backup.log 2>&1' | sudo tee /etc/cron.d/x2g-backup
   sudo chmod 644 /etc/cron.d/x2g-backup
   ```
5. Pull a copy to the PC (Git Bash or WSL, not PowerShell) into a folder **outside any git repo**:
   ```
   mkdir -p ~/x2g-backups
   scp -r <user>@<server-ip>:/opt/x2g-infrastructure/backups/daily/$(date +%F) ~/x2g-backups/
   ```

## Check it worked
- The run ends with a `done:` line and the folder holds the 9 files above (`supabase-db.sql.gz`, `supabase-files.tgz`, `n8n-database.sqlite.gz`, `n8n-files.tgz`, `n8n-config.tgz`, `caddy-data.tgz`, `caddy-config.tgz`, `projects.tgz`, `infra-extra.tgz`), all mode 600. `tar tzf .../infra-extra.tgz | grep -E 'bin/|sites/'` lists the scripts and `caddy/sites/`.
- The next morning `tail /var/log/x2g-backup.log` shows a run for that date.
- A backup is not proven until a restore works: load `supabase-db.sql.gz` into a **temporary** Postgres container (never the live one) and list the tables. *Restore tests: done 2026-10-26, see below.*

## Troubleshooting
- `tar: ...: No such file or directory` in the run → a file name differs from the list above; fix the name in the script.
- Run stops early with no `done:` → `set -e` stops at the first error; the line before it says which step failed.
- The n8n database is about 496 MB in WAL mode (recent writes sit in `database.sqlite-wal`), so it is copied with `sqlite3 .backup`, not `tar`, which gives a consistent file while n8n runs. That step installs `sqlite` inside a throwaway `alpine` container (needs outbound internet) and mounts the volume read-write, because SQLite may touch the `-shm` file. If `apk add` fails, `set -e` stops the script before the later steps; run it again, or check the server's DNS/outbound access. Not tested before 2026-10-23; the first run and the restore test are the proof.
- n8n restore: stop n8n, put `n8n-database.sqlite.gz` (gunzipped) back as `database.sqlite` in the volume and unpack `n8n-files.tgz` beside it (the `config` file holds the encryption key; without it saved credentials are unreadable), then start n8n.
- Disk filling up → lower `KEEP_DAYS`, or look at which tarball grew (`du -sh daily/*/*`).

## Restore tests (passed 2026-10-26)
Run on the server against `daily/2026-10-26`; the copy on the PC pulled fine over `scp` (key already installed; 9 files, mode 600).
- **Supabase:** start a throwaway container of the live image (`docker inspect supabase-db --format '{{.Config.Image}}'`), wait 20 seconds, then `zcat supabase-db.sql.gz | docker exec -i restore-test psql -U supabase_admin -d postgres`. Restore as **`supabase_admin`**, not `postgres`: the `auth`, `storage` and `realtime` schemas belong to `supabase_admin`, and as `postgres` about 1000 permission errors come up. With the right role only about 70 "already exists" errors remain. Compare counts with the live database (`auth.users`, `storage.buckets`, tables in `x2g_meta`): they matched (0 / 2 / 1). Then `docker rm -f restore-test`.
- **n8n:** `zcat n8n-database.sqlite.gz` into a file inside a throwaway `alpine` with `sqlite`, then `pragma integrity_check` printed `ok`; the copy holds 10 workflows and 4 credentials.
- The Supabase dump has no project data yet (53 KB), so the test proves the roles and structure, not large tables. Repeat it once a real project holds data.

## Gaps (as of 2026-10-26)
- Fixed 2026-10-23: `bin/`, `caddy/sites/` and the per-project folders are now in `infra-extra.tgz`. Still **not** backed up: the Docker volumes of projects (other than n8n and Caddy), the `.removed/` folder and the Caddy logs.
- Schema dumps from `remove-schema.sh` and `env.last` sit in `backups/supabase/` and are not in `daily/`.
- The off-server copy is pulled by hand (step 5), so it is only as fresh as the last pull. Pull again after big changes.

Related: [[Back up n8n and builder]], [[Hetzner server layout]], [[Self-host Supabase on Hetzner]], [[Add a project to the server]]
