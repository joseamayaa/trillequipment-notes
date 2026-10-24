---
type: hub
project: trillequipment
tags: [hub]
updated: 2026-10-23
---

# Trill Equipment

Website for Trill Equipment, a Houston trailer and heavy equipment rental company. Static HTML site (nginx in Docker) with a Supabase backend for the catalog and reservations.

**Status (2026-10-23):** the client dropped the website. This project is now the **test project** for a general multi-project Hetzner server (Docker + Caddy + a shared Supabase). The server template is built: one command adds a project (container, HTTPS route) and another gives it a schema in the shared Supabase → [[Add a project to the server]]. The Trill-specific work (schema, form, `test-2`) is **parked**. See [[2026-10-15 Caddy replaces nginx]] and [[2026-10-18 Shared Supabase and pulled backups]].

## Where things live
| What | Where |
|---|---|
| Project (site + `supabase/`) | WSL: `/home/joh/GitHub/trillequipment` (branch `Joh's-Branch`; `Dockerfile` and `.dockerignore` committed) |
| These notes | `C:\XidoProjects\trillequipment\TrillEquipmentNotes` (own private repo, see [[Back up these notes]]) |
| Docker image | Docker Hub: `jjohamaya/trillequipment` |
| Test server | Hetzner; Caddy in Docker serves `https://trill.xidolabs.cc` and `https://trill-api.xidolabs.cc`; layout in [[Hetzner server layout]] |
| Server scripts | `/opt/x2g-infrastructure/bin/` (source in [[Project template scripts]]) |
| Final code | GitHub, pushed only when final (partner's rule) |

## Workflow
Local dev → build and push a `test-N` image to Docker Hub → Hetzner pulls it for team testing → GitHub when final.
Details: [[2026-10-11 Docker Hub test pipeline]]. New project on the server: [[Add a project to the server]].

## Runbooks
- [[Add a project to the server]]
- [[Project template scripts]]
- [[Harden Studio and API access]]
- [[Build and push a test image]]
- [[Update the Hetzner test server]]
- [[Hetzner server layout]]
- [[Caddy reverse proxy on Hetzner]]
- [[Daily server backups]]
- [[Back up n8n and builder]]
- [[Self-host Supabase on Hetzner]]
- [[Apply the schema to self-hosted Supabase]]
- [[Supabase local dev]]
- [[Supabase token setup]]
- [[Git branch workflow]]
- [[GitHub CLI setup]]
- [[Resume a Claude Code session]]
- [[Back up these notes]]

## Decisions
- [[2026-10-07 Supabase schema & access rules]]
- [[2026-10-11 Docker Hub test pipeline]]
- [[2026-10-15 Caddy replaces nginx]]
- [[2026-10-18 Shared Supabase and pulled backups]]

## Logs
- [[2026-10-07 Supabase backend setup]]
- [[2026-10-11 Docker Hub and Hetzner setup]]
- [[2026-10-14 Self-hosting Supabase on Hetzner]]
- [[2026-10-15 Caddy cutover and server platform]]
- [[2026-10-18 n8n move, firewall and Supabase over HTTPS]]
- [[2026-10-23 Project template scripts and cleanup]]

## Status and next steps

### Done
- [x] Site image built and pushed (`test-1`), running on Hetzner
- [x] Supabase self-hosted on Hetzner, at `https://trill-api.xidolabs.cc`; gateway port 8000 bound to localhost
- [x] Caddy (Docker) replaced host nginx (disabled); HTTPS for `n8n.`, `builder.`, `trill.`, `trill-api.xidolabs.cc` (Cloudflare wildcard, Full strict)
- [x] `web` network declared in the Caddy, n8n and trillequipment compose files; n8n moved to `/opt/x2g-infrastructure/n8n`
- [x] Firewall: only 22 / 80 / 443 answer from outside; the 5678 and 8000 rules are gone in UFW and Hetzner
- [x] Decided: shared Supabase with a schema per project; backups pulled to the PC
- [x] **Project template built and tested (2026-10-23):** `new-project.sh`, `remove-project.sh`, `new-schema.sh`, `apply-migrations.sh`, `remove-schema.sh` → [[Add a project to the server]]
- [x] New sites are password-protected by default (Caddy Basic auth, random password printed once; `--public` to opt out) → [[Add a project to the server]]
- [x] Old copies deleted (`~/migrations`, `/root/supabase`, `/opt/n8n`); disk back to 22 GB used
- [x] Nightly backup script (03:15 cron) covers Supabase, n8n, Caddy, `bin/`, `caddy/sites/` and project folders; run by hand 2026-10-23: 9 files, 59 MB → [[Daily server backups]]

### Next: platform (details in [[2026-10-23 Project template scripts and cleanup]])
- [ ] **Backups: get a copy to the PC (SSH key); do the restore tests; confirm the first scheduled run happened (`tail /var/log/x2g-backup.log` after 03:15)**
- [ ] Studio hardening: own hostname behind Cloudflare Access; API hostname limited to the API paths. **Plan written (about 30 to 40 minutes), not run** → [[Harden Studio and API access]]
- [ ] Small cleanups: `.removed/demo-*`, the `.bak-*` files (the Supabase `.env.bak-2026-10-17` holds every secret; the values should be in a password manager first) and `backups/supabase/env.last`
- [ ] Look inside `/opt/website-builder/projects` (mode 777, public listing) for anything that should not be public
- [ ] Decide whether to move `/opt/website-builder` and `/var/www/website-builder` into `/opt/x2g-infrastructure/`; drop the unused `n8n-postgres` service if confirmed unused

### Parked (Trill-specific; see Notes in the 2026-10-18 log)
- [ ] Apply the project's SQL schema (needs the `public.` prefix removed for the shared stack; the catalog is missing 6 of the site's 14 equipment types)
- [ ] Connect the static site (`config.js` + `supabase-js`), build `test-2`, redeploy
- [ ] Set the Docker Hub repo to private; add collaborators / server login (access token)
- [ ] Make sure test images point at a **test** Supabase project, not production
- [ ] Create hosted Supabase project(s): `supabase link` + `db push`
- [ ] Fill in real rates / deposit / phone / address (only if the site is ever revived)
- [ ] Settle with partner how GitHub fits (nothing pushed until final) and update [[Git branch workflow]]
