---
type: runbook
project: trillequipment
tags: [supabase, security]
updated: 2026-10-08
---

# Supabase token setup

Give the Supabase CLI (and Claude Code) access to my Supabase account so it can create and update the hosted project. Hub: [[Trill Equipment]]

> [!warning] Never paste the token into chat, notes or the repo
> The token can control **every** project in the account. Store it only as an environment variable.

## Steps
1. Create the token: https://supabase.com/dashboard/account/tokens → **Generate new token** (name it e.g. `claude-trillequipment`). Copy it immediately; it starts with `sbp_` and is shown once.
2. In a separate PowerShell window (not Claude Code):
   ```powershell
   setx SUPABASE_ACCESS_TOKEN "sbp_your_token_here"
   ```
3. Restart Claude Code / the terminal so it sees the variable.
4. Check it works (doesn't print the token):
   ```
   npx supabase projects list
   ```

**Alternative:** `npx supabase login` opens the browser and saves the login locally.

## Where it lives
- Windows user environment variable `SUPABASE_ACCESS_TOKEN`

## Revoke / rotate
Same page: https://supabase.com/dashboard/account/tokens → delete the token. Good to do once setup is finished.

## Next (not done yet)
Link the repo to the hosted project and push the migrations:
```
npx supabase link --project-ref <project-ref>
npx supabase db push
```
