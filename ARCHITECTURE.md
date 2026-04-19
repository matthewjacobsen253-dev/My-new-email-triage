# ARCHITECTURE.md

System architecture for **My Email Triage**.

## 1. High-level diagram

```
                 ┌────────────────────────────────────────────────┐
                 │                   Vercel                       │
                 │                                                │
   Browser ──►   │  ┌──────────────────┐    ┌─────────────────┐   │
   (Next.js      │  │  Next.js App     │    │  Vercel Cron    │   │
    client       │  │  - RSC pages     │    │  */15 * * * *   │   │
    bundle)      │  │  - /api/auth/*   │    │                 │   │
                 │  │  - /api/reply    │    │  Hourly:        │   │
                 │  │  - /api/sync ◄───┼────┤  /api/snooze-wk │   │
                 │  │  - /api/snooze-* │    └─────────────────┘   │
                 │  └────┬────────┬────┘                          │
                 └───────┼────────┼──────────────────────────────-┘
                         │        │
              MS Graph   │        │  service-role
              REST + OAuth        │  Postgres
                         ▼        ▼
              ┌──────────────┐   ┌────────────────────────────┐
              │   Microsoft  │   │          Supabase          │
              │   Graph API  │   │  ┌──────────────────────┐  │
              │  graph.ms.com│   │  │ Postgres             │  │
              │              │   │  │  - auth_tokens       │  │
              │  delegated:  │   │  │  - emails            │  │
              │   Mail.Read  │   │  │  - tasks             │  │
              │   Mail.Send  │   │  │  - sync_state        │  │
              │   ...        │   │  │  - sync_runs         │  │
              └──────────────┘   │  └──────────────────────┘  │
                                 │  Realtime channel ──┐      │
                                 └─────────────────────┼──────┘
                                                       │
                                  websocket ◄──────────┘
                                  to browser (anon key, RLS)

              ┌──────────────┐
              │  Anthropic   │   server-side only;
              │  Claude API  │   called from /api/sync
              │  classify()  │   prompt cache on system block
              └──────────────┘

              ┌──────────────┐
              │   GitHub     │   repo + Actions for typecheck/lint/test
              │              │   Vercel GitHub integration handles deploys
              └──────────────┘
```

## 2. Component responsibilities

### Browser (Next.js client)
- Render RSC-streamed lanes and to-do list
- Open a Supabase realtime websocket using the anon key, subscribed to
  `emails` and `tasks` tables
- Send user actions (reply, snooze, complete) to `/api/*` route handlers
- Own keyboard shortcut handling (`lib/keymap.ts`)

### Next.js Route Handlers (Vercel Functions)
- `/api/auth/login` — build MSAL auth code URL, redirect to Microsoft
- `/api/auth/callback` — exchange code, enforce `OWNER_EMAIL`, persist
  encrypted tokens, set session cookie
- `/api/auth/logout` — clear cookie; tokens persist for sync to keep working
- `/api/sync` — cron-triggered; performs the full triage cycle
- `/api/snooze-wake` — hourly cron; flips snoozed tasks back to open
- `/api/reply` — sends a reply via Graph; pre-marks the related task complete
- `/api/tasks/[id]` — manual state changes (complete, dismiss, snooze)

### Triage engine (`lib/triage/`)
- `delta.ts` — wraps Graph delta queries, persists `@odata.deltaLink`
- `classifier.ts` — Anthropic call with cached system prompt; deterministic
  domain-rule fast path before LLM
- `taskWriter.ts` — turns classified emails into tasks; handles auto-resolve

### Microsoft Graph layer (`lib/graph/`)
- `client.ts` — token acquisition and refresh, single chokepoint for `fetch`
- `messages.ts`, `send.ts`, `user.ts` — typed wrappers per endpoint

### Database layer (`lib/db/`)
- `supabase.ts` — server-only client with service role key
- `tokens.ts`, `emails.ts`, `tasks.ts`, `syncState.ts` — typed repositories
- `crypto.ts` — AES-256-GCM helpers using `TOKEN_ENCRYPTION_KEY`

### Vercel Cron
- Configured in `vercel.json`; sends `Authorization: Bearer ${CRON_SECRET}`
- 15-min schedule for `/api/sync`, hourly for `/api/snooze-wake`

### Supabase
- Postgres for all persistence
- Realtime channel for live UI updates
- `pgcrypto` extension enabled; never used for token encryption (we encrypt
  in the app to keep keys out of the DB), only for `gen_random_uuid()`

### Anthropic
- `claude-sonnet-4-6` for classification
- System prompt (rubric + few-shot) marked `cache_control: ephemeral` to hit
  the prompt cache on every sync run
- Optional: same model for reply-draft generation (E3)

### GitHub
- Source of truth for the repo
- Vercel GitHub integration auto-deploys `main` to production and PRs to
  preview URLs (preview URLs do **not** have working OAuth — see DECISIONS.md)
- Single Actions workflow `.github/workflows/ci.yml` runs typecheck, lint,
  vitest on PRs

## 3. Data flow: a new email being triaged

```
T+0:00   Vercel Cron fires → POST /api/sync (Bearer CRON_SECRET)
T+0:01   /api/sync verifies secret, loads delta_link from sync_state
T+0:02   lib/graph/messages.delta() → Microsoft Graph
         (refreshes access token from auth_tokens if expiring)
T+0:05   N new messages returned + new @odata.deltaLink
T+0:06   Upsert messages into emails table
T+0:07   For each new message:
           - domain rule check (newsletter? noreply?)
           - if no rule match: classify() → Claude API (cache hit on system)
           - update emails row with category + confidence + rationale
T+0:30   For categories needing action: insert into tasks
T+0:31   For each open task: check sentitems for thread reply → auto-resolve
T+0:32   Persist new delta_link to sync_state
T+0:33   Insert sync_runs row (success)
T+0:34   Supabase realtime fires UPDATE/INSERT events
T+0:35   Browser receives websocket event → React state updates → UI rerenders
```

## 4. Deployment topology

- **Production**: a single Vercel project linked to the GitHub repo, deploying
  from `main`. Domain: a stable user-controlled domain (e.g.,
  `triage.example.com`). This domain is the only valid OAuth redirect host.
- **Preview**: every PR gets a Vercel preview URL. These can render the UI
  but **OAuth will fail** on them by design — the preview hostname is not
  registered in Entra. This is intentional; it forces auth testing against
  production or local.
- **Local dev**: `pnpm dev` on `http://localhost:3000` with a separate
  Entra-registered redirect URI `http://localhost:3000/api/auth/callback`.

## 5. Why polling lives where it does

Vercel Functions have a max execution duration (60s on Pro). The triage cycle
finishes well within that for a typical 15-min batch. The cycle is **stateless
between runs** — the only continuity is the delta link in Postgres, so each
cron tick is a fresh, short-lived function call. No long-running daemon, no
extra worker process, no Railway/Render/Fly account.

If sync ever exceeds 60s, the fix is to chunk: cap each run at N pages and let
the next cron tick continue from the same delta link. The architecture supports
this without changes — the delta link is updated only after a fully successful
page set.
