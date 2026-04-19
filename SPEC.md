# SPEC.md

Product specification for **My Email Triage**, v1.

## 1. Purpose & User Story

> As a knowledge worker drowning in a Microsoft Outlook work inbox, I want a tool
> that quietly classifies new mail every 15 minutes, surfaces only the messages
> that need me, and turns commitments into a living to-do list — so I can spend
> my mornings doing work instead of triaging the queue.

Success looks like: opening the app once in the morning and once after lunch,
clearing the "Need to Respond" lane in under five minutes each, and never
re-reading a newsletter.

## 2. Features

### 2.1 Core (must ship in v1)

- **OAuth sign-in** with the user's Microsoft work account
- **15-minute background sync** of new and updated mail via Graph delta queries
- **Automated triage** of every new message into one of the categories below
- **Triage inbox UI** with one lane per category, keyboard-driven navigation
- **Auto-generated to-do list** derived from emails in actionable categories
- **Auto-resolve on reply** — sending a reply via the app marks the originating
  to-do complete and moves the thread to "Waiting on Response"
- **Manual override** — keyboard shortcut to recategorize, snooze, or complete

### 2.2 Enhancements (proposed, prioritized by ROI vs. complexity)

| #  | Feature                       | Why it matters                                  | Complexity |
| -- | ----------------------------- | ----------------------------------------------- | ---------- |
| E1 | Smart snooze                  | Defer items until tomorrow / next week          | Low        |
| E2 | Daily morning digest email    | Pre-coffee summary; drives daily app open       | Low        |
| E3 | Reply-draft suggestions       | Claude drafts a reply you edit; saves typing    | Medium     |
| E4 | Follow-up nudges              | Resurface "Waiting on Response" after N days    | Medium     |
| E5 | Calendar conflict detection   | Flag meeting requests that overlap existing     | High       |

Build order: E1 → E2 → E3 → E4 → E5. Ship E1 + E2 inside v1 if cheap; gate the
rest behind explicit user approval.

## 3. Triage Categories

Six categories, mutually exclusive, every message lands in exactly one:

| Category              | Definition                                                                | Default action                                  |
| --------------------- | ------------------------------------------------------------------------- | ----------------------------------------------- |
| **Need to Respond**   | Sender explicitly or implicitly expects a reply from the user             | Creates a to-do; sits in primary lane           |
| **Waiting on Response** | User sent something; thread is awaiting the other party                | Creates a tracked item with a 3-day nudge timer |
| **Needs Scheduling**  | Calendar action required (meeting request, time-find, reschedule)         | Creates a to-do tagged `calendar`               |
| **Delegate**          | Should be forwarded to a teammate; user is not the right owner            | To-do prompts forward target                    |
| **FYI / No Action**   | Informational, no reply expected (announcements, status updates)          | Read-and-archive lane; no to-do                 |
| **Newsletter / Low Priority** | Bulk mail, marketing, automated reports                           | Auto-archive after 7 days; rolled into digest   |

### Classification logic

- For every new message, the sync pipeline calls
  `lib/triage/classifier.ts → classify({subject, from, snippet, threadContext})`
- Internally this hits Anthropic Claude with:
  - A cached system prompt containing the rubric above + few-shot examples
  - A user message with the email's subject, sender, snippet (first 500 chars),
    and last reply in the thread (if any)
  - Output schema: `{ category: enum, confidence: 0-1, rationale: string,
    suggested_due_at: timestamptz | null, extracted_action: string | null }`
- Confidence < 0.6 → flagged for manual review (yellow dot in UI)
- Sender domain rules can override (e.g., `noreply@`, `*.mailchimp.com` →
  Newsletter without LLM call) for cost and latency

## 4. To-Do List Model & Lifecycle

### 4.1 Data model

A `tasks` row carries:

