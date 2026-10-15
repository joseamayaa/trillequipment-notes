---
type: runbook
project: trillequipment
tags: [backup, n8n, docker, hetzner]
updated: 2026-10-15
---

# Back up n8n and builder

Take a copy of n8n (database, data folder, encryption key) and the builder site before risky server changes, and get it off the server. Hub: [[Trill Equipment]]

Stopping a container does **not** delete data (n8n's data is in the named volumes `n8n_n8n_data` and `n8n_postgres_data`). The risks are `docker compose down -v`, `docker volume prune`, a bad upgrade, a typo while moving folders, or the disk dying, since every copy is on one machine.

## Before you start
- SSH access as a user that can run `docker` and `sudo`.
- n8n containers: `n8n` and `n8n-postgres`. Data mount inside `n8n` is `/home/node/.n8n` (confirm with `docker inspect n8n --format '{{json .Mounts}}'`).

## Steps
1. Make the folder (owner-only):
   ```
   sudo mkdir -p /opt/x2g-infrastructure/backups/{builder,n8n}
   sudo chown -R $USER: /opt/x2g-infrastructure/backups
   chmod 700 /opt/x2g-infrastructure/backups
   ```
2. Builder (pages, nginx config, and an image of the pages):
   ```
   cd /opt/x2g-infrastructure/backups/builder
   sudo cp -r /var/www/website-builder/form ./site
   sudo cp /etc/nginx/sites-available/builder.xidolabs.cc ./nginx-builder.conf
   printf 'FROM nginx:alpine\nCOPY site /usr/share/nginx/html\n' > Dockerfile
   docker build -t builder:2026-10-14 .
   docker save builder:2026-10-14 | gzip > builder-image.tar.gz
   ```
   Also back up the generated sites n8n writes: `sudo cp -r /opt/website-builder ./website-builder-opt`.
3. n8n:
   ```
   cd /opt/x2g-infrastructure/backups/n8n
   docker exec n8n-postgres sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' | gzip > n8n-db.sql.gz
   docker run --rm --volumes-from n8n -v "$PWD":/backup alpine tar czf /backup/n8n-data.tar.gz /home/node/.n8n
   docker inspect n8n --format '{{json .Config.Env}}' > n8n-env.json
   chmod 600 n8n-env.json
   ```
4. Copy to the PC, **outside any git repo** (Windows PowerShell has no `scp` unless the OpenSSH client is installed; use **Git Bash** or **WSL**):
   ```
   scp -r root@<server-ip>:/opt/x2g-infrastructure/backups "/c/Users/jjose/server-backups/<date>"
   ```
   If files are root-owned, run `sudo chown -R $USER: /opt/x2g-infrastructure/backups` first (not needed as root).

## Check it worked
- Sizes match file by file (`find ... -exec ls -l {} \;` on the server, `Get-ChildItem -Recurse -File` on the PC).
- `sha256sum n8n-db.sql.gz` equals `Get-FileHash ... -Algorithm SHA256` on the PC.
- `tar tzf builder-image.tar.gz | head` lists `manifest.json` and layers.

## Notes
- `n8n-env.json` and `n8n-data.tar.gz` hold the encryption key and credentials: never in git, this vault or chat.
- Restore ideas: `docker load < builder-image.tar.gz`; `gunzip -c n8n-db.sql.gz | docker exec -i n8n-postgres psql ...`. The same encryption key must come back with the data or saved credentials are unreadable.
- The SSH password is the Linux login for the server, not the Supabase `ANON_KEY`.

## Troubleshooting
- `scp : The term 'scp' is not recognized` (PowerShell) → use Git Bash/WSL or install the OpenSSH client (`Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0` as admin).
- `Permission denied` on a file → it is root-owned; `chown` the backups folder.

Related: [[Caddy reverse proxy on Hetzner]], [[Hetzner server layout]]
