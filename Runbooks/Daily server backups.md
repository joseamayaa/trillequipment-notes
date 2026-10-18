---
type: runbook
project: trillequipment
tags: [backup, hetzner, docker, supabase, n8n, caddy]
updated: 2026-10-18
---

# Daily server backups

A script on the Hetzner server backs up Supabase, n8n and Caddy every night and keeps 7 days; I pull a copy to my PC. Hub: [[Trill Equipment]]

**Status: written 2026-10-18, not run yet.** Run it by hand first (steps below). The backup files contain secrets (`.env` files, the database dump): never in git, this vault or chat. Decision: [[2026-10-18 Shared Supabase and pulled backups]]

## What it saves
Into `/opt/x2g-infrastructure/backups/daily/<date>/`:
- `supabase-db.sql.gz`: `pg_dumpall` of `supabase-db` (the safe copy; do not copy the live `volumes/db`)
- `supabase-files.tgz`: the stack's `.env`, `docker-compose.yml`, and `volumes/storage` + `volumes/functions` if present
- `n8n-data.tgz` (volume `n8n_n8n_data`: workflows and credentials, SQLite) and `n8n-config.tgz` (`.env`, compose)
- `caddy-data.tgz` (volume `caddy_caddy_data`: account key and certificates) and `caddy-config.tgz` (`Caddyfile`, compose)
- `projects.tgz`: `migrations/` and `trillequipment/`

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

   log "n8n volume and config"
   docker run --rm -v n8n_n8n_data:/data:ro -v "$OUT":/out alpine \
     tar czf /out/n8n-data.tgz -C /data .
   tar czf "$OUT/n8n-config.tgz" -C "$BASE/n8n" .env docker-compose.yml

   log "caddy volume and config"
   docker run --rm -v caddy_caddy_data:/data:ro -v "$OUT":/out alpine \
     tar czf /out/caddy-data.tgz -C /data .
   tar czf "$OUT/caddy-config.tgz" -C "$BASE/caddy" Caddyfile docker-compose.yaml

   log "projects and migrations"
   tar czf "$OUT/projects.tgz" -C "$BASE" migrations trillequipment

   log "prune older than $KEEP_DAYS days"
   find "$BASE/backups/daily" -mindepth 1 -maxdepth 1 -type d -mtime +"$KEEP_DAYS" -exec rm -rf {} +

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
- The run ends with a `done:` line and the folder holds the 7 files above.
- The next morning `tail /var/log/x2g-backup.log` shows a run for that date.
- A backup is not proven until a restore works: load `supabase-db.sql.gz` into a **temporary** Postgres container (never the live one) and list the tables. *Restore test: still to do.*

## Troubleshooting
- `tar: ...: No such file or directory` in the run → a file name differs from the list above; fix the name in the script.
- Run stops early with no `done:` → `set -e` stops at the first error; the line before it says which step failed.
- The n8n volume holds a live SQLite database (about 496 MB, in WAL mode: recent writes sit in `database.sqlite-wal`) that is tarred while n8n runs, so the main file and the WAL are copied at slightly different moments. Usually restores fine, but is not guaranteed; the restore test is the real proof. A safer option is a consistent copy with `sqlite3 ... ".backup"` inside the container (or stopping n8n briefly) before the tar.
- Disk filling up → lower `KEEP_DAYS`, or look at which tarball grew (`du -sh daily/*/*`).

Related: [[Back up n8n and builder]], [[Hetzner server layout]], [[Self-host Supabase on Hetzner]]
