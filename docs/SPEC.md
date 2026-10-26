# SPEC — Subscription Manager

Status: **v1 approved 2026-10-26.**

## 1. Goal (required)
One owner, on an iPhone (and desktop), keeps every subscription and trial in one structured place and decides what to keep or cancel.

- Details come mostly from the owner's **emails** (Gmail, Yahoo, iCloud, Outlook), later from **transactions** (Apple Card statements first).
- AI extracts the details; **the owner confirms before anything is saved.**
- Trials are watched: a push notification arrives before a trial converts into a charge, with the cancel steps.
- AI gives keep-or-cancel advice and answers questions about the data.

v1 is done when: the owner can connect all four mailboxes (read-only), review AI findings in an inbox, see totals and upcoming renewals/trials, get trial push alerts on the iPhone, get keep-or-cancel advice, chat with the data, and import Apple Card statements.

## 2. Data (required)
Source: a **new dedicated Supabase project** (Postgres + Auth). It does not exist yet, so there is no external schema to copy. The names below are the contract; migrations must match them exactly. All tables have `id uuid pk`, `user_id uuid` (row-level security: `user_id = auth.uid()`), `created_at`, `updated_at`.

### 2.1 `subscriptions`
| Field | Type | Options / notes | App reads | App writes |
|---|---|---|---|---|
| name | text | service name | yes | yes |
| status | enum | `active`, `trial`, `cancelled`, `paused` | yes | yes |
| price_amount | numeric(12,2) | price per cycle, in `currency` | yes | yes |
| currency | text | ISO 4217 code | yes | yes |
| billing_cycle | enum | `weekly`, `monthly`, `quarterly`, `yearly` | yes | yes |
| next_renewal_date | date | | yes | yes |
| trial_start | date | null if no trial | yes | yes |
| trial_end | date | | yes | yes |
| price_after_trial | numeric(12,2) | | yes | yes |
| just_trying | boolean | drives cancel-before-charge alerts | yes | yes |
| cancel_url | text | | yes | yes |
| cancel_steps | text | | yes | yes |
| notice_period_days | int | | yes | yes |
| refund_min_term_notes | text | | yes | yes |
| payment_method | text | label + last 4 digits only, never a full number | yes | yes |
| billed_to_account | text | e.g. Apple ID, PayPal | yes | yes |
| category | text | | yes | yes |
| usage_rating | int | 1–5, owner's own | yes | yes |
| notes | text | | yes | yes |
| source | enum | `manual`, `email`, `statement` | yes | yes |

### 2.2 `findings` (the review inbox)
AI output waiting for the owner. `kind` (`new` / `update`), `proposed` (jsonb of `subscriptions` fields), `matched_subscription_id` (nullable), `confidence` (0–1), `source_ref` (mailbox + message id, **no body stored**), `evidence_snippet` (redacted text), `state` (`pending`, `accepted`, `rejected`). Nothing reaches `subscriptions` until `accepted`.

### 2.3 `mail_connections`
`provider` (`gmail`, `outlook`, `yahoo`, `icloud`), `account_label`, `auth_type` (`oauth`, `app_password`), `secret_encrypted` (AES-GCM, key in env), `last_scan_at`, `status`.

### 2.4 `push_subscriptions`
Web Push endpoint + keys per device.

### 2.5 `statement_imports`
`filename`, `period`, `row_count`, `state`. Parsed rows become `findings`, never direct subscriptions.

### 2.6 `fx_rates`
`base` (EUR), `quote`, `rate`, `as_of`. Source decided in Step 2 (see §6 Q1).

### 2.7 `chat_messages`
`role` (`user`/`assistant`), `content`.

### 2.8 External systems (names used exactly)
- **Gmail API**, scope `gmail.readonly` only.
- **Microsoft Graph**, scope `Mail.Read` only.
- **IMAP** for `imap.mail.yahoo.com` and `imap.mail.me.com`, app passwords, read-only (`EXAMINE`, never `SELECT`+write).
- **Groq** (primary) and **OpenRouter** (fallback) chat-completion APIs, behind one provider interface, called with `fetch` (no SDK).
- **Web Push** (VAPID) for iPhone PWA notifications (iOS 16.4+, installed to Home Screen).

Mailbox rule: the app **never** writes, flags, moves, deletes or sends mail.

