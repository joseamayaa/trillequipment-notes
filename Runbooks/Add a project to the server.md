---
type: runbook
project: trillequipment
tags: [hetzner, caddy, supabase, docker, template]
updated: 2026-10-23
---

# Add a project to the server

Put a new test project (a container, plus optionally its own schema in the shared Supabase) on the Hetzner server with a few commands. Hub: [[Trill Equipment]]

The scripts are in `/opt/x2g-infrastructure/bin/` (source and setup in [[Project template scripts]]). Run them as root on the server. Decision behind this: [[2026-10-18 Shared Supabase and pulled backups]].

## Before you start
- The project is a Docker image (on Docker Hub, or already on the server) that listens on one port. Static sites built like Trill (`nginx:alpine` + files) listen on 80.
- A name: lowercase letters, digits and dashes (it becomes the subdomain and the folder). Reserved: `caddy n8n supabase builder site web bin backups migrations trill trill-api trillequipment`.
- No DNS step: Cloudflare has a proxied wildcard `*.xidolabs.cc` (SSL mode Full strict).

## Add the site (container + HTTPS route)
```
sudo /opt/x2g-infrastructure/bin/new-project.sh <name> <image> [port] [hostname]
sudo /opt/x2g-infrastructure/bin/new-project.sh myapp jjohamaya/myapp:test-1 80
```
It creates `/opt/x2g-infrastructure/<name>/docker-compose.yaml` (service `app` on the `web` network, alias `<name>`, **no published ports**), writes `/opt/x2g-infrastructure/caddy/sites/<name>.caddy`, checks that Caddy can see it, validates the config and reloads Caddy with no downtime. If anything fails it rolls back (container, route, folder). Default hostname `<name>.xidolabs.cc`; use the 4th argument for another one.

The first request to a new hostname can return Cloudflare `525` for a few seconds while Caddy gets its certificate; wait about 20 seconds and retry. Tested with `nginx:alpine`: 525, then 200.

Update the image later: edit the tag in `/opt/x2g-infrastructure/<name>/docker-compose.yaml`, then `docker compose pull && docker compose up -d` in that folder ([[Update the Hetzner test server]]).

## Give it a database (shared Supabase, one schema per project)
```
sudo /opt/x2g-infrastructure/bin/new-schema.sh <project_name>          # underscores, not dashes
sudo /opt/x2g-infrastructure/bin/apply-migrations.sh <project_name> <folder-with-sql>
```
- `new-schema.sh` creates the schema, grants `anon` / `authenticated` / `service_role` access, adds it to `PGRST_DB_SCHEMAS` and recreates Studio and the REST service (a few seconds of API and Studio downtime). It prints the client snippet.
- `apply-migrations.sh` runs the folder's `.sql` files in name order into that schema, one transaction per file, stops at the first error, and skips files it already ran (history kept in `x2g_meta.migrations`, not exposed to the API). Write unqualified table names; `public.xxx` would land in `public`.
- The site talks to Supabase from the browser with `supabase-js` and the anon key: `createClient('https://trill-api.xidolabs.cc', '<ANON_KEY>', { db: { schema: '<project_name>' } })`. Raw HTTP: header `Accept-Profile: <project_name>` to read, `Content-Profile` to write. Never put the service-role key in a site.
- **Every table must enable RLS** (`alter table <schema>.<table> enable row level security;`) with policies. The anon key is public, so RLS is the only protection. Projects share the anon and service-role keys and the auth users, so this is for a small trusted team's test projects, not sensitive data.

Check as the public key (read the key from the file; never paste it anywhere):
```
cd /opt/x2g-infrastructure/supabase/docker
ANON=$(grep '^ANON_KEY=' .env | cut -d= -f2-)
curl -s -H "apikey: $ANON" -H "Authorization: Bearer $ANON" -H "Accept-Profile: <project_name>" https://trill-api.xidolabs.cc/rest/v1/<table>; echo
unset ANON
```
Tested on 2026-10-23 with a throwaway schema: reading with the schema header returned the row; the same table without the header gave `PGRST205` (not in `public`); an anon insert without a policy gave `42501 violates row-level security`.

## Remove a project
```
sudo /opt/x2g-infrastructure/bin/remove-project.sh <name>        # asks y/N
sudo /opt/x2g-infrastructure/bin/remove-schema.sh <project_name> # dumps first, then asks you to type the name
```
- `remove-project.sh` stops the container (volumes are **kept**), deletes the Caddy route, reloads Caddy and moves the folder to `/opt/x2g-infrastructure/.removed/<name>-<timestamp>`. Delete that folder by hand when you are sure.
- `remove-schema.sh` saves `backups/supabase/schema-<project>-<timestamp>.sql.gz` (mode 600), drops the schema with `CASCADE`, forgets its migrations and removes it from the API's list.

## Check it worked
- `curl -sI https://<name>.xidolabs.cc | head -3` returns `200` (after the certificate wait).
- `docker compose ps` in the project folder shows `app` Up. `ls /opt/x2g-infrastructure/caddy/sites` lists `<name>.caddy`.
- For a schema: `docker exec supabase-db psql -U postgres -d postgres -c '\dn'` lists it; `grep '^PGRST_DB_SCHEMAS=' /opt/x2g-infrastructure/supabase/docker/.env` includes it.

## Troubleshooting
- `error: Caddy cannot see /etc/caddy/sites` or `caddy validate` warns `No files matching import glob pattern` → Caddy's container has no `./sites` mount; fix the compose file and recreate Caddy ([[Project template scripts]]).
- `525` that does not go away → `docker compose logs caddy` (certificate errors); see [[Caddy reverse proxy on Hetzner]].
- `502` → the container is not on `web`, uses another port, or is not up (`docker compose ps`); the route points at `<name>:<port>`.
- `error: schema ... already exists` / `does not exist` → the script's guard; check `\dn`.
- Every new hostname is **public**, and scanners find new names within hours (check `caddy/logs/access.log`). Basic auth or Cloudflare Access is not built yet; do not put anything private on a test hostname.

Related: [[Project template scripts]], [[Hetzner server layout]], [[Daily server backups]]
