# Twitter AI Automation — Specification

Status: **Draft v0.2** · Owner: Jagat Panchal · Last updated: 2026-10-02

This document defines what we are building and how, before any implementation starts.
Sections marked **[DECISION]** are open choices with a proposed default; confirm or change them before the related milestone begins.

**Changes from v0.1:** the target platform is now **Vercel (Pro plan)**. Vercel runs serverless functions, not long-lived processes, so the Django + Celery + Vue stack is replaced by a Vercel-native stack: Next.js (TypeScript), Neon Postgres, Upstash Redis, Upstash QStash, and Vercel Cron.

---

## 1. Overview

An AI-assisted system for managing X (Twitter) accounts. Users connect one or more X accounts, generate tweet drafts with an LLM (in a configurable brand voice), review and edit them, schedule them, and have them published automatically. Post performance is collected and shown in a dashboard.

### Tech stack

| Concern | Choice | Why |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript + React** | Native to Vercel; UI and API in one codebase and one deployment |
| UI | Tailwind CSS + shadcn/ui; Recharts for charts; TanStack Query for client data | Fast to build, accessible components |
| Database | **PostgreSQL on Neon** (via Vercel Marketplace) | Serverless Postgres with connection pooling and branch-per-preview |
| ORM / migrations | **Drizzle ORM** + drizzle-kit | Lightweight, works with Neon's serverless driver, SQL-first. **[DECISION]** alternative: Prisma |
| Auth | **Better Auth** (email + password, organization/role plugin) | Built-in workspaces, invitations, roles. **[DECISION]** alternative: Auth.js |
| Background jobs | **Upstash QStash** (HTTP job queue with retries, delays, dedupe) | Serverless replacement for Celery workers |
| Scheduler | **Vercel Cron** (every minute, Pro plan) | Serverless replacement for Celery beat |
| Cache / rate limits / locks | **Upstash Redis** + `@upstash/ratelimit` | HTTP-based Redis that works from serverless functions |
| LLM | **Vercel AI SDK** with the Anthropic provider (Claude) | Structured output with zod schemas; provider swappable |
| Validation | zod (shared by API, forms, LLM output) | One schema source of truth |
| Email | Resend | Invitations, password reset, failure alerts |
| File storage (v1.1) | Vercel Blob | Media uploads |
| Observability | Vercel Logs + Sentry (optional) | |

## 2. Goals and non-goals

### Goals
- Generate high-quality tweet and thread drafts from a topic, URL, or free-form prompt, in a per-account voice.
- Human-in-the-loop workflow: draft → review/edit → approve → schedule → publish.
- Reliable scheduled publishing that respects X API rate limits and retries on transient failure.
- Basic analytics (impressions, likes, reposts, replies) per post and per account.
- Multi-user, multi-account, with role-based access within a workspace.
- Runs entirely on Vercel plus managed serverless services; no servers to operate.

### Non-goals (v1)
- Fully autonomous posting with no human approval (opt-in "auto-approve" may come later; see §12).
- Automated replies, mentions, DMs, follows/unfollows, or likes — these carry high platform-policy risk.
- Image/video generation (media *upload* of user-provided files is in scope for v1.1).
- Platforms other than X.
- Mobile apps.

## 3. Users and roles

| Role | Can do |
|---|---|
| **Owner** | Everything, incl. connecting/disconnecting X accounts, managing members, AI budget |
| **Editor** | Generate, edit, approve, schedule, and cancel posts; edit voice profiles |
| **Contributor** | Generate and edit drafts; submit for approval (cannot approve or schedule) |
| **Viewer** | Read-only: drafts, schedule, analytics |

Users belong to a **Workspace** (Better Auth "organization"). X accounts, voice profiles, and posts belong to a workspace.

## 4. Functional requirements

### 4.1 Authentication & workspaces
- FR-1: Email + password sign-up/login; password reset by email.
- FR-2: Session auth with HTTP-only secure cookies (same-origin, so no tokens in the browser).
- FR-3: A user creates a workspace on sign-up and can invite others by email with a role.