- `id` (uuid)
- `email_id` (fk → `emails.id`, nullable for manually created tasks)
- `thread_id` (Graph conversation ID, indexed)
- `title` (extracted action, e.g., "Reply to Jane re: Q3 forecast")
- `category` (enum, mirrors the email's category)
- `state` (`open` | `snoozed` | `completed` | `dismissed`)
- `snoozed_until` (timestamptz, nullable)
- `due_at` (timestamptz, nullable)
- `created_at`, `updated_at`, `completed_at` (timestamptz)
- `auto_resolved` (boolean) — true if completed by a sent reply, false if manual

### 4.2 Lifecycle

1. **Created** — sync pipeline creates a task when category ∈
   {Need to Respond, Waiting on Response, Needs Scheduling, Delegate}
2. **Open** — visible in the to-do list, sortable by due_at then created_at
3. **Snoozed** — `state=snoozed`, `snoozed_until` set; hidden until that time;
   a Vercel Cron job at the top of every hour wakes snoozed tasks
4. **Completed** — either:
   - Auto: a `Sent` event for the same `thread_id` arrives in the next sync
   - Manual: user presses `c` on the task
5. **Dismissed** — user explicitly removes (e.g., decided not to act); kept in
   DB for analytics, hidden from UI

### 4.3 Auto-resolve detection

After each sync, for every task in state `open` with a non-null `email_id`:

```sql
SELECT 1 FROM emails
WHERE thread_id = $1
  AND folder = 'sentitems'
  AND received_at > $task.created_at
LIMIT 1;
```

If a row exists, mark the task `completed` with `auto_resolved=true`. The
originating thread is moved to "Waiting on Response" until the next inbound
reply arrives.

## 5. Microsoft Graph Integration

### 5.1 Required scopes (delegated)

- `Mail.Read` — read inbox + sent items
- `Mail.Send` — send replies from the app
- `MailboxSettings.Read` — pick up the user's timezone for "snooze until tomorrow"
- `offline_access` — refresh tokens
- `User.Read` — basic profile for `OWNER_EMAIL` allowlist check

Application permissions are **not** used — this is a single-user delegated app.

### 5.2 Auth flow

1. User clicks **Sign in with Microsoft** → `/api/auth/login` → MSAL builds
   the auth code URL → 302 redirect to Microsoft
2. Microsoft authenticates → 302 back to `MS_REDIRECT_URI` (= `/api/auth/callback`)
3. Callback exchanges code for tokens, calls `me` to get UPN, asserts
   `upn === OWNER_EMAIL` (case-insensitive). Mismatch → 403, no token stored.
4. `access_token` + `refresh_token` are AES-256-GCM encrypted with
   `TOKEN_ENCRYPTION_KEY` and inserted/updated in `auth_tokens` (single row).
5. A signed session cookie (HTTP-only, Secure, SameSite=Lax, 30-day) is set;
   contains only `{user: OWNER_EMAIL}`. No tokens in cookies.

### 5.3 Token refresh

- Token refresh happens server-side on every Graph call: if `expires_at` is
  within 60 seconds, refresh first, persist, then proceed.
- Refresh failures (invalid_grant) clear the row and force re-login next visit.

### 5.4 Sync mechanics

- Use Graph **delta queries** on `/me/mailFolders/inbox/messages/delta` and
  `/me/mailFolders/sentitems/messages/delta`
- Persist the `@odata.deltaLink` per folder in `sync_state`
- On each cron tick: load delta link, fetch page(s), upsert into `emails`,
  classify newly-arrived items, write tasks
- **Rate limits**: respect `Retry-After` headers; cap a single sync run at
  500 messages, defer the rest to the next tick
- **Idempotency**: every upsert keys on Graph `id`; classification is skipped
  if the email row already has a `category`

## 6. Polling Architecture

- Vercel Cron entry in `vercel.json`:

  ```json
  { "crons": [{ "path": "/api/sync", "schedule": "*/15 * * * *" }] }
  ```

- `/api/sync` Route Handler verifies `Authorization: Bearer ${CRON_SECRET}`
  before doing any work — unauthenticated requests get a 401
- Function runtime: `nodejs20.x`, `maxDuration: 60` (Vercel Pro)
- Run is logged to `sync_runs (id, started_at, finished_at, messages_seen,
  classified, errors jsonb)`

A second cron at the top of every hour calls `/api/snooze-wake` to flip
snoozed tasks back to open when their `snoozed_until` is reached.

## 7. Supabase Schema (overview)

```
auth_tokens (
  id uuid pk default gen_random_uuid(),
  upn text unique not null,
  access_token_enc bytea not null,
  refresh_token_enc bytea not null,
  expires_at timestamptz not null,
  updated_at timestamptz not null default now()
)

emails (
  id text pk,                      -- Graph message id
  thread_id text not null,         -- conversationId
  folder text not null,            -- 'inbox' | 'sentitems'
  from_address text,
  from_name text,
  subject text,
  snippet text,                    -- first 500 chars of body, plain text
  received_at timestamptz not null,
  is_read boolean not null default false,
  category email_category,         -- nullable until classified
  classification_confidence numeric,
  classification_rationale text,
  raw_headers jsonb,
  created_at timestamptz default now()
)
create index emails_thread_idx on emails(thread_id);
create index emails_received_idx on emails(received_at desc);

tasks (
  id uuid pk default gen_random_uuid(),
  email_id text references emails(id) on delete set null,
  thread_id text,
  title text not null,
  category email_category not null,
  state task_state not null default 'open',
  snoozed_until timestamptz,
  due_at timestamptz,
  auto_resolved boolean not null default false,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  completed_at timestamptz
)
create index tasks_state_idx on tasks(state);
create index tasks_thread_idx on tasks(thread_id);

sync_state (
  folder text pk,                  -- 'inbox' | 'sentitems'
  delta_link text not null,
  updated_at timestamptz default now()
)

sync_runs (
  id uuid pk default gen_random_uuid(),
  started_at timestamptz not null,
  finished_at timestamptz,
  messages_seen int default 0,
  classified int default 0,
  errors jsonb
)
```

### Row-level security

- All tables: RLS **enabled**.
- The browser only reads via `NEXT_PUBLIC_SUPABASE_ANON_KEY` for realtime
  subscriptions on `emails` and `tasks`. Policy: `using (true)` on those two
  tables — acceptable because the app is single-user and the Supabase project
  is private. `auth_tokens`, `sync_state`, `sync_runs`: **no select policy** —
  only the service role can touch them.
- All server-side writes use the service role key, which bypasses RLS.

## 8. UI / UX Requirements

### 8.1 Layout

- Single page, two-pane:
  - **Left**: collapsible category lanes (Need to Respond on top, Newsletter at
    bottom). Each lane shows count and the next 3 messages.
  - **Right**: focused message viewer with the to-do list pinned to the top.
- Top bar: search, sync status indicator (last successful sync timestamp),
  manual "Sync now" button.
- Empty state: a calm "Inbox is clear" view, not a loading spinner.

### 8.2 Interaction model

- Keyboard-first. Mouse always works, keyboard always faster.
- Realtime: the UI subscribes to Supabase realtime on `emails` and `tasks`;
  new classifications appear without refresh.
- No modals for routine actions — everything inline.

### 8.3 Keyboard shortcuts (registered in `lib/keymap.ts`)

| Key       | Action                               |
| --------- | ------------------------------------ |
| `j` / `k` | Next / previous message in lane      |
| `J` / `K` | Next / previous lane                 |
| `r`       | Reply (opens inline composer)        |
| `c`       | Mark current task complete           |
| `s`       | Snooze (1=tomorrow, 2=next week)     |
| `e`       | Archive (move to FYI / done)         |
| `g i`     | Go to Inbox                          |
| `g t`     | Go to To-do list                     |
| `?`       | Show shortcut help overlay           |

### 8.4 Loading & error states

- Initial app load: skeleton lanes with shimmer; never blank
- Sync in progress: subtle spinner in the top bar, never blocks the UI
- Sync failure: persistent banner with last-success timestamp + "retry" button
- Token expired / refresh failed: full-bleed re-auth prompt

## 9. Out of Scope for v1

- Multi-user / multi-tenant
- Mobile app (the web UI must work on mobile, but no native app)
- Offline support
- Search across older mail (only mail seen since first sync is indexed)
- Calendar write operations (Calendar.Read may come with E5; never write)
- Attachments (links shown, downloads happen in Outlook)
- Custom rules engine (categorization is LLM + a small built-in domain allowlist)
- Slack / Teams integrations
- Analytics dashboards
