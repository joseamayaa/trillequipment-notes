---
type: runbook
project: trillequipment
tags: [scripts, caddy, supabase, hetzner, template]
updated: 2026-10-23
---

# Project template scripts

The five scripts in `/opt/x2g-infrastructure/bin/` and the one-time Caddy setup they rely on. This note holds the full source so they can be recreated if the server is lost. How to use them day to day: [[Add a project to the server]]. Hub: [[Trill Equipment]]

All five were run and tested on 2026-10-23. Scripts are mode 755, run as root.

## One-time Caddy setup (done 2026-10-23)
Caddy imports one snippet file per project from `/opt/x2g-infrastructure/caddy/sites/`. The scripts only add or delete files there; they never edit the Caddyfile (a single-file bind mount can go stale if an editor replaces the file).
```
cd /opt/x2g-infrastructure/caddy
mkdir -p sites
echo '# Per-project routes live here, one file per project (managed by new-project.sh).' > sites/00-readme.caddy
cat >> Caddyfile <<'EOF'

# One snippet per project, written and removed by new-project.sh / remove-project.sh
import /etc/caddy/sites/*.caddy
EOF
```
In `docker-compose.yaml` add the mount `- ./sites:/etc/caddy/sites:ro` under Caddy's `volumes:`, then **recreate** Caddy: `docker compose config -q && docker compose up -d` (all sites are down for a few seconds; certificates live in the `caddy_caddy_data` volume). The `import` line goes at the **end** of the Caddyfile, after the `(access_log)` snippet it uses.

Check the mount is live: `docker exec caddy ls /etc/caddy/sites` lists `00-readme.caddy`. If `caddy validate` prints `No files matching import glob pattern`, the running container has no `sites` mount (it happened on the first test: validate "passed" but the route never existed, and the new hostname returned Cloudflare 525).

## new-project.sh
`new-project.sh <name> <image> [port] [hostname]` makes `/opt/x2g-infrastructure/<name>/docker-compose.yaml` (service `app`, network `web`, alias `<name>`, no published ports), writes `caddy/sites/<name>.caddy`, validates, reloads Caddy. Rolls back on any failure. Default hostname `<name>.xidolabs.cc` (override the domain with `DOMAIN=`).
```
cat > /opt/x2g-infrastructure/bin/new-project.sh <<'EOF'
#!/usr/bin/env bash
# Usage: new-project.sh <name> <image> [port] [hostname]
# Creates /opt/x2g-infrastructure/<name>/ (compose on the shared `web` network, no published ports),
# adds a Caddy route in caddy/sites/<name>.caddy, validates, and reloads Caddy with no downtime.
# Default hostname: <name>.xidolabs.cc (override the domain with DOMAIN=...). Rolls back on any failure.
set -euo pipefail

BASE=/opt/x2g-infrastructure
DOMAIN=${DOMAIN:-xidolabs.cc}
NAME=${1:-}; IMAGE=${2:-}; PORT=${3:-80}; HOST=${4:-"$NAME.$DOMAIN"}

die() { echo "error: $*" >&2; exit 1; }

[ -n "$NAME" ] && [ -n "$IMAGE" ] || die "usage: $0 <name> <image> [port] [hostname]"
[[ $NAME =~ ^[a-z0-9]([a-z0-9-]*[a-z0-9])?$ ]] || die "name: lowercase letters, digits and dashes only"
[[ $PORT =~ ^[0-9]+$ ]] || die "port must be a number"
case $NAME in
  caddy|n8n|supabase|builder|site|web|bin|backups|migrations|trill|trill-api|trillequipment) die "'$NAME' is reserved" ;;
esac

DIR="$BASE/$NAME"; SNIP="$BASE/caddy/sites/$NAME.caddy"
[ ! -e "$DIR" ]  || die "$DIR already exists"
[ ! -e "$SNIP" ] || die "$SNIP already exists"
docker network inspect web >/dev/null 2>&1 || die "docker network 'web' not found"
docker network inspect web --format '{{range .Containers}}{{.Name}}{{"\n"}}{{end}}' | grep -qx "$NAME" \
  && die "a container named '$NAME' is already on the web network"
grep -rqsF -- "$HOST {" "$BASE/caddy/Caddyfile" "$BASE/caddy/sites" && die "$HOST already has a Caddy route"

created=0; ok=0
cleanup() {
  [ "$ok" = 1 ] && return
  [ "$created" = 1 ] || return
  echo "rolling back..." >&2
  (cd "$DIR" && docker compose down >/dev/null 2>&1) || true
  rm -f "$SNIP"
  rm -rf "$DIR"
}
trap cleanup EXIT

mkdir "$DIR"; created=1

cat > "$DIR/docker-compose.yaml" <<COMPOSE
name: $NAME

services:
  app:
    image: $IMAGE
    restart: unless-stopped
    networks:
      web:
        aliases:
          - $NAME

networks:
  web:
    external: true
COMPOSE

(cd "$DIR" && docker compose config -q && docker compose up -d)

cat > "$SNIP" <<CADDY
$HOST {
    import access_log
    reverse_proxy $NAME:$PORT
}
CADDY

docker exec caddy test -f "/etc/caddy/sites/$NAME.caddy" || die "Caddy cannot see /etc/caddy/sites (is the ./sites mount in its compose file, and was Caddy recreated?)"
if ! out=$(docker exec caddy caddy validate --config /etc/caddy/Caddyfile 2>&1); then
  echo "$out" >&2; die "Caddy config is invalid"
fi
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
ok=1

echo "ok: https://$HOST -> $NAME:$PORT   (folder: $DIR)"
echo "first request: $(curl -s -o /dev/null -w '%{http_code}' -m 30 "https://$HOST" || true)  (the first one can be slow while Caddy gets a certificate)"
EOF
```

