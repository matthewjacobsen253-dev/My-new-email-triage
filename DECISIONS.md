# DECISIONS.md

Append-only log of technical decisions. Each entry: what we picked, what we
rejected, and why. This file exists to prevent re-litigating settled choices
mid-build.

---

## D1 — Frontend: Next.js 14 (App Router)

**Picked**: Next.js 14 with the App Router and React Server Components.

**Rejected**:
- *React + Vite SPA*: would require a separate API server, doubling deploy
  surface and forcing CORS handling. No SSR for the initial inbox view.
- *HTMX*: wrong fit. The triage UI is keyboard-heavy, stateful, has realtime
  updates, and benefits from an SPA-style component tree.

**Why**: Next.js gives us co-located server actions for mutations, RSC
streaming for the initial paint of the lanes (data is server-fetched from
Supabase), and a single deploy unit on Vercel. The "framework tax" is real
but small for the productivity gain.

---

## D2 — Backend: Next.js Route Handlers on Vercel Functions

**Picked**: Route Handlers under `app/api/*`, deployed as Vercel Functions on
the `nodejs20.x` runtime.

**Rejected**:
- *Standalone Express / FastAPI server*: extra hosting account, extra deploy
  pipeline, no benefit for a single-user app.

**Why**: Co-located with the frontend, share types and validation, single
deploy. The 60s max execution is fine for a sync cycle.

---

## D3 — Polling: Vercel Cron Jobs

**Picked**: Vercel Cron `*/15 * * * *` calling `/api/sync` with a
`CRON_SECRET` bearer token.

**Rejected**:
- *Microsoft Graph webhook subscriptions*: more correct in theory (push, not
  poll). Rejected for v1 because: subscriptions expire every ~3 days and must
  be renewed; require a publicly reachable HTTPS endpoint that can pass
  Microsoft's validation handshake on every (re)subscription; significantly
  harder to test locally; debugging dropped notifications is painful. Revisit
  in v2 if 15-min cadence proves too slow.
- *Separate worker on Railway/Render/Fly*: adds a hosting account and a
  long-running process for no functional gain over cron, since the workload
  is naturally batched.

**Why**: Cron is GA, native to Vercel, free on Hobby for the volumes here,
zero extra infra, and the 15-min cadence matches the requirement exactly.

---

## D4 — Database: Supabase Postgres

**Picked**: Supabase managed Postgres with `pgcrypto` enabled.

**Rejected**:
- *SQLite local-first*: would mean either bundling the DB with the app (no
  multi-device, lost on reinstall) or running a server process that owns the
  file (no longer "local-first" in any meaningful sense). Realtime UI updates
  would need a custom websocket layer.

**Why**: Free tier covers this comfortably; realtime subscriptions give the
UI live updates after each sync without polling from the browser; standard
SQL means migrations are vendor-portable if Supabase becomes a problem;
service role key model fits the "server writes, client reads via RLS" pattern
we want.

---

## D5 — Auth: Direct Microsoft OAuth, not Supabase Auth

**Picked**: Use `@azure/msal-node` directly. Persist encrypted Graph tokens in
`auth_tokens`. Use a small signed session cookie (no token in cookie) to
identify the browser. Single-user gating via `OWNER_EMAIL` env var.

**Rejected**:
- *Supabase Auth with Azure provider*: Supabase's built-in Azure provider can
  handle sign-in, but it doesn't expose the underlying Graph access token in
  a way that's friction-free for ongoing Graph calls (we'd need to
  re-implement token refresh against Microsoft anyway). For a single-tenant
  work account, the Supabase abstraction adds complexity without value.
- *Custom OIDC provider plugged into Supabase Auth*: more wiring than just
  owning the OAuth handshake.

**Why**: We need the Graph tokens regardless. Owning the OAuth flow is ~150
lines of MSAL + token storage code; we save the auth-bridge complexity and
keep one mental model. The single-user design also means we don't need
Supabase Auth's user management.

**Security notes**:
- Tokens are AES-256-GCM encrypted at rest using `TOKEN_ENCRYPTION_KEY`
  (generated once, stored only in Vercel env vars + a sealed offline backup).
- Cookies are HTTP-only, Secure, SameSite=Lax.
- The session cookie carries no tokens — only `{user: OWNER_EMAIL}` signed.

---

## D6 — Hosting: Vercel, production-only OAuth

**Picked**: Vercel for hosting. The Entra app registration lists exactly two
redirect URIs:
1. `https://<production-domain>/api/auth/callback` — production
2. `http://localhost:3000/api/auth/callback` — local dev

**Rejected**:
- *Wildcard or per-PR preview redirect URIs*: Microsoft Entra does not
  support wildcards on redirect URIs for confidential clients. Manually
  registering each preview URL is operationally untenable.
- *Vercel preview alias trick* (a fixed preview hostname pointing to the
  latest preview): possible, but adds config drift and a subtle gotcha where
  preview-vs-production behavior differs only at the auth boundary.

**Why**: Preview deploys are still useful for UI work — they just can't
complete OAuth. This is an acceptable trade. To test auth changes, run
locally or merge to `main`. This is documented in `CLAUDE.md` so future
sessions don't try to "fix" it by adding preview URIs.

---

## D7 — Version control + CI: GitHub + Vercel integration + minimal Actions

**Picked**: GitHub for the repo. Vercel's native GitHub integration handles
build + deploy on push to `main` and preview deploys on PRs — no custom
Actions for that. A single Actions workflow `.github/workflows/ci.yml` runs
`pnpm typecheck`, `pnpm lint`, `pnpm test` on PRs.

**Rejected**:
- *Vercel CLI from Actions for deploys*: duplicates what the Vercel
  integration does for free.
- *No CI at all*: tempting for a personal tool, but the cost of one workflow
  file is trivial and the upside (catch regressions before deploy) is real.

**Why**: Smallest sufficient surface. Vercel deploys are automatic. Actions
covers what Vercel doesn't — running the test suite.

---

## D8 — Classification: Anthropic Claude with prompt caching

**Picked**: `claude-sonnet-4-6` for classification, with the rubric +
few-shot examples in a cached system block. Domain-rule fast path runs first
to avoid LLM calls for obvious newsletters / no-reply automation.

**Rejected**:
- *Local heuristic classifier*: rules don't generalize; the whole point of
  the tool is intelligent triage.
- *Embedding + nearest-neighbor*: needs labeled training data we don't have.
- *Claude Opus*: overkill for a 6-class classification with confidence; cost
  per email matters because every inbound message gets classified.

**Why**: Sonnet 4.6 is the right cost/quality point for this task. Prompt
caching keeps per-email cost low because the rubric (the bulk of input
tokens) is reused across every classification call.

---

## D9 — Single-user design

**Picked**: Hardcode a single `OWNER_EMAIL` allowlist; no user table.

**Rejected**:
- *Multi-user from day one*: requires sessions per user, RLS policies keyed
  on user, multi-tenant Entra registration, and a user management UI. None
  of this is in the requirements.

**Why**: Adding multi-user later is a known refactor (RLS policies + a
`users` table + session keying); building it now would slow v1 by weeks for
zero current value.
