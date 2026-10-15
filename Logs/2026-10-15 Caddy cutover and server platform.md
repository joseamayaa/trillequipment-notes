---
type: log
project: trillequipment
tags: [caddy, hetzner, docker, n8n, cloudflare]
date: 2026-10-15
---

# 2026-10-15 Caddy cutover and server platform

Hub: [[Trill Equipment]] · Worked with Claude Code · Continues [[2026-10-14 Self-hosting Supabase on Hetzner]]

## Done
- The client dropped the Trill site, so it is now a test project; the goal became a general Docker + Caddy host for other projects → [[2026-10-15 Caddy replaces nginx]]
- Found host nginx on 80/443 serving `builder.xidolabs.cc` and proxying `n8n.xidolabs.cc` (n8n runs in Docker: `n8n` + `n8n-postgres`, named volumes). Found that n8n mounts `/opt/website-builder`, so the builder and n8n are linked (n8n writes generated sites to `/opt/website-builder/projects`).
- Took backups of builder and n8n into `/opt/x2g-infrastructure/backups`, saved the builder pages as a Docker image, and walked through copying them to the PC (PowerShell had no `scp`; use Git Bash/WSL) → [[Back up n8n and builder]]
- Created the `web` Docker network and attached `n8n` and `trillequipment-site-1`.
- Wrote the Caddy project (`Caddyfile`, compose with log files, `logs/`) and cut over: stopped nginx, started Caddy. Certificates were issued, and `n8n`, `trill` and `trill-api` work. Fixed a compose `networks must be a mapping` error → [[Caddy reverse proxy on Hetzner]]
- Added `builder.xidolabs.cc` to Caddy (pages from `/var/www/website-builder/form`, `/projects/` with a directory listing from `/opt/website-builder/projects`). Everything works.
- Cloudflare facts: DNS is on Cloudflare, the wildcard `*.xidolabs.cc` is proxied, SSL mode is Full (strict). No new DNS records were needed.
- Documented the layout of `/opt/x2g-infrastructure` and the rest of the server → [[Hetzner server layout]]

## Roadmap
- [x] Phase 0: Supabase running, backups taken, `web` network
- [x] Phase 1: decisions (Cloudflare DNS, hostnames `trill.` / `trill-api.`, n8n stays)
- [x] Phase 2: Caddy replaces nginx (n8n, trill, trill-api, builder)
- [ ] Phase 3: the Trill test over HTTPS (Supabase URLs, `config.js`, `test-2`, schema + RLS)
- [ ] Phase 4: hardening (firewall, pooler ports, SSH, scheduled backups)
- [ ] Phase 5: reusable platform (project template, move folders into `/opt/x2g-infrastructure`, persist `web` in compose files)

## Commits / PRs
- Notes only; nothing committed in the project.

## Next
- [ ] **`sudo systemctl disable nginx`** (check `is-enabled` first). Otherwise a reboot starts nginx before Caddy and Caddy cannot bind 80/443. Also disable certbot's renewal timer once stable
- [ ] Add the `web` network (external) to the compose files of `n8n` (`/opt/n8n`) and `trillequipment`, with the alias `site`; the current attachments are lost if either container is recreated
- [ ] Confirm the backups were copied to the PC and sizes/hashes match; keep them outside git
- [ ] Phase 3: set the Supabase `.env` URLs to `https://trill-api.xidolabs.cc` (keep `/auth/v1` on `API_EXTERNAL_URL`) and `SITE_URL=https://trill.xidolabs.cc`, then `down` + `up -d`
- [ ] Apply the project's SQL (see [[Apply the schema to self-hosted Supabase]]), RLS on every table
- [ ] Add `config.js` + `supabase-js` to the static site, build `test-2` with the HTTPS URL, push, update the server's tag
- [ ] Close port 8000; confirm the firewall allows only 22/80/443 and blocks 5432/6543 (`Test-NetConnection <server-ip> -Port 5432` should fail)
- [ ] Look inside `/opt/website-builder/projects` (mode 777, served publicly with a listing) for anything that should not be public
- [ ] Decide whether to move n8n / builder / Supabase folders into `/opt/x2g-infrastructure/`
- [ ] Back up Caddy's `caddy_data` volume (its account key and certificates) or accept re-issuance

## Notes
- The builder container was never started; Caddy serves the pages directly from mounted folders (read-only).
- Host nginx is stopped but its configs and certbot certificates were kept, so rollback is `docker compose stop caddy && sudo systemctl start nginx`.
- Stopping a container keeps its data; volumes are lost only by `down -v`, `volume prune` or disk loss.