## remove-project.sh
Stops the project (no `-v`, volumes are kept), deletes its Caddy route, reloads Caddy, moves the folder to `/opt/x2g-infrastructure/.removed/<name>-<timestamp>`.
```
cat > /opt/x2g-infrastructure/bin/remove-project.sh <<'EOF'
#!/usr/bin/env bash
# Usage: remove-project.sh <name>
# Stops the project (no -v: volumes are kept), removes its Caddy route, reloads Caddy,
# and moves the folder to /opt/x2g-infrastructure/.removed/ instead of deleting it.
set -euo pipefail

BASE=/opt/x2g-infrastructure
NAME=${1:-}
die() { echo "error: $*" >&2; exit 1; }

[ -n "$NAME" ] || die "usage: $0 <name>"
[[ $NAME =~ ^[a-z0-9]([a-z0-9-]*[a-z0-9])?$ ]] || die "bad name"
case $NAME in
  caddy|n8n|supabase|builder|site|web|bin|backups|migrations|trill|trill-api|trillequipment) die "'$NAME' is reserved" ;;
esac

DIR="$BASE/$NAME"; SNIP="$BASE/caddy/sites/$NAME.caddy"
[ -d "$DIR" ] || die "no such project folder: $DIR"
[ -f "$SNIP" ] || echo "warning: $SNIP not found (no Caddy route to remove)"

read -r -p "Remove '$NAME'? Container stops, route is deleted, folder moves to .removed/. [y/N] " a
[ "$a" = y ] || die "cancelled"

(cd "$DIR" && docker compose down)
rm -f "$SNIP"
docker exec caddy caddy validate --config /etc/caddy/Caddyfile >/dev/null
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
mkdir -p "$BASE/.removed"
mv "$DIR" "$BASE/.removed/$NAME-$(date +%F-%H%M%S)"
echo "removed. Volumes were kept: docker volume ls | grep $NAME"
EOF
```

