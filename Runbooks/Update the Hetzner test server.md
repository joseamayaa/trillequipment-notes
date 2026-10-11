---
type: runbook
project: trillequipment
tags: [docker, hetzner, deploy]
updated: 2026-10-11
---

# Update the Hetzner test server

Make the Hetzner test server run a new image version from Docker Hub. Hub: [[Trill Equipment]]

The server **pulls** images; nothing is pushed to it. Nothing updates on its own after I push a new tag.

## Before you start
- The new tag is pushed: [[Build and push a test image]]
- SSH access to the server (my partner's server, so check with him).
- On the server, the folder `~/trillequipment` holds `docker-compose.yml`. The master copy of that file is in the project folder; the server has its own copy.
- If the Docker Hub repo is private, run `docker login` on the server once (an access token is better than a password).

## First-time setup (done)
```
mkdir -p ~/trillequipment
cd ~/trillequipment
nano docker-compose.yml
docker compose up -d
```
Compose file:
```yaml
services:
  site:
    image: jjohamaya/trillequipment:test-1
    ports:
      - "80:80"
    restart: unless-stopped
```

## Update to a new version
1. In `~/trillequipment`, edit `docker-compose.yml` and change the tag, e.g. `test-1` → `test-2`.
2. ```
   docker compose pull
   docker compose up -d
   ```
   Compose replaces the container; expect a few seconds of downtime.

## Roll back
Change the tag in the file back to the old one and run `docker compose up -d`.

## Check it worked
- `docker ps` shows the right image, status "Up".
- `curl -I http://localhost` on the server returns `200 OK` and `Server: nginx`.
- From my computer, open `http://SERVER_IP` (hard refresh with Ctrl+Shift+R).
- `docker images --digests jjohamaya/trillequipment` digest matches what the push printed.
- If it works on the server but not from outside → firewall (Hetzner cloud firewall, `ufw status`) blocking port 80.

## Troubleshooting
- `failed to resolve reference ... not found` → the tag doesn't exist on Docker Hub (it happened with `test-1` before it was pushed). Push it.
- Container not running → `docker ps -a`, then `docker logs <name>`.

## Not done yet
Domain + HTTPS (Caddy in front of the site container). See the next steps in [[Trill Equipment]].

Related: [[2026-10-11 Docker Hub test pipeline]]
