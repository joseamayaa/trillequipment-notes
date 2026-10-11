---
type: runbook
project: trillequipment
tags: [docker, docker-hub]
updated: 2026-10-11
---

# Build and push a test image

Turn the current site files into a Docker image and publish it to Docker Hub as a numbered test version. Hub: [[Trill Equipment]]

## Before you start
- Docker is installed and I'm logged in (`docker login`).
- Work from the project folder in WSL: `/home/joh/GitHub/trillequipment`
- The `Dockerfile` copies the whole folder into `nginx:alpine`. The `.dockerignore` keeps `supabase/`, `.git`, `.env`, `docs/` and the Docker files out of the image.
- Pick the next tag: `test-1` is done, so the next is `test-2`.

## Steps
1. Build with the full name and tag:
   ```
   docker build -t jjohamaya/trillequipment:test-2 .
   ```
2. (Optional) try it locally first:
   ```
   docker run --rm -p 8080:80 jjohamaya/trillequipment:test-2
   ```
   Open http://localhost:8080
3. Push:
   ```
   docker push jjohamaya/trillequipment:test-2
   ```
4. Then follow [[Update the Hetzner test server]].

If I already have a local image and just need another name for it: `docker tag trillequipment:latest jjohamaya/trillequipment:test-2`. `docker tag` only adds a name; it doesn't copy anything. The `jjohamaya/` prefix is what tells `docker push` which Docker Hub account to use.

## Check it worked
- `docker images jjohamaya/trillequipment` shows the tag locally.
- The tag appears in the repo's Tags tab on hub.docker.com.
- Check what's inside: `docker run --rm jjohamaya/trillequipment:test-2 ls /usr/share/nginx/html` (should be HTML, `images`, `assets`; no `supabase`, `.git`, `.env`).

## Troubleshooting
- Server says `not found` for a tag → that tag was never pushed. Push it, then retry.
- `denied` / `access denied` on pull → repo is private and the server isn't logged in.

## Good to know
- Use version tags, not `latest`, so I know what's running. Old tags stay on Docker Hub, which makes rollbacks easy.
- Anyone who can pull the image can read the site files. The site is plain HTML, so the privacy of the image comes from the repo being private.
- Never put secrets in the HTML or the image.

Related: [[2026-10-11 Docker Hub test pipeline]]
