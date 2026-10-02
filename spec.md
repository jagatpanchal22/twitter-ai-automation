# Twitter AI Automation — Specification

Status: **Draft v0.1** · Owner: Jagat Panchal · Last updated: 2026-10-02

This document defines what we are building and how, before any implementation starts.
Sections marked **[DECISION]** are open choices with a proposed default; confirm or change them before the related milestone begins.

---

## 1. Overview

An AI-assisted system for managing X (Twitter) accounts. Users connect one or more X accounts, generate tweet drafts with an LLM (in a configurable brand voice), review and edit them, schedule them, and have them published automatically. Post performance is collected and shown in a dashboard.

**Stack:** Django + Django REST Framework (backend/API), Celery (background jobs), Redis (broker/cache), PostgreSQL (database), Vue 3 + Vite (dashboard).

## 2. Goals and non-goals

### Goals
- Generate high-quality tweet and thread drafts from a topic, URL, or free-form prompt, in a per-account voice.
- Human-in-the-loop workflow: draft → review/edit → approve → schedule → publish.
- Reliable scheduled publishing that respects X API rate limits and retries on transient failure.
- Basic analytics (impressions, likes, reposts, replies) per post and per account.
- Multi-user, multi-account, with role-based access within a workspace.

### Non-goals (v1)
- Fully autonomous posting with no human approval (opt-in "auto-approve" may come later; see §11).
- Automated replies, mentions, DMs, follows/unfollows, or likes — these carry high platform-policy risk.
- Image/video generation (media *upload* of user-provided files is in scope for v1.1).
- Platforms other than X.
- Mobile apps.

## 3. Users and roles

| Role | Can do |
|---|---|
| **Owner** | Everything, incl. billing-like settings, connecting/disconnecting X accounts, managing members |
| **Editor** | Generate, edit, approve, schedule, and cancel posts; edit voice profiles |
| **Contributor** | Generate and edit drafts; submit for approval (cannot approve or schedule) |
| **Viewer** | Read-only: drafts, schedule, analytics |

Users belong to a **Workspace**. X accounts, voice profiles, and posts belong to a workspace.

## 4. Functional requirements

### 4.1 Authentication & workspaces
- FR-1: Email + password sign-up/login; password reset by email.
- FR-2: Session auth (cookie + CSRF) for the dashboard; JWT is not needed since the SPA is served same-origin. **[DECISION]** default: session auth.
- FR-3: A user creates a workspace on sign-up and can invite others by email with a role.

### 4.2 X account connection
- FR-4: Connect an X account via OAuth 2.0 Authorization Code with PKCE (scopes: `tweet.read tweet.write users.read offline.access`, plus `media.write` when media upload lands).
- FR-5: Store access and refresh tokens **encrypted at rest**; refresh automatically before expiry.
- FR-6: Show connection health (valid / expired / revoked); notify owners when re-auth is needed.
- FR-7: Disconnect revokes tokens and cancels that account's pending scheduled posts.

### 4.3 Voice profiles
- FR-8: Each X account has one or more voice profiles: description of tone, target audience, topics to cover/avoid, example tweets (few-shot), hashtag and emoji policy, language.
- FR-9: One voice profile is the default per account.

### 4.4 AI generation
- FR-10: Generate N (1–10) single-tweet drafts from: a topic, a source URL (fetched & summarized server-side), or a free-form prompt.
- FR-11: Generate a thread (2–15 tweets) from the same inputs.
- FR-12: Rewrite an existing draft: shorter, punchier, more formal, add hook, translate, etc.
- FR-13: Every generated tweet is validated against X's length rules (280 weighted chars, URLs count as 23) and regenerated/trimmed if over.
- FR-14: Generation runs asynchronously (Celery); the UI polls or receives a status update.
- FR-15: Store prompt inputs, model used, token usage, and latency for each generation (for cost tracking and debugging).
- FR-16: Basic content safety: reject/flag outputs that contain disallowed content per workspace "avoid" list; never auto-publish flagged content.

### 4.5 Drafts, review, and approval
- FR-17: Post states: `draft → pending_approval → approved → scheduled → publishing → published`, plus `failed`, `cancelled`.
- FR-18: Editors can edit any field until the post is `publishing`. Editing an approved/scheduled post returns it to `draft` unless the editor re-approves in the same action.
- FR-19: Keep a revision history of post text (who/when/what).
- FR-20: Comment on a draft (simple threaded notes) — **v1.1**.

