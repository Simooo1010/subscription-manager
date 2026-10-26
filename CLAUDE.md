# CLAUDE.md — working rules for this repo

## What this is
Subscription Manager: a phone-first responsive web app (installable PWA) that keeps all of the owner's subscriptions and trials in one place, fed mainly by read-only email scans and statement imports, with AI extraction, trial alerts and keep-or-cancel advice. Data lives in Supabase.
- Product spec: `docs/SPEC.md`
- Working method: `docs/WORKFLOW.md` — **read it before any work, every session.**
- Owner-only setup steps: `SETUP-FOR-OWNER.md`

## Project settings
- **Owner:** Simone (GitHub `simooo1010`)
- **Approval mode:** Autonomous (definitions in `docs/WORKFLOW.md` §3). Assistant advised Strict because the app handles real personal data; owner chose Autonomous on 2026-10-26.
- **Milestones** (Milestone mode only): n/a
- **Production merge:** needs owner OK.
- **Test device(s):** iPhone, Safari (installed to Home Screen for push)
- **Supported:** themes: light + dark · sizes: phone + desktop
- **Design system:** none. The look is defined in a separate chat; use neutral CSS variables in one theme file so it can be dropped in. Do not build a design system.

## Stack
- Next.js (App Router) + TypeScript strict, deployed on Vercel (preview per branch, production = `main`).
- Supabase (Postgres + Auth, RLS on every table). Credentials used on the server only.
- AI: Groq (primary) and OpenRouter (fallback) via `fetch` behind one provider interface.
- **Approved dependencies:** Next.js, React, TypeScript, Vitest + ESLint (dev tooling). Everything else is **pending until the owner approves the spec**: `@supabase/supabase-js`, `@supabase/ssr`, `zod`, `web-push`, `imapflow`, `mailparser`, plain `fetch` for Gmail/Graph. Any other (PDF parser, FX source, email sender) needs a separate ask. Record approvals below.
  - Approvals: _none yet_

## Folder layout
```
docs/                 # SPEC.md, WORKFLOW.md, HOW-TO-WORK-WITH-CLAUDE.md, SPEC.template.md
SETUP-FOR-OWNER.md    # owner-only steps
CLAUDE.md             # this file
(app code is added in Step 1; update this tree in the same commit)
```

## Commands
- `npm run lint` — lint
- `npm run build` — production build (must pass before every push)
- `npm test` — unit tests (Vitest)
- `npm run dev` — quick checks inside the sandbox only

## Where this runs
- Development in Claude Code on the web (cloud sandbox). No local machine.
- Secrets: sandbox environment variables + Vercel environment variables. Never in repo files.
- Owner tests on **preview URLs** on the test device.

## Hard rules
- Never commit secrets. Only `.env.example` is committed. Never print secret values.
- No secret reaches the client (no `NEXT_PUBLIC_` on secrets).
- Use the exact external names in `docs/SPEC.md` §Data. If something is missing or named differently: **stop and report**. Never guess or rename.
- The app writes only: its own Supabase tables listed in SPEC §2. Mailboxes are strictly read-only: never write, flag, move, delete or send mail.
- Mailbox tokens and app passwords are stored encrypted server-side; email bodies are not stored.
- Only redacted snippets go to AI providers (SPEC §3.4).
- AI output never changes the owner's data without an explicit owner confirmation: findings go through the review inbox, assistant actions through a confirm card (SPEC §3.7). The AI holds no database credentials and has only the closed action list.
- Always paginate external queries.
- No fake data, except fixtures inside tests.
- Ask before adding any dependency not listed above; record approvals here.
- Run lint, build and tests before saying a step is done. Show the output.
- Never start the next step or merge beyond what the approval mode allows.
- The playbook repo (`claude-project-playbook`) is read-only: never edit it.