### 4.2 X account connection
- FR-4: Connect an X account via OAuth 2.0 Authorization Code with PKCE (scopes: `tweet.read tweet.write users.read offline.access`, plus `media.write` when media upload lands).
- FR-5: Store access and refresh tokens **encrypted at rest** (AES-256-GCM, key from env); refresh automatically before expiry.
- FR-6: Show connection health (valid / expired / revoked); notify owners when re-auth is needed.
- FR-7: Disconnect revokes tokens and cancels that account's pending scheduled posts.

### 4.3 Voice profiles
- FR-8: Each X account has one or more voice profiles: description of tone, target audience, topics to cover/avoid, example tweets (few-shot), hashtag and emoji policy, language.
- FR-9: One voice profile is the default per account.

### 4.4 AI generation
- FR-10: Generate N (1–10) single-tweet drafts from: a topic, a source URL (fetched & summarized server-side), or a free-form prompt.
- FR-11: Generate a thread (2–15 tweets) from the same inputs.
- FR-12: Rewrite an existing draft: shorter, punchier, more formal, add hook, translate, etc.
- FR-13: Every generated tweet is validated against X's length rules (280 weighted chars, URLs count as 23, via `twitter-text`) and regenerated/trimmed if over.
- FR-14: Generation runs as a background job (QStash → `/api/jobs/generate`); the UI polls the generation status. Results may also be streamed for rewrites, which are short.
- FR-15: Store prompt inputs, model used, token usage, cost, and latency for each generation.
- FR-16: Basic content safety: reject/flag outputs that contain disallowed content per workspace "avoid" list; never auto-publish flagged content.

### 4.5 Drafts, review, and approval
- FR-17: Post states: `draft → pending_approval → approved → scheduled → publishing → published`, plus `failed`, `cancelled`.
- FR-18: Editors can edit any field until the post is `publishing`. Editing an approved/scheduled post returns it to `draft` unless the editor re-approves in the same action.
- FR-19: Keep a revision history of post text (who/when/what).
- FR-20: Comment on a draft (simple threaded notes) — **v1.1**.