## new-schema.sh
`new-schema.sh <project>` creates a schema in the shared Supabase, grants `anon` / `authenticated` / `service_role` usage and default privileges on its tables (**RLS is then the only protection**), appends it to `PGRST_DB_SCHEMAS` in the stack's `.env` (one rolling copy of the old `.env` goes to `backups/supabase/env.last`, mode 600) and recreates the services that read it (`supabase-studio` and `supabase-rest`; a few seconds of downtime). Needs the compose file's `PGRST_DB_SCHEMAS` lines (43 and 264 at the time).
```
cat > /opt/x2g-infrastructure/bin/new-schema.sh <<'EOF'
#!/usr/bin/env bash
# Usage: new-schema.sh <project>
# Creates a Postgres schema for a project in the shared Supabase, grants the API roles access
# (RLS must be enabled on every table), and exposes the schema to the REST API.
set -euo pipefail

STACK=/opt/x2g-infrastructure/supabase/docker
S=${1:-}
die() { echo "error: $*" >&2; exit 1; }
psqlx() { docker exec -i supabase-db psql -U postgres -d postgres -v ON_ERROR_STOP=1 "$@"; }

[ -n "$S" ] || die "usage: $0 <project>"
[[ $S =~ ^[a-z][a-z0-9_]{0,40}$ ]] || die "name: lowercase letters, digits and underscores, starting with a letter"
case $S in
  public|auth|storage|extensions|graphql|graphql_public|net|realtime|vault|pgbouncer|supabase_functions|x2g_meta|information_schema)
    die "'$S' is reserved" ;;
esac
[ -f "$STACK/.env" ] || die "$STACK/.env not found"
[ -z "$(psqlx -tA -c "select 1 from pg_namespace where nspname='$S'")" ] || die "schema '$S' already exists"

echo "creating schema $S"
psqlx -1 -q <<SQL
create schema "$S";
grant usage on schema "$S" to anon, authenticated, service_role;
grant all on all tables    in schema "$S" to anon, authenticated, service_role;
grant all on all routines  in schema "$S" to anon, authenticated, service_role;
grant all on all sequences in schema "$S" to anon, authenticated, service_role;
alter default privileges in schema "$S" grant all on tables    to anon, authenticated, service_role;
alter default privileges in schema "$S" grant all on routines  to anon, authenticated, service_role;
alter default privileges in schema "$S" grant all on sequences to anon, authenticated, service_role;
SQL

cur=$(grep -E '^PGRST_DB_SCHEMAS=' "$STACK/.env" | head -1 | cut -d= -f2-)
[ -n "$cur" ] || die "PGRST_DB_SCHEMAS not found in .env (the schema was created; add it by hand)"
if [[ ",$cur," != *",$S,"* ]]; then
  install -m 600 "$STACK/.env" /opt/x2g-infrastructure/backups/supabase/env.last
  sed -i "s|^PGRST_DB_SCHEMAS=.*|PGRST_DB_SCHEMAS=$cur,$S|" "$STACK/.env"
fi

echo "recreating the services that read PGRST_DB_SCHEMAS (a few seconds of API/Studio downtime)"
(cd "$STACK" && docker compose up -d >/dev/null)
for i in $(seq 1 30); do
  [ "$(docker inspect -f '{{.State.Health.Status}}' supabase-rest 2>/dev/null)" = healthy ] && break
  sleep 2
done

URL=$(grep -E '^SUPABASE_PUBLIC_URL=' "$STACK/.env" | cut -d= -f2-)
cat <<TXT

ok: schema '$S' is exposed. Exposed schemas now: $(grep -E '^PGRST_DB_SCHEMAS=' "$STACK/.env" | cut -d= -f2-)

Client:   createClient('$URL', '<ANON_KEY>', { db: { schema: '$S' } })
Raw HTTP: add  Accept-Profile: $S  (reads)  or  Content-Profile: $S  (writes)
Migrations: apply-migrations.sh $S <folder-with-sql-files>

Every table in '$S' must run:  alter table $S.<table> enable row level security;
The anon key is public; RLS policies are the only protection. Projects share the anon and
service-role keys and the auth users.
TXT
EOF
```

## apply-migrations.sh
`apply-migrations.sh <project> <folder>` runs the folder's `.sql` files in name order with `search_path` set to the project's schema, each file in one transaction (stops at the first error). Applied files are remembered in `x2g_meta.migrations` (a schema that is never exposed to the API) and skipped on the next run. Names written as `public.xxx` inside a file still go to `public`; a file with its own `BEGIN`/`COMMIT` clashes with the single transaction.

