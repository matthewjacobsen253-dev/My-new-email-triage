# CLAUDE.md

Project context for Claude Code. Read this first on every session.

## Project: My Email Triage

A personal, single-user productivity tool that connects to a Microsoft 365 work
account, runs automated triage on inbound mail every 15 minutes, and maintains a
self-updating to-do list derived from email context. Goal: zero-friction daily
inbox-zero workflow that the user actually keeps using.

## Tech Stack

| Layer            | Choice                                                                 |
| ---------------- | ---------------------------------------------------------------------- |
| Framework        | Next.js 14 (App Router, Route Handlers, React Server Components)       |
| Language         | TypeScript (strict mode, `noUncheckedIndexedAccess` on)                |
| UI               | Tailwind CSS + shadcn/ui primitives, Lucide icons                      |
| Database         | Supabase Postgres + `pgcrypto` (for token encryption at rest)          |
| Auth             | Microsoft OAuth 2.0 via `@azure/msal-node` — direct, no Supabase Auth  |
| Hosting          | Vercel (production environment only for OAuth)                         |
| Polling          | Vercel Cron Jobs → `/api/sync` Route Handler → Graph delta query       |
| Classification   | Anthropic Claude API (`claude-sonnet-4-6`) with prompt caching         |
| Package manager  | pnpm                                                                   |
| Tests            | Vitest (unit), Playwright (smoke E2E for OAuth + triage loop)          |

## Commands

- `pnpm dev` — local dev server on http://localhost:3000
- `pnpm build` — production build (also what Vercel runs)
- `pnpm typecheck` — `tsc --noEmit`
- `pnpm lint` — eslint with `next/core-web-vitals`
- `pnpm test` — vitest
- `pnpm test:e2e` — Playwright
- `pnpm db:migrate` — apply migrations from `supabase/migrations/`
- `pnpm db:reset` — local supabase reset (dev only — destructive)
- `pnpm sync:once` — invoke the sync handler locally with a dev `CRON_SECRET`

## Required Environment Variables

Server-only (never expose to the client):

- `MS_CLIENT_ID` — Microsoft Entra app registration client ID
- `MS_TENANT_ID` — work account tenant ID (use the specific tenant, not `common`)
- `MS_CLIENT_SECRET` — Entra client secret
- `MS_REDIRECT_URI` — must exactly match a URI registered in Entra
- `OWNER_EMAIL` — single-user allowlist; OAuth callbacks for any other UPN are rejected
- `SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY` — server-only, bypasses RLS
- `ANTHROPIC_API_KEY`
- `CRON_SECRET` — bearer token Vercel Cron sends in `Authorization` header
- `TOKEN_ENCRYPTION_KEY` — 32-byte hex key for encrypting Graph tokens at rest

Client-safe:

- `NEXT_PUBLIC_SUPABASE_URL` — used by the client only for realtime subscriptions
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — RLS-protected reads for realtime channel

## Conventions

- All Supabase access goes through `lib/db/` — never inline SQL elsewhere
- All Microsoft Graph calls go through `lib/graph/` — never `fetch` `graph.microsoft.com` directly
- Anthropic SDK calls go through `lib/triage/classifier.ts` with prompt caching on the system block
- Schema changes require a new file in `supabase/migrations/` with a `YYYYMMDDHHMMSS_` prefix
- Server-only secrets must never be imported from a `'use client'` file
- Keyboard shortcuts are registered in a single `lib/keymap.ts` to avoid conflicts
- Time values are stored as `timestamptz` in Postgres and rendered in the user's local timezone in the UI
- Errors thrown by sync are logged to the `sync_runs` table; do not silently swallow

## What Claude Code Must Never Do

- Commit `.env`, `.env.local`, or any file containing real tokens, secrets, or keys
- Mutate the Supabase schema directly via SQL — always create a migration file
- Print, log, or echo stored OAuth tokens (refresh tokens are bearer credentials)
- Call the Microsoft Graph API with application permissions — this app is delegated only
- Add a Supabase Auth signup flow or auth.users row — auth is owned by the MS OAuth bridge
- Register additional OAuth redirect URIs without updating `DECISIONS.md` and Entra
- Bypass `CRON_SECRET` verification on `/api/sync`
- Use the `SUPABASE_SERVICE_ROLE_KEY` from any `'use client'` file or the browser bundle
- Skip the `OWNER_EMAIL` check after OAuth — this is the only authorization gate
- Add multi-tenant or multi-user features without explicit user confirmation
