# TASKS.md

Ordered implementation checklist. Work top-to-bottom. Each phase ends in a
demoable state. Mark items `[x]` as completed.

---

## Phase 0 — Pre-build setup (one-time, manual)

- [ ] Create a Microsoft Entra app registration in the work tenant
  - Account type: **Accounts in this organizational directory only**
  - Redirect URIs (Web): `https://<production-domain>/api/auth/callback`
    and `http://localhost:3000/api/auth/callback`
  - Generate a client secret; save to a password manager
  - Grant delegated permissions: `Mail.Read`, `Mail.Send`,
    `MailboxSettings.Read`, `offline_access`, `User.Read` — admin consent if
    required by tenant policy
- [ ] Create a Supabase project; copy `URL`, `anon key`, `service_role key`
- [ ] Create an Anthropic API key; copy it
- [ ] Generate `TOKEN_ENCRYPTION_KEY`: `openssl rand -hex 32`
- [ ] Generate `CRON_SECRET`: `openssl rand -hex 32`
- [ ] Buy / point a domain at Vercel for the production deploy

---

## Phase 1 — Repo + Vercel + GitHub init

- [ ] `pnpm create next-app@latest` with TypeScript, Tailwind, App Router,
      ESLint, `src/` layout off
- [ ] Add `pnpm-workspace.yaml` only if a separate package is needed (likely no)
- [ ] Configure `tsconfig.json` with `strict`, `noUncheckedIndexedAccess`,
      `noImplicitOverride`
- [ ] Install deps: `@azure/msal-node`, `@supabase/supabase-js`,
      `@anthropic-ai/sdk`, `zod`, `lucide-react`, `clsx`,
      `class-variance-authority`, shadcn/ui init
- [ ] Install dev deps: `vitest`, `@playwright/test`, `eslint-config-next`,
      `@types/node`
- [ ] Create `vercel.json` with the two cron entries (sync + snooze-wake)
- [ ] Create `.env.example` listing every var from `CLAUDE.md` (no values)
- [ ] Create `.github/workflows/ci.yml` running typecheck + lint + vitest on PR
- [ ] Push to GitHub; connect the repo to a new Vercel project; add all env
      vars in the Vercel dashboard for **Production** environment only
- [ ] Verify `main` deploys to the production domain

---

## Phase 2 — Supabase schema + token encryption

- [ ] `supabase init` locally; commit `supabase/config.toml`
- [ ] First migration `supabase/migrations/<ts>_init.sql`:
  - Enable `pgcrypto`
  - Create enums `email_category`, `task_state`
  - Create tables `auth_tokens`, `emails`, `tasks`, `sync_state`, `sync_runs`
  - Create indexes per `SPEC.md` §7
  - Enable RLS on all tables; add policies described in `SPEC.md` §7
- [ ] Apply migration to the Supabase project
- [ ] Implement `lib/db/supabase.ts` (server-only client; throw if imported
      from client bundle)
- [ ] Implement `lib/db/crypto.ts` (AES-256-GCM with `TOKEN_ENCRYPTION_KEY`)
- [ ] Unit-test the crypto roundtrip in `lib/db/crypto.test.ts`
- [ ] Implement repositories: `tokens.ts`, `emails.ts`, `tasks.ts`,
      `syncState.ts`, `syncRuns.ts`

---

## Phase 3 — Microsoft OAuth flow

- [ ] Implement `lib/graph/client.ts` using MSAL ConfidentialClientApplication
- [ ] Implement `/api/auth/login` — build auth code URL, redirect
- [ ] Implement `/api/auth/callback`:
  - Exchange code → tokens
  - Call `/me` to get UPN; assert `=== OWNER_EMAIL` (case-insensitive); else 403
  - Encrypt + persist tokens in `auth_tokens`
  - Set signed session cookie
- [ ] Implement `/api/auth/logout` — clear cookie only
- [ ] Implement `lib/auth/session.ts` — sign / verify cookie with HMAC of
      a server secret
