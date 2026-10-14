---
type: runbook
project: trillequipment
tags: [supabase, hetzner, migrations]
updated: 2026-10-14
---

# Apply the schema to self-hosted Supabase

Load the project's SQL (tables, RLS) into the Hetzner Supabase database so it shows in Studio. Hub: [[Trill Equipment]]

## Before you start
- The stack is running: [[Self-host Supabase on Hetzner]].
- Find the SQL in the project (WSL: `/home/joh/GitHub/trillequipment`):
  ```
  find . \( -name "*.sql" -o -name "supabase" \) -not -path "./.git/*"
  ```
- `<user>` = the Linux user I SSH into the server with (`whoami` on the server).

## Option A: Studio SQL Editor (few files)
Open `http://<server-ip>:8000` → SQL Editor → paste each file → Run, in filename (timestamp) order.

## Option B: copy to the server and run psql (many files)
On **my PC** (scp copies from the PC):
```
ssh <user>@<server-ip> "mkdir -p ~/migrations"
scp supabase/migrations/*.sql <user>@<server-ip>:~/migrations/
```
On the **server** (needs docker):
```
for f in ~/migrations/*.sql; do
  docker exec -i supabase-db psql -U postgres -d postgres < "$f"
done
```
Port 5432 stays closed; no tunnel needed this way.

## Check it worked
- Studio → Table Editor / Database → Tables shows the tables in `public`.
- **RLS is enabled on every table** and policies exist (Studio flags tables with RLS off). The anon key is public, so RLS is the only protection. See [[2026-10-07 Supabase schema & access rules]].

## Troubleshooting
- No `.sql` files in the repo → the schema isn't there; it may live in a hosted Supabase project (dump with `supabase db dump`) or be built in Studio.
- Errors from running files out of order → apply in timestamp order.

Related: [[Supabase local dev]]