### 4.6 Scheduling and publishing
- FR-21: Schedule a post for a specific datetime (stored UTC, displayed in the workspace's timezone).
- FR-22: Optional per-account **posting slots** (e.g. weekdays 09:00, 13:00, 18:00) and "add to next free slot".
- FR-23: Vercel Cron calls `/api/cron/dispatch` every minute. It selects due posts and enqueues one QStash job per post (`/api/jobs/publish`), using the post ID as the QStash deduplication ID.
- FR-24: Publishing is **idempotent**. The job claims the post with a single atomic statement (`UPDATE posts SET status='publishing' … WHERE id=$1 AND status='scheduled' RETURNING *`). If no row returns, the job exits. Each tweet's X ID is saved immediately after it is created, and an item that has an X ID is never posted again.
- FR-25: Threads are published sequentially in one job, each reply referencing the previous tweet ID. On partial failure, the job records which tweets went out, and a retry resumes from the first unpublished item.
- FR-26: Retries: on 429/5xx/network errors the job returns a 5xx so QStash retries with backoff (max 5 attempts). On auth errors (401/403), the account is marked `needs_reauth` and the post fails without retry.
- FR-27: Respect X rate limits using response headers (`x-rate-limit-remaining/reset`), tracked per account in Upstash Redis. If the limit is exhausted, the job reschedules itself (QStash `notBefore` = reset time) instead of calling X.
- FR-28: "Publish now" action for approved posts (enqueues the publish job immediately).

### 4.7 Analytics
- FR-29: Fetch public metrics for published tweets at +1h, +24h, and +7d after publishing (three delayed QStash jobs enqueued at publish time), then stop.
- FR-30: Store metric snapshots (time-series), not just latest values.
- FR-31: Dashboard: per-account totals over a date range, top posts, simple chart of engagement over time.
- FR-32: Track AI generation cost per workspace per month.

### 4.8 Notifications
- FR-33: In-app notification list for: post failed, account needs re-auth, post awaiting approval.
- FR-34: Email for failures and re-auth (Resend) — **v1.1**.

### 4.9 Audit log
- FR-35: Record significant actions (connect/disconnect account, approve, schedule, publish, delete, role changes) with actor and timestamp.

## 5. Non-functional requirements

| Area | Requirement |
|---|---|
| Reliability | Scheduled posts publish within 2 minutes of the target time when X is available. No duplicate posts. |
| Security | Tokens and API keys encrypted at rest. HTTPS only. Every job and cron endpoint verifies its caller (QStash signature / `CRON_SECRET`). Object-level permission checks on every endpoint (workspace scoping). |
| Privacy | No secrets in logs. Generation logs store prompts but never OAuth tokens. |
| Performance | API p95 < 300 ms for non-AI endpoints. Generation p95 < 30 s. |
| Serverless limits | Every function finishes well under its max duration (see §7.2); no in-memory state between requests. |
| Maintainability | Strict TypeScript, ESLint + Prettier, ≥ 80% coverage on `lib/` services. |
| Compliance | Follow X Developer Agreement and automation rules: human approval by default, no duplicate content across accounts, no automated engagement actions. |

## 6. Architecture

```
                         ┌──────────────────────────── Vercel (Pro) ─────────────────────────────┐
 Browser ── HTTPS ──────▶│ Next.js app                                                           │
                         │  • React pages (Server + Client Components)                           │
                         │  • /api/v1/*      REST route handlers (session auth)                  │
                         │  • /api/jobs/*    background job handlers (QStash-signed)             │
                         │  • /api/cron/*    scheduled handlers (Vercel Cron, CRON_SECRET)      │
                         └──────┬───────────────┬──────────────────┬──────────────────┬──────────┘
                                │               │                  │                  │
                        ┌───────▼──────┐ ┌──────▼───────┐  ┌───────▼───────┐   ┌──────▼──────┐
                        │ Neon Postgres│ │ Upstash Redis│  │ Upstash QStash│   │ Vercel Cron │
                        │  (data)      │ │ rate limits, │  │ job queue,    │   │ every 1 min │
                        └──────────────┘ │ cache, locks │  │ retries/delay │   └─────────────┘
                                         └──────────────┘  └───────┬───────┘
                                                                   │ calls back /api/jobs/*
                         External APIs:  X API v2  ·  Anthropic (Claude)  ·  Resend
```

How jobs flow:
1. An API request or cron run **enqueues** a job by publishing a message to QStash, addressed to one of our `/api/jobs/*` URLs.
2. QStash **calls that URL** (immediately or after a delay), signed with a key we verify.
3. The handler does the work. A 2xx response completes the job; a 5xx makes QStash retry with backoff. After the last retry, the message goes to the QStash dead-letter queue, and a failure callback marks the post `failed`.

Code is organised so that route handlers are thin: all business logic lives in `lib/` services (`posts`, `generation`, `x`, …), which are unit-testable without HTTP. External APIs are wrapped in `XClient` and `LLMClient` so tests can mock them.

### 6.1 Background jobs

| Job / endpoint | Triggered by | Notes |
|---|---|---|
| `POST /api/jobs/generate` | API request | LLM call; max duration 300 s |
| `GET /api/cron/dispatch` | Vercel Cron, `* * * * *` | Finds due posts, enqueues publish jobs (dedupe by post ID) |
| `POST /api/jobs/publish` | dispatch / publish-now | Atomic claim, publish tweet(s), enqueue metrics jobs |
| `POST /api/jobs/metrics` | delayed QStash jobs (+1h, +24h, +7d) | Fetch and store one metric snapshot |
| `GET /api/cron/refresh-tokens` | Vercel Cron, every 15 min | Refresh X tokens expiring within 30 min |
| `GET /api/cron/reconcile` | Vercel Cron, every 10 min | Posts stuck in `publishing` > 10 min: check X for the tweet, then mark published or re-enqueue |
| `POST /api/jobs/failed` | QStash failure callback | Marks the job's post/generation `failed`, creates notification |

## 7. Deployment (Vercel)

### 7.1 Environments
- **Production:** `main` branch → production deployment, production Neon branch, production Upstash databases. Cron jobs run **only** here.
- **Preview:** every PR gets a preview deployment with its own Neon database branch (Neon–Vercel integration). Previews share a separate staging Upstash Redis/QStash. A separate "dev" X app is used so previews never post to real accounts. **[DECISION]**
- **Local:** `next dev` with a Neon dev branch, and the QStash local dev server (`npx @upstash/qstash-cli dev`) so jobs can be tested without deploying.

### 7.2 Platform limits to design around (verify current values when starting M0)
- **Function duration:** with Fluid compute on Pro, the default max is 300 s, and it can be raised. Set `maxDuration` per route. A generation or thread publish must fit in one invocation; anything longer is split into multiple QStash jobs.
- **Cron:** Pro allows per-minute schedules. Cron timing is approximate (within the minute) and cron calls are not retried, so `dispatch` must be safe to run late or twice. The atomic claim (FR-24) and QStash dedupe make it so.
- **Database connections:** use Neon's pooled connection string and serverless driver; never open a connection per request without pooling.
- **No local state:** no in-memory caches or files between requests; use Redis or Postgres.
- **Request body size:** ~4.5 MB limit — media uploads (v1.1) go directly from the browser to Vercel Blob.

### 7.3 Migrations and releases
- Drizzle migrations run in the Vercel build step (`drizzle-kit migrate`) against the target branch's database before the new deployment receives traffic. Migrations must be backward-compatible for one release (expand → migrate → contract).
- Vercel cron and function settings live in `vercel.json` in the repo.

## 8. Data model (initial)

Defined with Drizzle in `db/schema.ts`. All tables have `id` (UUID), `created_at`, `updated_at`. Auth tables (`user`, `session`, `account`, `verification`, `organization`, `member`, `invitation`) are generated by Better Auth.

| Model | Key fields |
|---|---|
| `user` | email (unique), name, emailVerified (Better Auth) |
| `organization` (= Workspace) | name, slug, timezone, monthly_ai_budget_usd (nullable) |
| `member` | user, organization, role (owner/editor/contributor/viewer) |
| `invitation` | organization, email, role, expires_at, status |
| `x_accounts` | workspace, x_user_id, username, display_name, avatar_url, access_token_enc, refresh_token_enc, token_expires_at, status (active/needs_reauth/disconnected), scopes |
| `voice_profiles` | x_account, name, tone, audience, topics, avoid_topics, example_tweets (jsonb), hashtag_policy, emoji_policy, language, is_default |
| `posting_slots` | x_account, weekday (0–6), local_time |
| `posts` | workspace, x_account, voice_profile (nullable), kind (tweet/thread), status, scheduled_at, published_at, created_by, approved_by, approved_at, attempt_count, last_error, generation (nullable) |
| `post_items` | post, position (0..n), text, x_tweet_id (nullable), published_at — one row for a tweet, n rows for a thread |
| `post_revisions` | post, author, snapshot (jsonb of items' text) |
| `generations` | workspace, x_account, voice_profile, requested_by, mode (tweets/thread/rewrite), input (jsonb), status (queued/running/succeeded/failed), output (jsonb candidates), model, input_tokens, output_tokens, cost_usd, latency_ms, error |
| `metric_snapshots` | post_item, captured_at, impressions, likes, reposts, replies, quotes, bookmarks |
| `notifications` | user, workspace, kind, payload (jsonb), read_at |
| `audit_events` | workspace, actor (nullable for system), action, target_type, target_id, metadata (jsonb) |

Indexes: `posts(status, scheduled_at)` for the due-post scan; `metric_snapshots(post_item, captured_at)`; `generations(workspace, created_at)`.

## 9. REST API (v1)

Next.js route handlers under `/api/v1/`, JSON, validated with zod. Workspace-scoped resources live under `/api/v1/workspaces/{ws}/…`. Cursor-based pagination. The dashboard uses this same API (plus Server Components that call `lib/` services directly for initial page loads).

| Method & path | Purpose |
|---|---|
| `/api/auth/*` | Better Auth handlers: sign-up, sign-in, sign-out, password reset, invitations |
| `GET /api/v1/me` | Current user + workspaces |
| `GET/POST /workspaces`, `GET/PATCH /workspaces/{ws}` | Workspaces |
| `GET/POST/PATCH/DELETE /workspaces/{ws}/members`, `/invitations` | Team management |
| `GET …/x-accounts`; `POST …/x-accounts/connect` (returns authorize URL); `GET /api/v1/oauth/x/callback`; `DELETE …/x-accounts/{id}` | X accounts |
| `GET/POST/PATCH/DELETE …/x-accounts/{id}/voice-profiles` | Voice profiles |
| `GET/PUT …/x-accounts/{id}/slots` | Posting slots |
| `POST …/generations` → 202 with id; `GET …/generations/{id}` | AI generation |
| `GET/POST …/posts`; `GET/PATCH/DELETE …/posts/{id}` | Posts (filter by status, account, date range) |
| `POST …/posts/{id}/submit`, `/approve`, `/schedule`, `/unschedule`, `/publish-now`, `/cancel`, `/retry` | State transitions |
| `GET …/posts/{id}/revisions` | Revision history |
| `GET …/analytics/summary?account=&from=&to=`; `GET …/analytics/top-posts` | Analytics |
| `GET …/notifications`; `POST …/notifications/{id}/read` | Notifications |
| `GET …/audit-events` | Audit log (owner only) |

State transitions are implemented in one service module (`lib/posts/state.ts`); handlers never set `status` directly. An OpenAPI document is generated from the zod schemas (`zod-openapi`) for reference.

## 10. AI integration

- **Provider [DECISION]:** Anthropic Claude through the Vercel AI SDK: `claude-sonnet-5-5` for generation, `claude-haiku-4-5-20251001` for cheap rewrites and checks. Model IDs are configurable via env. The AI SDK keeps the provider swappable.
- **Prompting:** system prompt built from the voice profile (tone, audience, avoid list, few-shot examples) + X formatting rules; user prompt from the request input. Use `generateObject` with a zod schema (list of candidates; thread as ordered list) so output is always valid JSON.
- **URL inputs:** fetch server-side with timeout and size cap, extract main text (`@mozilla/readability` + `linkedom`), block private/internal IPs (SSRF protection).
- **Post-processing:** weighted-length check (`twitter-text`), dedupe against the account's last 50 posts (exact and near-duplicate), avoid-list check.
- **Cost control:** record tokens & cost per generation; refuse new generations when the workspace's monthly budget is exceeded; per-user rate limit on generation requests (`@upstash/ratelimit`).
- **Prompt caching** for the static system/voice portion to reduce cost.

## 11. Project layout

```
twitter-ai-automation/
├── app/
│   ├── (auth)/                     # login, signup, accept-invite pages
│   ├── (dashboard)/[workspace]/    # overview, compose, queue, drafts, posts/[id], analytics, accounts, settings
│   └── api/
│       ├── auth/[...all]/          # Better Auth
│       ├── v1/                     # REST route handlers
│       ├── jobs/{generate,publish,metrics,failed}/
│       └── cron/{dispatch,refresh-tokens,reconcile}/
├── components/                     # shadcn/ui + app components
├── lib/
│   ├── auth.ts, permissions.ts
│   ├── db/{schema.ts,client.ts}    # Drizzle + Neon
│   ├── posts/                      # state machine, scheduling, publishing
│   ├── generation/                 # prompts, LLMClient, validators
│   ├── x/                          # XClient, OAuth, rate-limit tracking
│   ├── jobs/                       # QStash enqueue + signature verification helpers
│   ├── analytics/, notifications/, audit/
│   └── crypto.ts                   # token encryption
├── drizzle/                        # generated migrations
├── tests/                          # unit, integration, e2e (Playwright)
├── vercel.json                     # crons, function maxDuration
├── .env.example
└── .github/workflows/ci.yml
```

## 12. Open questions / decisions

1. **X API tier** — the Free tier allows very limited posting and cannot read metrics; analytics (§4.7) needs Basic or higher. Which tier will be used? This is the biggest cost and feasibility factor, independent of Vercel.
2. **LLM provider** — Claude by default; any requirement for another provider?
3. **ORM** — Drizzle (default) or Prisma.
4. **Auth library** — Better Auth (default) or Auth.js.
5. **Auto-approve** — should a later version allow fully automated posting for trusted accounts? (Off in v1.)
6. **Preview deployments and X** — confirm a separate dev X app for previews.
7. **Media upload** — v1.1 (default) or v1?

## 13. Dashboard screens

1. **Login / signup / accept invite**
2. **Overview** — upcoming posts, pending approvals, failed posts, account health, key metrics
3. **Compose / Generate** — choose account + voice, input topic/URL/prompt, view candidates, edit inline with live character count, save as draft / submit / schedule
4. **Queue / Calendar** — week/month calendar and list view of scheduled posts; drag to reschedule
5. **Drafts & Approvals** — filterable list; approve/reject
6. **Post detail** — items, revisions, status history, metrics
7. **Analytics** — date range, per account, charts and top posts
8. **Accounts** — connect/disconnect X, voice profiles, posting slots
9. **Settings** — workspace, members & roles, timezone, AI budget

## 14. Testing strategy

- **Unit (Vitest):** post state machine, length validation, prompt building, rate-limit logic, token encryption, permission checks.
- **Integration (Vitest):** route handlers against a real Postgres (Neon branch in CI, or a Postgres service container), with X, Anthropic, and QStash mocked via MSW. Job handlers are called directly with signed test payloads.
- **Critical-path tests:** no duplicate publish when two publish jobs run at once; thread partial failure resumes correctly; token refresh on 401; dispatch running twice in one minute.
- **E2E (Playwright):** login → generate → approve → schedule against a preview deployment, with a mocked X.
- **CI (GitHub Actions):** typecheck (`tsc`), ESLint, Vitest, `drizzle-kit check`, `next build` on every PR. Vercel creates the preview deployment.

## 15. Configuration (env)

Set in Vercel project settings (Marketplace integrations add the database ones automatically):

`DATABASE_URL` (Neon, pooled), `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `TOKEN_ENCRYPTION_KEY`, `X_CLIENT_ID`, `X_CLIENT_SECRET`, `X_REDIRECT_URI`, `ANTHROPIC_API_KEY`, `LLM_MODEL_GENERATION`, `LLM_MODEL_LIGHT`, `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `QSTASH_TOKEN`, `QSTASH_CURRENT_SIGNING_KEY`, `QSTASH_NEXT_SIGNING_KEY`, `CRON_SECRET`, `RESEND_API_KEY`, `APP_URL`, `SENTRY_DSN` (optional).

## 16. Milestones

| # | Milestone | Scope | Done when |
|---|---|---|---|
| M0 | Foundation | Next.js + TypeScript + Tailwind/shadcn, Drizzle + Neon, Upstash Redis/QStash clients, `vercel.json`, CI, `.env.example`, health endpoint, one example cron and one example QStash job wired end to end | Deployed to Vercel; CI green; example job and cron run in production |
| M1 | Auth & workspaces | FR-1 – FR-3, roles & permissions, auth UI | Users can sign up, create workspace, invite member |
| M2 | X connection | FR-4 – FR-7, encrypted tokens, refresh cron, accounts UI | Can connect/disconnect a real X account |
| M3 | AI generation | Voice profiles, FR-8 – FR-16, compose UI | Generate tweets & threads in a voice, save as drafts |
| M4 | Workflow & scheduling | FR-17 – FR-19, FR-21 – FR-28, queue/calendar UI | Approved posts publish on time, no duplicates, retries work |
| M5 | Analytics & notifications | FR-29 – FR-33, FR-35, analytics UI | Metrics collected & charted; failure notifications visible |
| M6 | Hardening | Security review, concurrency tests on publishing, docs, runbook | Production-ready v1 |

v1.1 backlog: media upload (Vercel Blob), draft comments, email notifications, auto-approve (if approved), best-time-to-post suggestions.
