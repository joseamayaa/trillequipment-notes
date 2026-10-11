---
type: log
project: trillequipment
tags: [docker, hetzner]
date: 2026-10-11
---

# 2026-10-11 Docker Hub and Hetzner setup

Hub: [[Trill Equipment]] · Worked with Claude Code

## Done
- Added `Dockerfile` (`nginx:alpine`, copies the site) and `.dockerignore` (excludes `.git`, `.github`, `.env`, `supabase`, `docs`, README, Docker files, `staticwebapp.config.json`) to the project in WSL.
- Built the image (about 96 MB) and pushed it to Docker Hub as `jjohamaya/trillequipment:latest`.
- Learned what `docker tag` does: adds a name to an existing image; the `username/` prefix tells `docker push` where to send it. Two local names, one image ID, one image on Docker Hub.
- Decided to keep `supabase/` out of the image and keep one frontend repo → [[2026-10-11 Docker Hub test pipeline]]
- Wrote a `docker-compose.yml` for the Hetzner server (`~/trillequipment`). The first `docker compose up -d` failed with `not found` for `test-1`: the tag had never been pushed (only `latest`). Tagged and pushed `test-1`; the server then pulled it and the site is running.
- Learned how to test the server and how to ship `test-2`, and how to roll back → [[Update the Hetzner test server]], [[Build and push a test image]]
- Reviewed this vault: fixed paths to the WSL project folder, added this hub, runbooks and decision, and added `trillpw.txt` to `.gitignore`.

## Commits / PRs
- The project's `Dockerfile` and `.dockerignore` are **not committed** yet.

## Next
- [ ] **Deal with `trillpw.txt`** in this vault: move it to a password manager and delete it; if it was ever pushed, change that password
- [ ] Set the Docker Hub repo to private; decide who gets access
- [ ] Buy a domain, add an A record, put Caddy in front for HTTPS
- [ ] Point test images at a test Supabase project
- [ ] Commit the Docker files in the project; keep a master `docker-compose.yml` in the project folder

## Notes
- The older Windows copy of the project is no longer used; the project lives in WSL.
- Before each `git add -A` on the vault, run `git status` and read the list.
