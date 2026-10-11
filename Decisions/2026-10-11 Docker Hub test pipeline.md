---
type: decision
project: trillequipment
status: accepted
tags: [docker, hetzner, deploy]
date: 2026-10-11
---

# Docker Hub test pipeline

Hub: [[Trill Equipment]] · How to: [[Build and push a test image]], [[Update the Hetzner test server]]

## Context
My partner's plan: use Docker Hub for local development and testing, a Hetzner server for team testing, and GitHub only for the final code. He doesn't want anything pushed to GitHub until it's final.

## Decision
- One Docker Hub repo, `jjohamaya/trillequipment`, holding **frontend-only** images (nginx serving the static site).
- Numbered tags (`test-1`, `test-2`, …) for work in progress, not `latest`.
- The Hetzner server pulls a tag and runs it with Docker Compose; updating is a manual pull.
- GitHub gets the code only when it's final.
- The `supabase/` folder stays in the project, in git, and **out of the image**.

## Why
- The site is static HTML; the backend is Supabase (hosted or local CLI), which doesn't need my own image. There's no custom backend code, so no second repo for a backend image.
- `supabase/` is migrations and seed data, not a running service. Putting it in the image would just expose the schema.
- Version tags make rollbacks easy and show what the team is looking at.
- An image holds the actual HTML, so anyone who can pull it can read the code. The privacy comes from the Docker Hub repo being private, plus no secrets in the HTML.

## Alternatives considered
- Separate frontend and backend Docker Hub repos (suggested to me). Only makes sense if I add custom backend code, e.g. an API server or containerized functions.
- Push to GitHub early and build images in CI. Rejected for now: partner wants nothing on GitHub until final.
- Azure Static Web Apps (the repo has `staticwebapp.config.json`): still possible for production, not part of this test setup.

## Open questions
- Make the Docker Hub repo private; who gets access (collaborators vs. a server access token)?
- Separate hosted Supabase project for testing so testers can't touch real data.
- Domain name and HTTPS (Caddy) for the test server.
- Where the code is backed up until GitHub: the project's local git only lives on my machine.
