---
type: runbook
project: trillequipment
tags: [obsidian, git]
updated: 2026-10-08
---

# Back up these notes

This vault is its own **private** repo: https://github.com/joseamayaa/trillequipment-notes
It is *not* part of the team repo, and must never be: the site deploys the whole team repo publicly.

## Save a backup (end of each session)
In PowerShell:
```powershell
cd "C:\XidoProjects\trillequipment\TrillEquipmentNotes"
git add -A
git commit -m "Notes: short description"
git push
```
Or ask Claude Code: "commit and push my notes".

## Get a deleted note back
```powershell
git log --oneline -- "path/to/Note.md"          # find when it existed
git restore --source=<commit> -- "path/to/Note.md"
```

## Good to know
- `.obsidian/workspace.json` (window layout) is not backed up on purpose; it changes every time Obsidian opens.
- The real project (site + Supabase) lives in WSL at `/home/joh/GitHub/trillequipment`. This vault is a separate repo on the Windows side; the older Windows copy of the project is no longer used.
- Never put tokens or passwords in notes.

Related: [[Git branch workflow]]
