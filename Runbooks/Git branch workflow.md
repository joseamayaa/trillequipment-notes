---
type: runbook
project: trillequipment
tags: [git]
updated: 2026-10-08
---

# Git branch workflow

Hub: [[Trill Equipment]]

## Branches
- `main`: the shared branch; changes go in through PRs
- `Joh's-Branch`: mine (capital **B**)
- `Jason's-Branch`: teammate's

## Check which branch I'm on
```
git branch --show-current
git status -sb
```
`git status -sb` also shows if I'm ahead/behind the remote.

## Switch to my branch
```
git switch "Joh's-Branch"
```
Quotes are needed because of the apostrophe.

## Save work
```
git add <files>
git commit -m "Short description of the change"
git push
```

## Get it into main
Open a PR → see [[GitHub CLI setup]].
