---
type: runbook
project: trillequipment
tags: [claude-code, tooling]
updated: 2026-10-08
---

# Resume a Claude Code session

Sessions save automatically per project folder. Hub: [[Trill Equipment]]

## Resume
From `C:\GitHub Repos\trillequipment`:
| Command | Use when |
|---|---|
| `claude --continue` | Pick up the most recent session |
| `claude --resume` | Choose from a list of past sessions |
| `/resume` | Same list, from inside Claude Code |

## Other handy commands
- `/rename`: name the session (e.g. "Supabase setup") so it's easy to find
- `/export`: save the conversation as text or copy it to the clipboard
- `! <command>`: run a command myself inside the session, output goes into the chat (e.g. `! gh auth login`)

## Habit
Before ending a session, write a short entry in `Logs/` → template [[Session log]].
The code itself is safe once it's committed; see [[Git branch workflow]].
