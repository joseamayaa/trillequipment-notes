---
type: decision
project: trillequipment
status: accepted
tags: [caddy, nginx, hetzner, platform]
date: 2026-10-15
---

# 2026-10-15 Caddy replaces nginx; server becomes a multi-project host

Hub: [[Trill Equipment]]

## Context
The Hetzner server already ran host-level nginx (with certbot) on 80/443, serving `builder.xidolabs.cc` and proxying `n8n.xidolabs.cc` to a Docker container. The Trill Equipment client dropped the website, so this project is now only a **test** of the setup. The partner wants Caddy as the web server, and the server should host other frontend/backend projects.

## Decision
- Caddy in Docker replaces host nginx on 80/443; the config lives in `/opt/x2g-infrastructure/caddy`.
- Every project is a container on the shared `web` network with no published ports; Caddy routes hostnames to them.
- Cloudflare keeps a proxied wildcard `*.xidolabs.cc`, SSL mode Full (strict), so new projects need no DNS work.
- `trillequipment` stays as a test project to prove the pattern. n8n stays. Builder stays reachable (served by Caddy) because n8n writes the generated sites it serves.
- Details: [[Caddy reverse proxy on Hetzner]], [[Hetzner server layout]]

## Why
- One tool for routing and HTTPS (automatic certificates) and a short, readable config.
- The wildcard plus Caddy makes a new project a Caddyfile block and a compose file.
- Nothing about n8n had to be rebuilt: it was only attached to a network.

## Alternatives considered
- Keep nginx + certbot and add server blocks (lowest risk, but not what the partner wanted).
- Caddy behind nginx, or Caddy on other ports: adds a layer / cannot get certificates.
- A separate small server for this project: costs more.

## Open questions
- Move n8n, builder and Supabase folders into `/opt/x2g-infrastructure/`?
- Share one Supabase for future projects (schema per project), or one stack per project?
- Disable nginx and certbot's renewal timer for good.
