---
type: hub
project: trillequipment
tags: [hub]
updated: 2026-10-15
---

# Trill Equipment

Website for Trill Equipment, a Houston trailer and heavy equipment rental company. Static HTML site (nginx in Docker) with a Supabase backend for the catalog and reservations.

**Status (2026-10-15):** the client dropped the website. This project is now the **test project** for a general multi-project Hetzner server (Docker + Caddy). See [[2026-10-15 Caddy replaces nginx]].

## Where things live
| What | Where |
|---|---|
| Project (site + `supabase/`) | WSL: `/home/joh/GitHub/trillequipment` |
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

## Logs
- [[2026-10-07 Supabase backend setup]]
- [[2026-10-11 Docker Hub and Hetzner setup]]
- [[2026-10-14 Self-hosting Supabase on Hetzner]]
- [[2026-10-15 Caddy cutover and server platform]]

## Status and next steps
- [x] Site image built and pushed (`test-1`), running on Hetzner
- [x] Supabase self-hosted on Hetzner (port 8000), Studio login works
- [x] Caddy (Docker) replaced host nginx; HTTPS for `n8n.`, `builder.`, `trill.`, `trill-api.xidolabs.cc` (Cloudflare wildcard, Full strict)
- [ ] **`sudo systemctl disable nginx`** so a reboot cannot grab 80/443 before Caddy
- [ ] Put the `web` network in the `n8n` and `trillequipment` compose files (the current attachment is lost on recreate)
- [ ] Point Supabase and the site at the `https://` URLs, then close port 8000
- [ ] Check the firewall: only 22/80/443; block 5432/6543 (pooler publishes them)
- [ ] Apply the project's SQL schema to the server, RLS on every table
- [ ] Connect the static site (`config.js` + `supabase-js`), build `test-2`, redeploy
- [ ] Confirm the n8n / builder backups were copied to the PC
- [ ] Hardening, scheduled backups, project template (roadmap in [[2026-10-15 Caddy cutover and server platform]])
- [ ] Set the Docker Hub repo to private; add collaborators / server login (access token)
- [ ] Make sure test images point at a **test** Supabase project, not production
- [ ] Create hosted Supabase project(s): `supabase link` + `db push`
- [ ] Commit `Dockerfile`, `.dockerignore`, `docker-compose.yml` to the project's local git
- [ ] Fill in real rates / deposit / phone / address (only if the site is ever revived)
- [ ] Settle with partner how GitHub fits (nothing pushed until final) and update [[Git branch workflow]]
