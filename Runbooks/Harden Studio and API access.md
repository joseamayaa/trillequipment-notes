---
type: runbook
project: trillequipment
tags: [supabase, caddy, cloudflare, security, hetzner]
updated: 2026-10-23
---

# Harden Studio and API access

Give Supabase Studio its own hostname and make the API hostname answer only API paths. Hub: [[Trill Equipment]]

**Status: planned 2026-10-23, NOT run yet.** Nothing here has been tested. Estimated time: about 30 to 40 minutes (Part 1 about 10 minutes; Part 2 about 15 to 25 minutes, mostly in the Cloudflare dashboard).

## Why
Today `trill-api.xidolabs.cc` serves everything through Envoy: the API and Studio (behind Envoy's Basic-auth login, `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD`). New hostnames get scanned within hours, so Studio should not sit on the public API hostname, and a second gate in front of it is cheap. Related: [[Hetzner server layout]], [[Self-host Supabase on Hetzner]], [[Caddy reverse proxy on Hetzner]].

## Target
- `trill-api.xidolabs.cc` forwards only `/rest/*`, `/auth/*`, `/storage/*`, `/realtime/*`, `/functions/*`, `/graphql/*` to `supabase-envoy:8000`; everything else returns 404.
- `studio.xidolabs.cc` forwards everything to `supabase-envoy:8000` (still behind Envoy's login), and later sits behind Cloudflare Access (Part 2).
- The Supabase `.env` URLs stay as they are (`SUPABASE_PUBLIC_URL=https://trill-api.xidolabs.cc`). The Cloudflare wildcard `*.xidolabs.cc` already covers `studio.`, so no DNS step.

## Before you start
- A catch with the Caddyfile: it is a **single-file bind mount**. Editing it with `nano` or `sed -i` replaces the file, and the container keeps seeing the old copy, so a reload does nothing. Overwrite it with `cat > Caddyfile` (keeps the same file) or put routes in `sites/` (a mounted folder, always live).
- Back up first: `cp -a Caddyfile Caddyfile.bak-studio`.

## Part 1: split Studio from the API in Caddy
In `/opt/x2g-infrastructure/caddy`:
1. Create `sites/trill-api.caddy`:
   ```
   trill-api.xidolabs.cc {
       import access_log
       @api path /rest/* /auth/* /storage/* /realtime/* /functions/* /graphql/*
       handle @api {
           reverse_proxy supabase-envoy:8000
       }
       handle {
           respond "Not found" 404
       }
   }
   ```
2. Create `sites/studio.caddy`:
   ```
   studio.xidolabs.cc {
       import access_log
       reverse_proxy supabase-envoy:8000
   }
   ```
3. Remove the old `trill-api.xidolabs.cc { ... }` block from the main Caddyfile (a duplicate site block fails validation). Overwrite the file with `cat > Caddyfile <<'EOF'` holding: the global `{ log { ... } }` block, the `(access_log)` snippet, the `n8n.`, `builder.` and `trill.` blocks exactly as they are, and at the end the `import /etc/caddy/sites/*.caddy` line (copy the current file, delete only the `trill-api` block).
4. Check the container sees the new file and the config is valid:
   ```
   docker exec caddy grep -c 'trill-api' /etc/caddy/Caddyfile      # must print 0
   docker exec caddy caddy validate --config /etc/caddy/Caddyfile 2>&1 | tail -2   # Valid configuration
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```
5. Test (read the anon key from the file, never paste it):
   ```
   cd /opt/x2g-infrastructure/supabase/docker
   ANON=$(grep '^ANON_KEY=' .env | cut -d= -f2-)
   curl -s -o /dev/null -w 'api root:   %{http_code} (404 expected)\n' https://trill-api.xidolabs.cc/
   curl -s -o /dev/null -w 'api health: %{http_code} (200 expected)\n' -H "apikey: $ANON" https://trill-api.xidolabs.cc/auth/v1/health
   curl -s -o /dev/null -w 'studio:     %{http_code} (401 expected)\n' https://studio.xidolabs.cc/
   unset ANON
   ```
   Also check `n8n`, `trill` and `builder` still return 200, and open `https://studio.xidolabs.cc` in a browser, log in, and try the Table Editor and the SQL editor.
6. Rollback: `cat Caddyfile.bak-studio > Caddyfile; rm sites/trill-api.caddy sites/studio.caddy; docker exec caddy caddy reload --config /etc/caddy/Caddyfile`.

A brand-new hostname can return Cloudflare `525` for a few seconds until Caddy has its certificate; retry after about 20 seconds.

## Part 2: Cloudflare Access on the Studio hostname
This is an outline; the exact screens depend on the Cloudflare account and were not checked. The Zero Trust free plan may ask for a payment method; read what the page asks before confirming.
1. Cloudflare dashboard → Zero Trust (set up a team name if it is the first time).
2. Access → Applications → add a **self-hosted** application for `studio.xidolabs.cc`.
3. Add a policy: allow only the email addresses that should reach Studio, with a one-time PIN or another identity method.
4. Test in a private browser window: `https://studio.xidolabs.cc` should show Cloudflare's login first, then Envoy's Basic-auth prompt.
5. Keep `DASHBOARD_PASSWORD` long and random: Access is an extra gate, not a replacement.

Keep in mind: `curl` from the server to `studio.` will now be blocked by Access (expected); the API hostname is not behind Access, so apps keep working.

## Check it worked
- `trill-api` root returns 404 and `/auth/v1/health` with the anon key returns 200.
- `studio.xidolabs.cc` asks for the logins (Cloudflare Access, then Envoy), and Studio pages load, including the Table Editor and SQL editor.
- Nothing else changed: `n8n`, `trill`, `builder` return 200.

## Troubleshooting
- A Studio page misbehaves after the split → note which page, then find the path it calls in `caddy/logs/access.log` or the browser's network tab; if it is an API path missing from the `@api` list, add it, otherwise it belongs on the Studio host.
- Reload changes nothing → the container sees a stale single-file mount: `docker exec caddy cat /etc/caddy/Caddyfile` shows what it really reads; recreate Caddy with `docker compose up -d --force-recreate` as a last resort (all sites down for a few seconds).
- `duplicate site block` → the old `trill-api` block is still in the Caddyfile.

Related: [[Add a project to the server]], [[Project template scripts]]