The first version failed with `syntax error near unexpected token 'done'` (a `{ ...; \` continuation, probably a trailing space after the backslash when pasted). This version builds each migration in a temp file instead.
```
cat > /opt/x2g-infrastructure/bin/apply-migrations.sh <<'EOF'
#!/usr/bin/env bash
# Usage: apply-migrations.sh <project> <folder-with-.sql-files>
# Runs the folder's .sql files in name order into the project's schema (search_path is set to it),
# each in one transaction that stops on the first error. Applied files are remembered in
# x2g_meta.migrations (a schema that is not exposed to the API) and skipped next time.
# Unqualified names go to the project schema; names written as public.xxx still go to public.
set -euo pipefail

S=${1:-}; DIR=${2:-}
die() { echo "error: $*" >&2; exit 1; }
psqlx() { docker exec -i supabase-db psql -U postgres -d postgres -v ON_ERROR_STOP=1 "$@"; }

[[ $S =~ ^[a-z][a-z0-9_]{0,40}$ ]] || die "usage: $0 <project> <folder>"
[ -d "$DIR" ] || die "$DIR is not a folder"
[ -n "$(psqlx -tA -c "select 1 from pg_namespace where nspname='$S'")" ] || die "schema '$S' does not exist (run new-schema.sh $S first)"

shopt -s nullglob
files=("$DIR"/*.sql)
[ ${#files[@]} -gt 0 ] || die "no .sql files in $DIR"

psqlx -q <<'SQL'
create schema if not exists x2g_meta;
revoke all on schema x2g_meta from public, anon, authenticated;
create table if not exists x2g_meta.migrations (
  project text not null, name text not null, applied_at timestamptz not null default now(),
  primary key (project, name));
SQL

tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT

for f in "${files[@]}"; do
  base=$(basename "$f")
  [[ $base =~ ^[A-Za-z0-9._-]+$ ]] || die "odd file name: $base"
  seen=$(psqlx -tA -c "select 1 from x2g_meta.migrations where project='$S' and name='$base'")
  if [ -n "$seen" ]; then
    echo "skip   $base"
    continue
  fi
  echo "apply  $base"
  printf 'set search_path to "%s", extensions;\n' "$S" > "$tmp"
  cat "$f" >> "$tmp"
  printf "\ninsert into x2g_meta.migrations (project, name) values ('%s', '%s');\n" "$S" "$base" >> "$tmp"
  psqlx -1 -q < "$tmp"
done
echo "done"
EOF
```

## remove-schema.sh
Dumps the schema to `backups/supabase/schema-<project>-<timestamp>.sql.gz` (mode 600), asks you to type the schema name, drops it with `CASCADE`, forgets its migration history, removes it from `PGRST_DB_SCHEMAS` and recreates Studio/REST.
```
cat > /opt/x2g-infrastructure/bin/remove-schema.sh <<'EOF'
#!/usr/bin/env bash
# Usage: remove-schema.sh <project>
# Dumps the schema to backups/supabase/, asks you to type its name, drops it (CASCADE),
# forgets its migration history and removes it from PGRST_DB_SCHEMAS.
set -euo pipefail

STACK=/opt/x2g-infrastructure/supabase/docker
S=${1:-}
die() { echo "error: $*" >&2; exit 1; }
psqlx() { docker exec -i supabase-db psql -U postgres -d postgres -v ON_ERROR_STOP=1 "$@"; }

[[ $S =~ ^[a-z][a-z0-9_]{0,40}$ ]] || die "usage: $0 <project>"
case $S in
  public|auth|storage|extensions|graphql|graphql_public|net|realtime|vault|pgbouncer|supabase_functions|x2g_meta|information_schema)
    die "'$S' is reserved" ;;
esac
[ -n "$(psqlx -tA -c "select 1 from pg_namespace where nspname='$S'")" ] || die "schema '$S' does not exist"

DUMP=/opt/x2g-infrastructure/backups/supabase/schema-$S-$(date +%F-%H%M%S).sql.gz
docker exec supabase-db pg_dump -U postgres -d postgres -n "$S" | gzip > "$DUMP"
gzip -t "$DUMP"; chmod 600 "$DUMP"
echo "dump saved: $DUMP"

read -r -p "This DROPS schema '$S' and everything in it. Type its name to confirm: " a
[ "$a" = "$S" ] || die "cancelled"

psqlx -1 -q <<SQL
drop schema "$S" cascade;
delete from x2g_meta.migrations where project = '$S';
SQL

cur=$(grep -E '^PGRST_DB_SCHEMAS=' "$STACK/.env" | head -1 | cut -d= -f2-)
new=""; IFS=, read -ra parts <<< "$cur"
for p in "${parts[@]}"; do [ "$p" = "$S" ] || new="${new:+$new,}$p"; done
install -m 600 "$STACK/.env" /opt/x2g-infrastructure/backups/supabase/env.last
sed -i "s|^PGRST_DB_SCHEMAS=.*|PGRST_DB_SCHEMAS=$new|" "$STACK/.env"
(cd "$STACK" && docker compose up -d >/dev/null)
echo "removed. Exposed schemas now: $new"
EOF
```

## Finish
```
chmod 755 /opt/x2g-infrastructure/bin/*.sh
for f in new-project remove-project new-schema apply-migrations remove-schema; do bash -n /opt/x2g-infrastructure/bin/$f.sh && echo "$f OK"; done
```
Run `bash -n` and check it prints OK for all five; a paste can damage a script (see the apply-migrations note).

## Not covered yet
- No authentication on new sites: every new hostname is public (a Basic-auth option is planned).
- `bin/`, `caddy/sites/` and the per-project folders are in the nightly backup (`infra-extra.tgz`, since 2026-10-23, see [[Daily server backups]]). Until a copy is pulled off the server, the notes here are the only copy of the scripts outside the server.

Related: [[Add a project to the server]], [[Caddy reverse proxy on Hetzner]], [[Self-host Supabase on Hetzner]], [[2026-10-18 Shared Supabase and pulled backups]]
