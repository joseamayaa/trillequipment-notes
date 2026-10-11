---
type: hub
project: trillequipment
tags: [hub]
updated: 2026-10-11
---

# Trill Equipment

Website for Trill Equipment, a Houston trailer and heavy equipment rental company. Static HTML site (nginx in Docker) with a Supabase backend for the catalog and reservations.

## Where things live
| What | Where |
|---|---|
| Project (site + `supabase/`) | WSL: `/home/joh/GitHub/trillequipment` |
| These notes | `C:\XidoProjects\trillequipment\TrillEquipmentNotes` (own private repo, see [[Back up these notes]]) |
| Docker image | Docker Hub: `jjohamaya/trillequipment` |
| Test server | Hetzner, runs the image with Docker Compose |
| Final code | GitHub, pushed only when final (partner's rule) |

## Workflow
Local dev → build and push a `test-N` image to Docker Hub → Hetzner pulls it for team testing → GitHub when final.
Details: [[2026-10-11 Docker Hub test pipeline]]

## Runbooks
- [[Build and push a test image]]
- [[Update the Hetzner test server]]
- [[Supabase local dev]]
- [[Supabase token setup]]
- [[Git branch workflow]]
- [[GitHub CLI setup]]
- [[Resume a Claude Code session]]
- [[Back up these notes]]

## Decisions
- [[2026-10-07 Supabase schema & access rules]]
- [[2026-10-11 Docker Hub test pipeline]]

## Logs
- [[2026-10-07 Supabase backend setup]]
- [[2026-10-11 Docker Hub and Hetzner setup]]

## Status and next steps
- [x] Site image built and pushed (`test-1`), running on Hetzner
- [ ] Set the Docker Hub repo to private; add collaborators / server login (access token)
- [ ] Buy a domain, add a DNS A record to the Hetzner server
- [ ] Add HTTPS with Caddy in the compose file
- [ ] Make sure test images point at a **test** Supabase project, not production
- [ ] Create hosted Supabase project(s): `supabase link` + `db push`
- [ ] Commit `Dockerfile`, `.dockerignore`, `docker-compose.yml` to the project's local git
- [ ] Fill in real rates / deposit / phone / address
- [ ] Settle with partner how GitHub fits (nothing pushed until final) and update [[Git branch workflow]]
