---
type: runbook
project: trillequipment
tags: [supabase, database]
updated: 2026-10-08
---

# Supabase local dev

Run the Supabase backend on my machine for testing. Hub: [[Trill Equipment]]

## Before you start
- Docker Desktop must be running (open it from the Start menu, wait for "Engine running").
- Run every command from the repo folder: `C:\GitHub Repos\trillequipment`
- No install needed: `npx supabase ...` downloads the CLI automatically.

## Start
```
npx supabase start
```
Prints the local URLs and keys. Local keys are demo keys, safe to use on my machine only.
- API: http://127.0.0.1:54321
- Studio (database UI): http://127.0.0.1:54323

Lighter start (skips extra services):
```
npx supabase start -x studio,imgproxy,edge-runtime,logflare,vector,realtime,supavisor,postgres-meta,mailpit
```

## Stop
```
npx supabase stop
```

## Reset the database
Wipes local data, re-runs all migrations, then loads `supabase/seed.sql` (fake `DEMO-…` data).
```
npx supabase db reset
```

## Change the schema
1. Create a new migration file:
   ```
   npx supabase migration new describe_the_change
   ```
2. Write the SQL in the new file under `supabase/migrations/`.
3. Apply and check:
   ```
   npx supabase db reset
   npx supabase db lint
   ```
4. Commit the migration. Never edit a migration that's already been pushed to production; add a new one instead.

## Where things are
| File | What it is |
|---|---|
| `supabase/config.toml` | Local settings (auth, storage, ports) |
| `supabase/migrations/` | Schema changes, applied in order, also go to production |
| `supabase/seed.sql` | Fake data, local only |

## Troubleshooting
- `failed to connect to the docker API` → Docker Desktop isn't running. Start it and wait.
- Port already in use → another Supabase project is running: `npx supabase stop --all`

Related: [[2026-10-07 Supabase schema & access rules]]