- [ ] Implement token-refresh wrapper: `lib/graph/withFreshToken.ts`
- [ ] Smoke-test sign-in locally end-to-end

---

## Phase 4 — Sync + triage engine

- [ ] Implement `lib/graph/messages.ts` — delta query for `inbox` and
      `sentitems`, returns `{messages, deltaLink}`
- [ ] Implement `lib/triage/domainRules.ts` — fast-path for newsletters and
      `noreply@` senders
- [ ] Implement `lib/triage/classifier.ts`:
  - Anthropic SDK call with `claude-sonnet-4-6`
  - System block carries the rubric + 6–10 few-shot examples,
    marked `cache_control: { type: 'ephemeral' }`
  - User block carries `{subject, from, snippet, threadContext}`
  - Output validated with `zod` against the classification schema
- [ ] Implement `lib/triage/taskWriter.ts` — derives tasks from classified
      emails per the rules in `SPEC.md` §4
- [ ] Implement `/api/sync`:
  - Verify `Authorization: Bearer ${CRON_SECRET}`; 401 otherwise
  - Insert `sync_runs` row with `started_at`
  - Run inbox delta → upsert → classify → taskWriter
  - Run sentitems delta → upsert → auto-resolve open tasks for those threads
  - Persist new delta links
  - Update `sync_runs` with `finished_at`, `messages_seen`, `classified`,
    `errors`
- [ ] Implement `/api/snooze-wake` — set `state='open'` on tasks where
      `state='snoozed' AND snoozed_until <= now()`
- [ ] Add `pnpm sync:once` script for local manual triggering

---

## Phase 5 — To-do list + reply

- [ ] Implement `/api/tasks/[id]` — PATCH with `{state, snoozed_until?}`
- [ ] Implement `/api/reply` — POST `{taskId, body}`:
  - Send via Graph `POST /me/messages/{id}/reply`
  - Mark task `completed`, `auto_resolved=false` (it'll be re-confirmed by
    the next sentitems sync)
- [ ] UI: To-do list component, sorted by `due_at` then `created_at`
- [ ] UI: Snooze submenu (today / tomorrow / next week / custom)
- [ ] UI: Manual complete + dismiss

---

## Phase 6 — UI polish

- [ ] Build the two-pane layout (collapsible lanes left, viewer right)
- [ ] Wire Supabase realtime subscriptions on `emails` and `tasks`
- [ ] Implement `lib/keymap.ts` and the help overlay
- [ ] Skeleton loading states for the lanes
- [ ] Sync status indicator in the top bar (reads latest `sync_runs` row)
- [ ] Re-auth prompt when session is missing or token refresh failed
- [ ] Mobile-responsive (single-pane stack)
- [ ] Run Lighthouse; resolve any score below 90 on Performance / A11y

---

## Phase 7 — Enhancements (gated; pick from `SPEC.md` §2.2)

- [ ] **E1 Smart snooze** — already wired in §5; add UX affordances
- [ ] **E2 Daily digest** — new cron at 07:00 local; sends a Graph email
      summarizing yesterday + today's open tasks
- [ ] **E3 Reply drafts** — "Draft reply" button calls Anthropic with thread
      context, opens composer pre-filled
- [ ] **E4 Follow-up nudges** — `/api/sync` flags tasks in
      `Waiting on Response` older than 3 days into a "Nudge" virtual lane
- [ ] **E5 Calendar conflict detection** — request `Calendar.Read` scope,
      detect overlap on meeting requests; flag in the message viewer

---

## Done criteria for v1

- OAuth sign-in works end-to-end on production domain
- Sync runs automatically every 15 minutes; `sync_runs` table shows green
- All six categories appear in the UI; classification confidence visible
- To-do items auto-resolve when a reply is sent through the app
- Manual complete / snooze / dismiss work via keyboard
- Mobile layout is usable
- README documents environment setup