### 4.6 Scheduling and publishing
- FR-21: Schedule a post for a specific datetime (stored UTC, displayed in the workspace's timezone).
- FR-22: Optional per-account **posting slots** (e.g. weekdays 09:00, 13:00, 18:00) and "add to next free slot".
- FR-23: A periodic Celery beat task (every minute) enqueues due posts; a worker publishes them.
- FR-24: Publishing is **idempotent**: a post is locked (`SELECT … FOR UPDATE SKIP LOCKED`, state → `publishing`) before calling X, and the returned tweet ID is stored; a post with a tweet ID is never posted again.
- FR-25: Threads are published sequentially, each reply referencing the previous tweet ID; partial failure records which tweets went out and resumes from the failed one on retry.
- FR-26: Retries with exponential backoff on 429/5xx/network errors (max 5 attempts); 4xx auth errors mark the account as needing re-auth and fail the post without retry.
- FR-27: Respect X rate limits using response headers (`x-rate-limit-remaining/reset`), tracked per account in Redis.
- FR-28: "Publish now" action for approved posts.

### 4.7 Analytics
- FR-29: Periodically fetch public metrics for published tweets: at +1h, +24h, +7d after publishing, then stop.
- FR-30: Store metric snapshots (time-series), not just latest values.
- FR-31: Dashboard: per-account totals over a date range, top posts, simple chart of engagement over time.
- FR-32: Track AI generation cost per workspace per month.

### 4.8 Notifications
- FR-33: In-app notification list for: post failed, account needs re-auth, post awaiting approval.
- FR-34: Email for failures and re-auth — **v1.1**.

### 4.9 Audit log
- FR-35: Record significant actions (connect/disconnect account, approve, schedule, publish, delete, role changes) with actor and timestamp.

## 5. Non-functional requirements

| Area | Requirement |
|---|---|
| Reliability | Scheduled posts publish within 2 minutes of the target time when X is available. No duplicate posts. |
| Security | Tokens and API keys encrypted at rest (Fernet via `cryptography`, key from env). HTTPS only in production. CSRF on all unsafe requests. Object-level permission checks on every endpoint (workspace scoping). |
| Privacy | No secrets in logs. Generation logs store prompts but not OAuth tokens. |
| Performance | API p95 < 300 ms for non-AI endpoints. Generation p95 < 30 s. |
| Observability | Structured JSON logs, Celery task status visible in Flower (dev) and in logs; Sentry integration optional via env. |
| Maintainability | Type hints, `ruff` + `black` (or `ruff format`) for Python, ESLint + Prettier for Vue; ≥ 80% coverage on core services. |
| Compliance | Follow X Developer Agreement and automation rules: human approval by default, no duplicate content across accounts, no automated engagement actions. |

## 6. Architecture

```
          ┌──────────────┐        HTTPS         ┌──────────────────────┐
 Browser ─┤ Vue 3 SPA    ├──────────────────────▶ Django + DRF (API)   │
          └──────────────┘                       │  gunicorn            │
                                                 └─────┬────────┬───────┘
                                                       │        │ enqueue
                                               ORM     │        ▼
                                         ┌─────────────▼─┐  ┌────────┐
                                         │  PostgreSQL   │  │ Redis  │ broker, rate-limit
                                         └─────────────▲─┘  └───┬────┘ counters, cache
                                                       │        │
                                         ┌─────────────┴────────▼──────┐
                                         │ Celery workers + Celery beat │
                                         └──────┬───────────────┬──────┘
                                                │               │
                                          X API v2          LLM provider API
```

- One Django project; Celery workers share the codebase and settings.
- Separate Celery queues: `generation` (LLM calls, slower), `publishing` (time-sensitive, own worker), `default` (metrics, housekeeping).
- External integrations are wrapped in service classes (`XClient`, `LLMClient`) behind interfaces so they can be mocked in tests and swapped.

## 7. Data model (initial)

All tables have `id` (UUID), `created_at`, `updated_at`.

| Model | Key fields |
|---|---|
| `User` | email (unique, login), name, is_active |
| `Workspace` | name, timezone, monthly_ai_budget_usd (nullable) |
| `Membership` | user → User, workspace → Workspace, role (owner/editor/contributor/viewer); unique (user, workspace) |
| `Invitation` | workspace, email, role, token, expires_at, accepted_at |
| `XAccount` | workspace, x_user_id, username, display_name, avatar_url, access_token_enc, refresh_token_enc, token_expires_at, status (active/needs_reauth/disconnected), scopes |
| `VoiceProfile` | x_account, name, tone, audience, topics, avoid_topics, example_tweets (JSON list), hashtag_policy, emoji_policy, language, is_default |
| `PostingSlot` | x_account, weekday (0–6), local_time |
| `Post` | workspace, x_account, voice_profile (nullable), kind (tweet/thread), status, scheduled_at, published_at, created_by, approved_by, approved_at, attempt_count, last_error, source_generation (nullable) |
| `PostItem` | post, position (0..n), text, x_tweet_id (nullable), published_at — one row for a tweet, n rows for a thread |
| `PostRevision` | post, author, snapshot (JSON of items' text), created_at |
| `Generation` | workspace, x_account, voice_profile, requested_by, mode (tweets/thread/rewrite), input (JSON: topic/url/prompt/options), status (queued/running/succeeded/failed), output (JSON list of candidates), model, input_tokens, output_tokens, cost_usd, latency_ms, error |
| `MetricSnapshot` | post_item, captured_at, impressions, likes, reposts, replies, quotes, bookmarks |
| `Notification` | user, workspace, kind, payload (JSON), read_at |
| `AuditEvent` | workspace, actor (nullable for system), action, target_type, target_id, metadata (JSON) |

Indexes: `Post(status, scheduled_at)` for the due-post scan; `MetricSnapshot(post_item, captured_at)`; `Generation(workspace, created_at)`.

## 8. REST API (v1)

Base path `/api/v1/`. JSON. All resources are scoped to the current workspace (header `X-Workspace-ID` or path prefix — **[DECISION]** default: path prefix `/api/v1/workspaces/{ws}/…`). Pagination: cursor-based.

| Method & path | Purpose |
|---|---|
| `POST /auth/signup`, `/auth/login`, `/auth/logout`, `/auth/password-reset` | Auth |
| `GET /me` | Current user + memberships |
| `GET/POST /workspaces`, `GET/PATCH /workspaces/{ws}` | Workspaces |
| `GET/POST/PATCH/DELETE /workspaces/{ws}/members`, `/invitations` | Team management |
| `GET /workspaces/{ws}/x-accounts`; `POST …/x-accounts/connect` (returns authorize URL); `GET /oauth/x/callback`; `DELETE …/x-accounts/{id}` | X accounts |
| `GET/POST/PATCH/DELETE …/x-accounts/{id}/voice-profiles` | Voice profiles |
| `GET/PUT …/x-accounts/{id}/slots` | Posting slots |
| `POST …/generations` → 202 with id; `GET …/generations/{id}` | AI generation |
| `GET/POST …/posts`; `GET/PATCH/DELETE …/posts/{id}` | Posts (filter by status, account, date range) |
| `POST …/posts/{id}/submit`, `/approve`, `/schedule`, `/unschedule`, `/publish-now`, `/cancel`, `/retry` | State transitions |
| `GET …/posts/{id}/revisions` | Revision history |
| `GET …/analytics/summary?account=&from=&to=`; `GET …/analytics/top-posts` | Analytics |
| `GET …/notifications`; `POST …/notifications/{id}/read` | Notifications |
| `GET …/audit-events` | Audit log (owner only) |

OpenAPI schema generated with `drf-spectacular` at `/api/schema/` and Swagger UI at `/api/docs/` (dev only).

State transitions are implemented in a single service module (`posts/services.py`) — views never set `status` directly.

## 9. Background jobs (Celery)

| Task | Queue | Trigger |
|---|---|---|
| `run_generation(generation_id)` | generation | API request |
| `enqueue_due_posts()` | publishing | beat, every 60 s |
| `publish_post(post_id)` | publishing | from `enqueue_due_posts` / publish-now |
| `refresh_x_token(account_id)` | default | beat, every 15 min for tokens expiring < 30 min |
| `fetch_metrics(post_item_id)` | default | scheduled with `countdown` at +1h, +24h, +7d |
| `cleanup_stale_publishing()` | default | beat, every 10 min — posts stuck in `publishing` > 10 min are reconciled (check X for the tweet before retrying) |

Celery config: `acks_late=True`, `task_reject_on_worker_lost=True`, JSON serializer, time limits per task.

## 10. AI integration

- **Provider [DECISION]:** default Anthropic Claude (`claude-sonnet-5-5` for generation; `claude-haiku-4-5-20251001` for cheap rewrites/validation). Model IDs configurable via env. `LLMClient` interface keeps the provider swappable.
- **Prompting:** system prompt built from the voice profile (tone, audience, avoid list, few-shot examples) + X formatting rules; user prompt from the request input. Ask for structured JSON output (list of candidates; thread as ordered list) and validate it with a schema.
- **URL inputs:** fetch server-side with timeout and size cap, extract main text (e.g. `trafilatura`), block private/internal IPs (SSRF protection).
- **Post-processing:** weighted-length check (`twitter-text` rules), dedupe against the account's last 50 posts (exact and near-duplicate), avoid-list check.
- **Cost control:** record tokens & cost per generation; refuse new generations when the workspace's monthly budget is exceeded.
- **Prompt caching** for the static system/voice portion to reduce cost.

## 11. Open questions / decisions

1. **LLM provider** — Claude by default; any requirement for OpenAI or local models?
2. **X API tier** — Free tier allows very limited posting and no read of metrics; analytics (§4.7) needs Basic or higher. Which tier will be used?
3. **Auto-approve** — should a later version allow fully automated posting for trusted accounts? (Off in v1.)
4. **Multi-tenant SaaS vs single-team self-hosted** — affects signup, billing, and invitations. Default: self-hosted, multi-workspace capable.
5. **Media upload** — v1.1 or v1?
6. **Auth** — session (default) vs JWT if a separate frontend domain or mobile client is planned.

## 12. Project layout

```
twitter-ai-automation/
├── backend/
│   ├── manage.py
│   ├── pyproject.toml
│   ├── config/                 # settings (base/dev/prod), urls, celery.py, wsgi/asgi
│   └── apps/
│       ├── accounts/           # User, Workspace, Membership, Invitation, auth
│       ├── xaccounts/          # XAccount, OAuth, token refresh, XClient
│       ├── voice/              # VoiceProfile, PostingSlot
│       ├── generation/         # Generation, LLMClient, prompts, validators
│       ├── posts/              # Post, PostItem, PostRevision, state machine, publishing tasks
│       ├── analytics/          # MetricSnapshot, metrics tasks, summary queries
│       ├── notifications/
│       └── audit/
├── frontend/                   # Vue 3 + Vite + TypeScript + Pinia + Vue Router
│   └── src/{api,components,views,stores,router}
├── docker/                     # Dockerfiles
├── docker-compose.yml          # postgres, redis, backend, worker-generation, worker-publishing, beat, frontend
├── .env.example
└── .github/workflows/ci.yml    # lint, type-check, tests for backend & frontend
```

## 13. Dashboard (Vue) screens

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

- **Backend:** `pytest` + `pytest-django` + `factory_boy`. Unit tests for the post state machine, length validation, prompt building, rate-limit logic. Integration tests for API endpoints with permission matrix. `XClient` and `LLMClient` mocked (recorded fixtures via `respx`/`responses`). Celery tasks run eagerly in tests.
- **Critical-path tests:** no duplicate publish under concurrent workers; thread partial failure resume; token refresh on 401.
- **Frontend:** Vitest for stores/components; Playwright smoke test for login → generate → schedule.
- **CI:** GitHub Actions on every PR — ruff, mypy, pytest (with Postgres & Redis services), ESLint, vue-tsc, Vitest.

## 15. Configuration (env)

`DJANGO_SECRET_KEY`, `DJANGO_DEBUG`, `ALLOWED_HOSTS`, `DATABASE_URL`, `REDIS_URL`, `FIELD_ENCRYPTION_KEY`, `X_CLIENT_ID`, `X_CLIENT_SECRET`, `X_REDIRECT_URI`, `LLM_PROVIDER`, `ANTHROPIC_API_KEY`, `LLM_MODEL_GENERATION`, `LLM_MODEL_LIGHT`, `EMAIL_URL`, `SENTRY_DSN` (optional), `FRONTEND_URL`.

## 16. Milestones

| # | Milestone | Scope | Done when |
|---|---|---|---|
| M0 | Foundation | Repo layout, Docker Compose, Django + DRF + Celery + Redis + Postgres wired, Vue skeleton, CI, `.env.example` | `docker compose up` serves API health check and SPA; CI green |
| M1 | Auth & workspaces | FR-1 – FR-3, roles & permissions, login UI | Users can sign up, create workspace, invite member |
| M2 | X connection | FR-4 – FR-7, encrypted tokens, refresh task, accounts UI | Can connect/disconnect a real X account |
| M3 | AI generation | Voice profiles, FR-8 – FR-16, compose UI | Generate tweets & threads in a voice, save as drafts |
| M4 | Workflow & scheduling | FR-17 – FR-19, FR-21 – FR-28, queue/calendar UI | Approved posts publish on time, no duplicates, retries work |
| M5 | Analytics & notifications | FR-29 – FR-33, FR-35, analytics UI | Metrics collected & charted; failure notifications visible |
| M6 | Hardening | Security review, load test on scheduler, docs, deployment guide | Production-ready v1 |

v1.1 backlog: media upload, draft comments, email notifications, auto-approve (if approved), best-time-to-post suggestions.
