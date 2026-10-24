---
type: decision
project: trillequipment
status: accepted
tags: [supabase, backups, platform, hetzner]
date: 2026-10-18
---

# 2026-10-18 Shared Supabase, schema per project; backups pulled to the PC

Hub: [[Trill Equipment]]

## Context
The server is a general test host for several projects ([[2026-10-15 Caddy replaces nginx]]). Trill itself is parked. Two choices were open: how future projects use Supabase, and where backups go.

## Decision
- **One shared Supabase stack** (the one running in `/opt/x2g-infrastructure/supabase/docker`). Each project gets **its own Postgres schema** with its own RLS, not tables in `public`.
- **Backups:** daily script on the server, 7 days kept; the copy off the server is **pulled to the PC on demand** with `scp` (no new account or cost) → [[Daily server backups]].
- Scope for now: only what the platform needs (backups, Caddy snippet + `new-project.sh`, hardening). Trill-specific work stays parked.

Implemented 2026-10-23: `new-schema.sh`, `apply-migrations.sh`, `remove-schema.sh` ([[Add a project to the server]]).

## Why
- A full Supabase stack is about 11 containers; one per project would not fit many projects on a small server.
- Pulling to the PC costs nothing and needs no keys on the server. Trade-off: it protects only when a backup is actually pulled.

## Trade-offs to remember
- Projects **share the anon and service-role keys and the auth users**, so isolation is weaker. A project holding the service key can read everything. Fine for a small trusted team testing; do not host anything sensitive or client-facing this way.
- Trill's migration puts types, tables and functions in `public`; it should **not** be applied to the shared stack as it is (name collisions, projects seeing each other's tables). It would need a schema prefix first.
- Each extra schema must be exposed to the API (`PGRST_DB_SCHEMAS` in the stack's `.env`) and requested by the client (`Accept-Profile` / `Content-Profile` header, or the `db.schema` option of `supabase-js`), with grants for `anon` / `authenticated` and RLS on every table.

## Alternatives considered
- One full stack per project: full isolation, too heavy for this server.
- Hetzner Storage Box or an S3-compatible bucket (R2/B2) with automatic push: better protection, needs an account and a key on the server; can be added later.

## Open questions
- Naming and ownership of project schemas, and who may create them.
- Whether to add an automatic off-server push later.
