---
type: runbook
project: trillequipment
tags: [github, tooling]
updated: 2026-10-08
---

# GitHub CLI setup

`gh` lets me (and Claude Code) open PRs and check their status from the terminal. Hub: [[Trill Equipment]]

## Install (done 2026-10-07, v2.102.0)
```powershell
winget install --id GitHub.cli -e
```
Restart the terminal / Claude Code afterwards so `gh` is found.

## Sign in (done 2026/10/81. Y)
```
gh auth login
```
Choose **GitHub.com → HTTPS → Login with a web browser**.
Check: `gh auth status`

## Everyday PR commands
| Command | What it does |
|---|---|
| `gh pr create --base main` | Open a PR from the current branch into `main` |
| `gh pr status` | My PRs and their review/check state |
| `gh pr checks` | CI checks on the current branch's PR |
| `gh pr view --web` | Open the PR in the browser |
| `gh pr list` | All open PRs on the repo |

Related: [[Git branch workflow]]
