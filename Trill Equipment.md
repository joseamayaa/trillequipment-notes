---
type: hub
project: trillequipment
tags: [hub]
updated: 2026-10-18
---

# Trill Equipment

Website for Trill Equipment, a Houston trailer and heavy equipment rental company. Static HTML site (nginx in Docker) with a Supabase backend for the catalog and reservations.

**Status (2026-10-18):** the client dropped the website. This project is now the **test project** for a general multi-project Hetzner server (Docker + Caddy + a shared Supabase). The Trill-specific work (schema, form, `test-2`) is **parked**; the work now is the platform. See [[2026-10-15 Caddy replaces nginx]] and [[2026-10-18 Shared Supabase and pulled backups]].

## Where things live
| What | Where |
|---|---|
| Project (site + `supabase/`) | WSL: `/home/joh/GitHub/trillequipment` (branch `Joh's-Branch`; `Dockerfile` and `.dockerignore` committed) |
| These notes | `C:\XidoProjects\trillequipment\TrillEquipmentNotes` (own private repo, see [[Back up these notes]]) |
| Docker image | Docker Hub: `jjohamaya/trillequipment` |
| Test server | Hetzner; Caddy in Docker serves `https://trill.xidolabs.cc` and `https://trill-api.xidolabs.cc`; layout in [[Hetzner server layout]] |
| Final code | GitHub, pushed only when final (partner's rule) |

## Workflow
Local dev → build and push a `test-N` image to Docker Hub → Hetzner pulls it for team testing → GitHub when final.
Details: [[2026-10-11 Docker Hub test pipeline]]

## Runbooks
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

## Status and next steps

### Done
- [x] Site image built and pushed (`test-1`), running on Hetzner
- [x] Supabase self-hosted on Hetzner, now at `https://trill-api.xidolabs.cc` (URLs in its `.env` are `https://`)
- [x] Caddy (Docker) replaced host nginx; HTTPS for `n8n.`, `builder.`, `trill.`, `trill-api.xidolabs.cc` (Cloudflare wildcard, Full strict)
- [x] `sudo systemctl disable nginx` (done, `disabled` / `inactive`)
- [x] `web` network is in the compose files (Caddy, n8n, trillequipment); n8n moved to `/opt/x2g-infrastructure/n8n`
- [x] Firewall from the PC: only 22 / 80 / 443 answer; pooler ports and n8n bound to localhost; Supabase gateway port 8000 bound to `127.0.0.1`
- [x] Supabase stack, test site and migrations are in `/opt/x2g-infrastructure/`
- [x] Decided: shared Supabase with a schema per project; backups pulled to the PC
- [x] Studio login works over HTTPS; firewall rules for 5678 and 8000 deleted in both UFW and the Hetzner Cloud Firewall

### Next: platform setup (details in [[2026-10-18 n8n move, firewall and Supabase over HTTPS]])
- [ ] **Run the backup script**, schedule it, pull one copy to the PC, do a restore test → [[Daily server backups]]
- [ ] Caddy `import sites/*.caddy` + `new-project.sh` / `remove-project.sh` + templates (static site, and a Supabase schema-per-project path)
- [ ] Studio hardening: own hostname behind Cloudflare Access; API hostname limited to the API paths
- [ ] **Confirm (still unchecked):** n8n workflows and credentials intact after the move; the `trillequipment` recreate (only `caddy` and `trillequipment-site-1` listed, site on `web`, `trill.` returns 200)
- [ ] **Reminder, on or after 2026-10-22: delete the old copies** (`~/migrations.old-delete-me`, `~/supabase.old-delete-me`, and `/opt/n8n.old-delete-me` once renamed with `mv /opt/n8n /opt/n8n.old-delete-me`); checks in [[Hetzner server layout]]
- [ ] Secrets in a password manager; delete `.env.bak-2026-10-17` / `docker-compose.yml.bak-2026-10-17` in the Supabase folder
- [ ] Look inside `/opt/website-builder/projects` (mode 777, public listing) for anything that should not be public
- [ ] Decide whether to move `/opt/website-builder` and `/var/www/website-builder` into `/opt/x2g-infrastructure/`; drop the unused `n8n-postgres` service if confirmed unused

### Parked (Trill-specific; see Notes in the 2026-10-18 log)
- [ ] Apply the project's SQL schema (needs a schema prefix first for the shared stack; catalog is missing 6 of the site's 14 equipment types)
- [ ] Connect the static site (`config.js` + `supabase-js`), build `test-2`, redeploy
- [ ] Set the Docker Hub repo to private; add collaborators / server login (access token)
- [ ] Make sure test images point at a **test** Supabase project, not production
- [ ] Create hosted Supabase project(s): `supabase link` + `db push`
- [ ] Fill in real rates / deposit / phone / address (only if the site is ever revived)
- [ ] Settle with partner how GitHub fits (nothing pushed until final) and update [[Git branch workflow]]
