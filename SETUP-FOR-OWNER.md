# Setup — what only you can do

Claude keeps this file current: a step is added when needed, ticked when done. Plain words, one action per line.

Status: 0 of 14 done. Nothing here is needed until you approve `docs/SPEC.md`.

## Accounts and access
- [ ] Connect GitHub to Claude (Settings → Connectors → GitHub). Done if Claude can open PRs on `simooo1010/subscription-manager`.
- [ ] Create a **new Supabase project** for this app (Claude can do it for you after you confirm any cost).
- [ ] Create a Vercel project from the GitHub repo `simooo1010/subscription-manager`.
- [ ] Create a free Groq account and an API key.
- [ ] Create a free OpenRouter account and an API key.

## Secrets (environment variables)
Never paste secret values into chat. Set them here:
- [ ] Claude cloud environment **and** Vercel (Preview + Production): `SUPABASE_URL`, `SUPABASE_ANON_KEY` — from the Supabase project settings.
- [ ] Same places: `SUPABASE_SERVICE_ROLE_KEY` — server only.
- [ ] Same places: `GROQ_API_KEY`, `OPENROUTER_API_KEY`.
- [ ] Same places: `ALLOWED_EMAIL` — the one email allowed to log in.
- [ ] Same places: `MAIL_SECRET_KEY` — a random 32-byte string; Claude gives you the command to generate it.
- [ ] Same places: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` — for push; Claude gives you the command.

## Connections to outside systems (needed from Step 4)
- [ ] Google Cloud: create an OAuth client (Gmail API, read-only scope) and add yourself as a test user. Claude writes the exact clicks when Step 4 starts.
- [ ] Microsoft Entra: register an app with `Mail.Read` (Step 5).
- [ ] Yahoo: create an **app password** (Account Security) (Step 6).
- [ ] iCloud: create an **app-specific password** at appleid.apple.com (Step 6).

## On your device
- [ ] Open the preview URL in Safari → Share → **Add to Home Screen** (needed for push, Step 7). Allow notifications when asked.
