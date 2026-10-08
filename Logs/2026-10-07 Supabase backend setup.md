---
type: log
project: trillequipment
tags: [supabase, github]
date: 2026-10-07
---

# 2026-10-07 Supabase backend setup

Hub: [[Trill Equipment]] · Worked with Claude Code

## Done
- Confirmed I'm on `Joh's-Branch` (tracks `origin/Joh's-Branch`).
- Reviewed trillequipment.com to work out what the backend needs.
- Created the `supabase/` folder: config, 2 migrations, local seed data → [[2026-10-07 Supabase schema & access rules]]
- Started Docker Desktop + local Supabase; tested public access, validation, double-booking protection and staff permissions. All passed; `db lint` clean.
- Committed `376eb96` to `Joh's-Branch` (not pushed).
- Installed GitHub CLI v2.102.0 → [[GitHub CLI setup]]
- Learned how to give a Supabase token safely → [[Supabase token setup]]
- Set up this vault: runbooks, decisions, logs, templates.

## Next
- [ ] Restart Claude Code, run `gh auth login`
- [ ] Push branch, open PR into `main`
- [ ] Create hosted Supabase project, `supabase link` + `db push`
- [ ] Fill in real rates / deposit / phone / address
- [ ] Bring website code into the repo and connect the reserve form

## Notes
- Local Supabase + Docker were left running: `npx supabase stop` to stop.