## 3. Features
1. **Subscription list and detail:** add/edit/cancel, all §2.1 fields, filters by status/category/renewal.
2. **Dashboard:** monthly and yearly total in EUR, next 30 days of renewals, active trials with days left, "just trying" items highlighted.
3. **Review inbox:** accept / edit-then-accept / reject each finding; updates show a field-by-field diff against the existing item.
4. **Mail scan (on demand):** connect, press "Scan"; a local filter (sender/subject/keyword, date window) picks candidates; text is **redacted** (names, addresses, card numbers) and trimmed to snippets before any AI call.
5. **Trial watch:** daily job checks trials and renewals; push N days before (default 3, editable) with the cancel steps.
6. **Keep-or-cancel advice:** per subscription and a whole-portfolio view (cost, overlap, usage rating, trial status).
7. **Chat:** questions answered from the owner's own subscription rows only.
8. **Apple Card statement import:** CSV and PDF upload, recurring-charge detection, results go to the review inbox.
9. **Look:** a separate chat defines it. Until then use neutral CSS variables in a single `theme` file so the look can be dropped in. No design system is built here.

## 4. Edge cases
- Duplicates: match on normalised name + amount + cycle; matches become `update` findings.
- Free-tier AI limits: rate-limit, retry with backoff, then fall back to the other provider; surface "AI busy" in the UI, never fake output.
- Unparseable email or low confidence (<0.5): shown in the inbox flagged, never auto-accepted.
- Multi-currency: each item keeps its own currency; totals converted to EUR with the stored rate and shown with its date.
- Tokens expiring (Google "testing" mode refresh tokens last 7 days): connection shows "reconnect".
- iOS push only works after Add to Home Screen; the app explains this on first run.
- Single user: signups disabled except the owner's email.

## 5. Build order (required)
Mode: Autonomous. Each step is its own branch and draft PR, passes gate 1 before the next starts, and gets a hand-off.

1. **Step 1 — Foundation:** Next.js + TS strict, Supabase project, magic-link login (allow-listed), RLS, `.env.example`, Vercel preview. Check: log in on the iPhone; a second email is refused.
2. **Step 2 — Subscriptions and dashboard (manual):** §2.1 schema, CRUD, pure cost/renewal logic with tests, EUR totals, FX source. Check: add 3 real subscriptions; totals and dates are right.
3. **Step 3 — AI layer and review inbox:** provider interface (Groq → OpenRouter), redaction, paste-text extraction, inbox. Check: paste a real receipt; finding appears; accept saves it.
4. **Step 4 — Gmail:** read-only OAuth, scan on demand, candidate filter. **Gate 2: run against the real mailbox.**
5. **Step 5 — Outlook:** Microsoft Graph, same flow.
6. **Step 6 — Yahoo and iCloud (IMAP):** app-password connections, same flow.
7. **Step 7 — Trial watch and push:** daily job, Web Push, PWA install. Check: a test trial triggers a push on the iPhone.
8. **Step 8 — Keep-or-cancel advice.**
9. **Step 9 — Chat with your subscriptions.**
10. **Step 10 — Apple Card import:** CSV and PDF, recurring detection.
11. **Release:** gates 3, 4, 5 (code review, final QA, visual check in light/dark × phone/desktop). Production merge needs the owner's OK.

Later (not v1): personal finance tracker transactions; iPhone Subscriptions screenshot import (needs a vision-capable model; most free tiers lack one).

## 6. Open questions (answered as they arise, recorded in §7)
1. FX rate source (the owner approves any new dependency or API first).
2. PDF parsing library for Apple Card (needs approval).

## 7. Decision log (required)
| Date | Decision | Decided by | § |
|---|---|---|---|
| 2026-10-26 | Web app, phone-first, Next.js on Vercel, installable PWA | Owner | 1, 5 |
| 2026-10-26 | Supabase (new dedicated project), single user, email magic link | Owner | 2 |
| 2026-10-26 | Approval mode: Autonomous (assistant advised Strict for real personal data; owner chose Autonomous) | Owner | all |
| 2026-10-26 | Email-first intake: Gmail, Yahoo, iCloud, Outlook all in v1; read-only, on-demand scan | Owner | 2.8, 5 |
| 2026-10-26 | Transactions: Apple Card CSV/PDF first; finance tracker deferred | Owner | 5 |
| 2026-10-26 | Alerts via iPhone PWA push | Owner | 3, 4 |
| 2026-10-26 | AI: free Groq + OpenRouter; filter locally, redact, send snippets only | Owner | 3, 4 |
| 2026-10-26 | EUR home currency with conversion; light+dark, phone+desktop; English UI | Owner | 2, 4 |
| 2026-10-26 | Every AI finding goes to the review inbox before saving | Owner | 2.2 |
| 2026-10-26 | Visual look defined in a separate chat; no design system here | Owner | 3.9 |
| 2026-10-26 | Spec v1 approved | Owner | all |
