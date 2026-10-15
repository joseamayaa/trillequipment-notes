---
type: log
project: trillequipment
tags: [supabase, hetzner, docker]
date: 2026-10-14
---

# 2026-10-14 Self-hosting Supabase on Hetzner

Hub: [[Trill Equipment]] · Worked with Claude Code

## Done
- Created `.env` from `.env.example` in the cloned `supabase/docker/` folder on the server, ran `utils/generate-keys.sh` (answered yes to updating `.env`), then set the URLs (`SUPABASE_PUBLIC_URL`, `API_EXTERNAL_URL` keeping its `/auth/v1` suffix, `SITE_URL`) → [[Self-host Supabase on Hetzner]]
- Started the stack with `docker compose up -d`; all services healthy.
- Debugged a `403 RBAC: access denied` from `/rest/v1/`: the gateway in this Supabase version is **Envoy**, and the bare REST root is restricted for the anon key. `/auth/v1/health` returns `200`, so the stack works.
- Studio loads at `http://<server-ip>:8000` and the dashboard login works.
- Worked out how the static site connects to the backend: the browser calls Supabase directly with `supabase-js` and the anon key; no Node.js needed (the image is `nginx:alpine` serving static files, so config is baked into the image, not read at runtime).
- Chose to get the project's schema into Studio by applying SQL migrations → [[Apply the schema to self-hosted Supabase]]

## Commits / PRs
- None in the project.

## Next
- [ ] **Firewall check:** the pooler publishes 5432/6543 on `0.0.0.0`. Confirm the Hetzner firewall allows only 22 and 8000 (and 80 for the site); from the PC, `Test-NetConnection <server-ip> -Port 5432` should fail
- [ ] Find the project's SQL (`find` for `*.sql` / `supabase/`) and apply it ([[Apply the schema to self-hosted Supabase]]); confirm tables appear in Studio
- [ ] Turn on RLS + policies on every table before the site uses the API
- [ ] Connect the static site: `config.js` with the server URL and anon key, load `supabase-js` (pinned CDN version) in `index.html`, never the service role key
- [ ] Rebuild the image as `test-2`, push, update the tag in the server's `docker-compose.yaml`, `pull` + `up -d` ([[Build and push a test image]], [[Update the Hetzner test server]])
- [ ] Set `SITE_URL` in the Supabase `.env` to the real frontend URL, then `down` + `up -d`
- [ ] **Next session: Caddy + HTTPS.** Needs a domain (two A records, e.g. `app.` and `api.`). Plan: Caddy on 80/443 in front of `site` and `supabase-envoy`, shared Docker network, `site` stops publishing port 80, close 8000 afterward; then update `config.js` and the Supabase URLs to `https://`
- [ ] Back up the stack's `.env` values in a password manager (never in this vault)

## Continued
Caddy, HTTPS and the nginx cutover happened next: [[2026-10-15 Caddy cutover and server platform]].

## Notes
- The anon key is public by design; `SERVICE_ROLE_KEY`, `JWT_SECRET` and `POSTGRES_PASSWORD` must never be pasted anywhere or put in the site.
- A static site can't read runtime env vars; changing the Supabase URL means rebuilding and pushing a new image tag.
- Not verified yet today: that the schema was applied, and that the firewall blocks 5432/6543.
