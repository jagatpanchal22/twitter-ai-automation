# Twitter AI Automation — Specification

Status: **Draft v0.5** · Owner: Jagat Panchal · Last updated: 2026-10-03

This document defines what we are building and how, before any implementation starts.
Sections marked **[DECISION]** are open choices with a proposed default; confirm or change them before the related milestone begins.

**Changes from v0.1:** the target platform is now **Vercel**. Vercel runs serverless functions, not long-lived processes, so the Django + Celery + Vue stack is replaced by a Vercel-native stack: Next.js (TypeScript), Neon Postgres, Upstash Redis, Upstash QStash, and Vercel Cron.

**Changes in v0.4:** the target is the **Vercel Hobby (free) plan**. Hobby cron jobs run at most once a day, so publishing no longer depends on an every-minute cron. Instead, each scheduled post gets its own **delayed QStash message** that fires at the post's time. One daily Vercel Cron job handles posts scheduled more than 7 days ahead and housekeeping. Every service in the stack has a free tier that fits this design (see §7.2).

**Changes in v0.5:** content is now produced by **five AI agents**, each with one job: Research, Writer, Reviewer, Image Creator and Analyst (§4.10, §10). The owner chose **fully automatic** posting ("Autopilot"): the agents research, write, review, illustrate, schedule and publish without a human step, inside the guardrails in §4.11. Image generation moves into v1.

---

## 1. Overview

An AI system that runs X (Twitter) accounts. Users connect one or more X accounts. A team of five AI agents finds topics, writes posts in the account's voice, reviews them, creates matching images, and learns from performance. With **Autopilot** on, approved-by-Reviewer posts are scheduled and published with no human step; with it off, posts wait in an approval queue. Users can also write and generate posts manually at any time. Performance and costs are shown in a dashboard.

### Tech stack

| Concern | Choice | Why |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript + React** | Native to Vercel; UI and API in one codebase and one deployment |
| UI | Tailwind CSS + shadcn/ui; Recharts for charts; TanStack Query for client data | Fast to build, accessible components |
| Database | **PostgreSQL on Neon** (via Vercel Marketplace) | Serverless Postgres with connection pooling and branch-per-preview |
| ORM / migrations | **Drizzle ORM** + drizzle-kit | Lightweight, works with Neon's serverless driver, SQL-first. **[DECISION]** alternative: Prisma |
| Auth | **Better Auth** (email + password, organization/role plugin) | Built-in workspaces, invitations, roles. **[DECISION]** alternative: Auth.js |
| Background jobs | **Upstash QStash** (HTTP job queue with retries, delays, dedupe) | Serverless replacement for Celery workers |
| Scheduler | **Delayed QStash messages** (one per scheduled post) + **one daily Vercel Cron** job | Works on the Hobby plan, where cron can run only once a day |
| Cache / rate limits / locks | **Upstash Redis** + `@upstash/ratelimit` | HTTP-based Redis that works from serverless functions |
| LLM (agents) | **Anthropic TypeScript SDK** (`@anthropic-ai/sdk`) calling Claude | Official SDK; structured outputs with zod (`messages.parse`), server-side web search for the Research agent, image input for the Reviewer |
| Image generation | An image-generation API **[DECISION]** (e.g. OpenAI, Google, or Flux via fal.ai) called through the Vercel AI SDK's `generateImage`, plus **template graphics** rendered with `@vercel/og` | AI images for illustrations; templates for quote cards and stat cards (free, sharp text) |
| Validation | zod (shared by API, forms, LLM output) | One schema source of truth |
| Email | Resend | Invitations, password reset, failure alerts |
| File storage | Vercel Blob | Generated images (Hobby includes 1 GB; images are deleted 30 days after publishing) |
| Observability | Vercel Logs + Sentry (optional) | |

## 2. Goals and non-goals

### Goals
- Generate high-quality tweet and thread drafts from a topic, URL, or free-form prompt, in a per-account voice.
- A team of AI agents with dedicated duties that keeps each account supplied with good, on-voice posts every day.
- **Autopilot:** fully automatic research → write → review → image → schedule → publish, with guardrails (§4.11).
- Manual mode for any account: posts wait for human approval.
- Reliable scheduled publishing that respects X API rate limits and retries on transient failure.
- Basic analytics (impressions, likes, reposts, replies) per post and per account.
- Multi-user, multi-account, with role-based access within a workspace.
- Runs entirely on Vercel plus managed serverless services; no servers to operate.

### Non-goals (v1)
- Automated replies, mentions, DMs, follows/unfollows, or likes — these carry high platform-policy risk.
- Video generation. (AI images are in scope; uploading your own images is v1.1.)
- Using trending hashtags or topics unrelated to the account's niche (X treats this as spam).
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
- FR-4: Connect an X account via OAuth 2.0 Authorization Code with PKCE (scopes: `tweet.read tweet.write users.read offline.access media.write`).
- FR-5: Store access and refresh tokens **encrypted at rest** (AES-256-GCM, key from env); refresh automatically before expiry.
- FR-6: Show connection health (valid / expired / revoked); notify owners when re-auth is needed.
- FR-7: Disconnect revokes tokens and cancels that account's pending scheduled posts.

### 4.3 Voice profiles
- FR-8: Each X account has one or more voice profiles: description of tone, target audience, topics to cover/avoid, example tweets (few-shot), hashtag and emoji policy, language.
- FR-9: One voice profile is the default per account.
- FR-9a: **Auto-create from history.** When an account is connected, the app can read its recent posts (e.g. last 50; costs ~$0.25 in X reads) and have Claude draft the voice profile from them. The user reviews and edits the draft. The Analyst agent later suggests updates based on what performs well.

### 4.4 AI generation
- FR-10: Generate N (1–10) single-tweet drafts from: a topic, a source URL (fetched & summarized server-side), or a free-form prompt.
- FR-11: Generate a thread (2–15 tweets) from the same inputs.
- FR-12: Rewrite an existing draft: shorter, punchier, more formal, add hook, translate, etc.
- FR-13: Every generated tweet is validated against X's length rules (weighted characters, URLs count as 23, via `twitter-text`) and regenerated/trimmed if over. The limit is 280 for standard accounts. For accounts with an X Premium subscription, long posts are allowed: a per-account `max_post_length` setting (default 280) lets the user opt in to longer single posts as an alternative to threads. **[DECISION]** confirm X API support for long posts on the user's account during M2.
- FR-14: Generation runs as a background job (QStash → `/api/jobs/agent`, running the Writer then the Reviewer); the UI polls the generation status. Results may also be streamed for rewrites, which are short.
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
- FR-23: When a post is scheduled for within the next 7 days, the app immediately publishes a **delayed QStash message** to `/api/jobs/publish` that fires at `scheduled_at` (QStash free tier allows delays up to 7 days). The message ID is stored on the post. Posts scheduled further ahead get their message later from the daily cron (§6.1), once they are within 7 days.
- FR-23a: Rescheduling or cancelling a post deletes its pending QStash message and, for a reschedule, creates a new one. As a safety net, the publish job payload carries the `scheduled_at` it was created for; if that no longer matches the post, or the post is no longer `scheduled`, the job does nothing.
- FR-24: Publishing is **idempotent**. The job claims the post with a single atomic statement (`UPDATE posts SET status='publishing' … WHERE id=$1 AND status='scheduled' RETURNING *`). If no row returns, the job exits. Each tweet's X ID is saved immediately after it is created, and an item that has an X ID is never posted again.
- FR-25: Threads are published sequentially in one job, each reply referencing the previous tweet ID. On partial failure, the job records which tweets went out, and a retry resumes from the first unpublished item.
- FR-26: Retries: on 429/5xx/network errors the job returns a 5xx so QStash retries with backoff (max 5 attempts). On auth errors (401/403), the account is marked `needs_reauth` and the post fails without retry.
- FR-27: Respect X rate limits using response headers (`x-rate-limit-remaining/reset`), tracked per account in Upstash Redis. If the limit is exhausted, the job reschedules itself (QStash `notBefore` = reset time) instead of calling X.
- FR-28: "Publish now" action for approved posts (enqueues the publish job immediately).

### 4.7 Analytics
- FR-29: Fetch public metrics for published tweets at +1h, +24h, and +7d after publishing (three delayed QStash jobs enqueued at publish time), then stop.
- FR-30: Store metric snapshots (time-series), not just latest values.
- FR-31: Dashboard: per-account totals over a date range, top posts, simple chart of engagement over time.
- FR-32: Track cost per workspace per month for AI generation **and X API usage** (X bills pay-per-use per post created and per post read; posts containing a link cost more — see §12). Show both in the dashboard, with an optional monthly X API budget that blocks scheduling new posts when exceeded.

### 4.8 Notifications
- FR-33: In-app notification list for: post failed, account needs re-auth, post awaiting approval.
- FR-34: Email for failures and re-auth (Resend) — **v1.1**.

### 4.9 Audit log
- FR-35: Record significant actions (connect/disconnect account, approve, schedule, publish, delete, role changes, autopilot on/off) with actor and timestamp. Agent actions are recorded with the agent as actor.

### 4.10 AI agents
Each agent has one duty, its own prompt, its own input/output schema (zod), and its own model and effort setting. Agents never call each other directly: each runs as a separate background job, and a **pipeline** (code, not an LLM) passes one agent's saved output to the next (§6.2). Every run is logged in `agent_runs`.

| Agent | Duty | Input | Output | Runs |
|---|---|---|---|---|
| **Research** | Find what to post about | Account niche and topics, content sources (RSS feeds, websites, keywords), Analyst insights, recent posts (to avoid repeats) | A daily **content brief**: N topic ideas, each with angle, why now, and source links with key facts | Daily, per autopilot account |
| **Writer** | Write posts in the account's voice | One topic idea + its sources, voice profile, Analyst insights, Reviewer feedback (on rewrite) | A tweet or thread (text per item), plus an **image brief** describing the visual | Per idea; again on rewrite |
| **Reviewer / Editor** | Gatekeeper for quality and safety | Draft, sources, voice profile, avoid list, recent posts; later the image | Scores (voice match, hook, clarity, factual support, safety), pass/fail, and specific edit notes | After Writer; after Image Creator |
| **Image Creator** | Make the visual for an approved draft | Reviewer-passed text, image brief, brand settings (colours, logo, style) | One image (AI-generated or template card), alt text | After text passes review |
| **Analyst** | Learn what works | Metric snapshots, posts, topics, posting times | **Insights**: what topics/hooks/formats/times perform best; suggested posting slots; suggested voice-profile tweaks | Weekly, per account |

- FR-36: **Research agent** uses Claude's server-side web search and the account's content sources. Every fact it passes on must carry a source URL. It skips topics the account posted about in the last 14 days and topics on the avoid list.
- FR-37: **Writer agent** produces drafts that follow FR-13 (length) and the voice profile. Manual "Generate" in the Compose page (FR-10 – FR-12) uses the same Writer agent.
- FR-38: **Reviewer agent** passes a draft only if every score meets the account's threshold (default 7/10) and the safety check passes. A failed draft goes back to the Writer with the Reviewer's notes, at most **2 rewrites**; after that it is discarded (autopilot) or left as a draft for a human (manual mode). Claims not supported by the Research sources fail the factual check.
- FR-39: **Image Creator agent** decides between a template card (quotes, numbers, lists) and an AI illustration, writes the image prompt, generates the image, stores it in Vercel Blob, and writes alt text. The Reviewer then checks the image (Claude reads images): matches the post, no garbled text, no real people's likenesses, no logos or brands, nothing unsafe. A failed image is regenerated once; if it fails again the post goes out as text only.
- FR-40: **Analyst agent** runs weekly. Its insights are saved and included in the Research and Writer prompts. In autopilot it may move posting slots within the bounds the user set (FR-43); voice-profile changes are suggestions only and need a human.
- FR-41: Agents page in the dashboard: for each post, the full trail — research brief, drafts, review scores and notes, images, and cost per agent.

### 4.11 Autopilot (fully automatic posting)
- FR-42: Autopilot is a per-account switch. When on, a Reviewer-passed post (text and image) is **approved by the system** and placed into the next free posting slot, then published by the normal scheduling flow (§4.6). When off, Reviewer-passed posts wait in the approval queue.
- FR-43: **Guardrails** (per account, editable):
  - posts per day (default 3, hard maximum 10) and minimum gap between posts (default 2 h);
  - allowed posting hours;
  - Reviewer score threshold;
  - topics and words to avoid; sensitive categories always blocked (politics, tragedies and breaking disasters, health or financial advice, legal claims) unless explicitly allowed;
  - no posting about a news event less than 2 hours old (facts are often wrong early);
  - monthly budgets for AI, images and X API; when one is reached, autopilot pauses and notifies.
- FR-44: **Emergency stop:** one button pauses autopilot for all accounts and cancels everything scheduled in the next 24 hours. Also available per account.
- FR-45: **Shadow mode** (recommended first 7 days): agents run fully but posts go to the approval queue marked "would have posted", so the owner can check quality before switching to live autopilot.
- FR-46: **Daily digest** (in-app; email in v1.1): what was posted, what was rejected and why, spend.
- FR-47: Autopilot accounts should turn on X's **"automated account" label** in X settings (X's automation rules); the app reminds the owner when autopilot is first enabled.

## 5. Non-functional requirements

| Area | Requirement |
|---|---|
| Reliability | Scheduled posts publish within 2 minutes of the target time when X is available. No duplicate posts. |
| Security | Tokens and API keys encrypted at rest. HTTPS only. Every job and cron endpoint verifies its caller (QStash signature / `CRON_SECRET`). Object-level permission checks on every endpoint (workspace scoping). |
| Privacy | No secrets in logs. Generation logs store prompts but never OAuth tokens. |
| Performance | API p95 < 300 ms for non-AI endpoints. Each agent step finishes in one function invocation (< 300 s). |
| Serverless limits | Every function finishes well under its max duration (see §7.2); no in-memory state between requests. |
| Maintainability | Strict TypeScript, ESLint + Prettier, ≥ 80% coverage on `lib/` services. |
| Compliance | Follow X Developer Agreement and automation rules: autopilot guardrails (§4.11), automated-account label, no duplicate content across accounts, no unrelated trending hashtags, no automated engagement actions (replies, likes, follows). |

## 6. Architecture

```
                         ┌─────────────────────────── Vercel (Hobby) ────────────────────────────┐
 Browser ── HTTPS ──────▶│ Next.js app                                                           │
                         │  • React pages (Server + Client Components)                           │
                         │  • /api/v1/*      REST route handlers (session auth)                  │
                         │  • /api/jobs/*    background job handlers (QStash-signed)             │
                         │  • /api/cron/*    scheduled handlers (Vercel Cron, CRON_SECRET)      │
                         └──────┬───────────────┬──────────────────┬──────────────────┬──────────┘
                                │               │                  │                  │
                        ┌───────▼──────┐ ┌──────▼───────┐  ┌───────▼───────┐   ┌──────▼──────┐
                        │ Neon Postgres│ │ Upstash Redis│  │ Upstash QStash│   │ Vercel Cron │
                        │  (data)      │ │ rate limits, │  │ job queue,    │   │ once a day  │
                        └──────────────┘ │ cache, locks │  │ retries/delay │   └─────────────┘
                                         └──────────────┘  └───────┬───────┘
                                                                   │ calls back /api/jobs/*
                         External APIs:  X API v2  ·  Anthropic (Claude)  ·  Image API  ·  Vercel Blob  ·  Resend
```

How jobs flow:
1. An API request (or the daily cron) **enqueues** a job by publishing a message to QStash, addressed to one of our `/api/jobs/*` URLs.
2. QStash **calls that URL** — immediately, or at a set time for scheduled posts and metrics — signed with a key we verify.
3. The handler does the work. A 2xx response completes the job; a 5xx makes QStash retry with backoff. After the last retry, the message goes to the QStash dead-letter queue, and a failure callback marks the post `failed`.

Code is organised so that route handlers are thin: all business logic lives in `lib/` services (`posts`, `generation`, `x`, …), which are unit-testable without HTTP. External APIs are wrapped in `XClient` and `LLMClient` so tests can mock them.

### 6.1 Background jobs

| Job / endpoint | Triggered by | Notes |
|---|---|---|
| `POST /api/jobs/agent` | pipeline / API request | Runs one agent step (§6.2); max duration 300 s |
| `POST /api/jobs/publish` | delayed QStash message at `scheduled_at` / publish-now | Check payload still matches post, atomic claim, publish tweet(s), enqueue metrics jobs |
| `POST /api/jobs/metrics` | delayed QStash jobs (+1h, +24h, +7d) | Fetch and store one metric snapshot |
| `GET /api/cron/daily` | Vercel Cron, once a day (`0 3 * * *` UTC) | (0) start today's agent pipeline for each autopilot account, and the weekly Analyst run on Mondays; (1) create QStash messages for posts that are now within 7 days and have none; (2) find posts stuck in `publishing` > 10 min, check X for the tweet, then mark published or re-enqueue; (3) refresh X tokens for accounts not used in the last 24 h so they don't lapse; (4) find `scheduled` posts whose time has passed without publishing and publish them now (safety net) |
| `POST /api/jobs/failed` | QStash failure callback | Marks the job's post/generation `failed`, creates notification |

### 6.2 Agent pipeline

The pipeline is ordinary code that moves one item through the agents. Each step is its own QStash job (`/api/jobs/agent` with `{pipelineRunId, step}`), so a slow step never hits the function time limit, and a failed step is retried on its own.

```
Daily cron (per autopilot account)
   │
   ▼
Research ──▶ content brief (N ideas, with sources)
   │  one branch per idea
   ▼
Writer ──▶ draft + image brief
   │
   ▼
Reviewer ── fail (≤2 times) ──▶ back to Writer with notes
   │ pass                         (3rd fail: discard / leave as draft)
   ▼
Image Creator ──▶ image + alt text
   │
   ▼
Reviewer (image) ── fail ──▶ regenerate once, else text-only
   │ pass
   ▼
Autopilot on:  system-approve → next free slot → delayed publish job (§4.6)
Autopilot off: approval queue (shadow mode: marked "would have posted")
   │
   ▼
Published → metrics at +1h/+24h/+7d → Analyst (weekly) → insights feed Research & Writer
```

- State lives in `pipeline_runs` (one per account per day) and `agent_runs` (one per step). Steps are idempotent: a retried step first checks whether its output already exists.
- The daily cron runs once a day and may drift by up to an hour, so the pipeline plans **tomorrow's** posts. Being late never delays a post.
- QStash messages per post: about 10 (writer, 1–3 reviews, image, image review, publish, 3 metric checks). At 5 posts/day per account this stays far below the free 1,000/day.

**Token refresh is on demand:** before every X API call, `XClient` refreshes the access token if it expires within 5 minutes, holding a short Redis lock so two jobs don't refresh the same account at once. This replaces the every-15-minute refresh cron, which Hobby can't run.

## 7. Deployment (Vercel)

### 7.1 Environments
- **Production:** `main` branch → production deployment, production Neon branch, production Upstash databases. Cron jobs run **only** here.
- **Preview:** every PR gets a preview deployment with its own Neon database branch (Neon–Vercel integration). Previews share a separate staging Upstash Redis/QStash. A separate "dev" X app is used so previews never post to real accounts. **[DECISION]**
- **Local:** `next dev` with a Neon dev branch, and the QStash local dev server (`npx @upstash/qstash-cli dev`) so jobs can be tested without deploying.

### 7.2 Platform limits to design around (verify current values when starting M0)
- **Function duration:** Hobby with Fluid compute allows up to 300 s per invocation. Set `maxDuration` per route. A generation or thread publish must fit in one invocation; anything longer is split into multiple QStash jobs.
- **Cron (Hobby):** at most 2 cron jobs, each at most once a day, and the run time can drift by up to an hour. We use one (`/api/cron/daily`), and nothing time-sensitive depends on it. Cron calls are not retried, so it must be safe to run late or twice.
- **Hobby usage allowance (monthly):** about 1 million function invocations and 4 hours of active CPU. Waiting on the X API or Claude is not active CPU, so personal use stays well inside this.
- **Hobby is for non-commercial use only.** If the app is used for paid client work or a business, Vercel requires the Pro plan. Moving to Pro needs no code change. Optionally, publishing could then use a finer-grained cron as an extra safety net.
- **QStash free tier:** 1,000 messages per day and a maximum delay of 7 days. Each post uses about 10 messages through the agent pipeline, so this covers about 100 posts a day across all accounts.
- **Vercel Blob (Hobby):** about 1 GB storage. Images are stored compressed and deleted 30 days after publishing; X keeps its own copy.
- **Neon and Upstash Redis free tiers** are sufficient for one person or a small team.
- **Database connections:** use Neon's pooled connection string and serverless driver; never open a connection per request without pooling.
- **No local state:** no in-memory caches or files between requests; use Redis or Postgres.
- **Request body size:** ~4.5 MB limit — generated images are uploaded to Blob from the server; user uploads (v1.1) go directly from the browser to Vercel Blob.

### 7.3 Migrations and releases
- Drizzle migrations run in the Vercel build step (`drizzle-kit migrate`) against the target branch's database before the new deployment receives traffic. Migrations must be backward-compatible for one release (expand → migrate → contract).
- Vercel cron and function settings live in `vercel.json` in the repo.

## 8. Data model (initial)

Defined with Drizzle in `db/schema.ts`. All tables have `id` (UUID), `created_at`, `updated_at`. Auth tables (`user`, `session`, `account`, `verification`, `organization`, `member`, `invitation`) are generated by Better Auth.

| Model | Key fields |
|---|---|
| `user` | email (unique), name, emailVerified (Better Auth) |
| `organization` (= Workspace) | name, slug, timezone, monthly_ai_budget_usd (nullable), monthly_x_budget_usd (nullable) |
| `member` | user, organization, role (owner/editor/contributor/viewer) |
| `invitation` | organization, email, role, expires_at, status |
| `x_accounts` | workspace, x_user_id, username, max_post_length (default 280), niche, autopilot_mode (off/shadow/live), guardrails (jsonb: posts_per_day, min_gap_minutes, allowed_hours, score_threshold, blocked_categories, budgets), brand (jsonb: colours, logo_url, image_style), display_name, avatar_url, access_token_enc, refresh_token_enc, token_expires_at, status (active/needs_reauth/disconnected), scopes |
| `voice_profiles` | x_account, name, tone, audience, topics, avoid_topics, example_tweets (jsonb), hashtag_policy, emoji_policy, language, is_default |
| `posting_slots` | x_account, weekday (0–6), local_time |
| `posts` | workspace, x_account, voice_profile (nullable), origin (manual/autopilot), pipeline_run (nullable), review_score (nullable), kind (tweet/thread), status, scheduled_at, qstash_message_id (nullable), published_at, created_by, approved_by, approved_at, attempt_count, last_error, generation (nullable) |
| `post_items` | post, position (0..n), text, x_tweet_id (nullable), published_at — one row for a tweet, n rows for a thread |
| `post_media` | post_item, blob_url, kind (ai_image/template_card), prompt, alt_text, x_media_id (nullable), review_status, cost_usd |
| `content_sources` | x_account, kind (rss/website/keyword), value, is_active |
| `content_briefs` | x_account, pipeline_run, ideas (jsonb: topic, angle, why_now, sources[{url, title, key_facts}]) |
| `pipeline_runs` | x_account, run_date, kind (daily/manual/analyst), status, current_step, posts_created, cost_usd |
| `agent_runs` | pipeline_run (nullable), post (nullable), agent (research/writer/reviewer/image/analyst), attempt, input (jsonb), output (jsonb), status, model, input_tokens, output_tokens, web_searches, image_count, cost_usd, latency_ms, error |
| `account_insights` | x_account, period_start, period_end, insights (jsonb), suggested_slots (jsonb), suggested_voice_changes (jsonb), applied_at |
| `post_revisions` | post, author, snapshot (jsonb of items' text) |
| `generations` | workspace, x_account, voice_profile, requested_by, mode (tweets/thread/rewrite), input (jsonb), status (queued/running/succeeded/failed), output (jsonb candidates), model, input_tokens, output_tokens, cost_usd, latency_ms, error |
| `metric_snapshots` | post_item, captured_at, impressions, likes, reposts, replies, quotes, bookmarks |
| `notifications` | user, workspace, kind, payload (jsonb), read_at |
| `audit_events` | workspace, actor (nullable for system), action, target_type, target_id, metadata (jsonb) |

Indexes: `posts(status, scheduled_at)` for the due-post scan; `agent_runs(pipeline_run)`; `pipeline_runs(x_account, run_date)` unique; `metric_snapshots(post_item, captured_at)`; `generations(workspace, created_at)`.

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
| `GET/PATCH …/x-accounts/{id}/autopilot` (mode, guardrails, brand); `POST …/autopilot/stop` (emergency stop, all accounts) | Autopilot |
| `GET/POST/DELETE …/x-accounts/{id}/sources` | Content sources for the Research agent |
| `POST …/x-accounts/{id}/pipeline-runs` (run now); `GET …/pipeline-runs`, `GET …/pipeline-runs/{id}` (with agent runs) | Agent pipeline |
| `GET …/x-accounts/{id}/insights` | Analyst insights |
| `POST …/x-accounts/{id}/voice-profiles/from-history` | Draft a voice profile from past posts |
| `GET …/notifications`; `POST …/notifications/{id}/read` | Notifications |
| `GET …/audit-events` | Audit log (owner only) |

State transitions are implemented in one service module (`lib/posts/state.ts`); handlers never set `status` directly. An OpenAPI document is generated from the zod schemas (`zod-openapi`) for reference.

## 10. AI integration

### 10.1 Models and settings per agent
All Claude calls use the official Anthropic TypeScript SDK. Each agent's model and effort are set in config (env overrides), so cost can be tuned per agent without code changes.

| Agent | Default model | Effort | Claude features used |
|---|---|---|---|
| Research | `claude-opus-5-5` | medium | Web search tool (`web_search_20260209`, `max_uses` capped per run, optional `allowed_domains` from content sources); structured output |
| Writer | `claude-opus-5-5` | medium | Structured output (zod schema: items[], image_brief) |
| Reviewer | `claude-opus-5-5` | high | Structured output (scores, pass, notes); image input for image review |
| Image Creator (prompting) | `claude-opus-5-5` | low | Structured output (template vs AI, prompt, alt text) |
| Analyst | `claude-opus-5-5` | high | Structured output (insights) |

- **Opus 5.5 everywhere by default** for best quality. To cut cost, the Writer and Image Creator can be switched to `claude-sonnet-5-5` in config; measure quality first (Reviewer pass rate is a good signal).
- Thinking cannot be disabled on Opus 5.5; the effort level above is set explicitly on every call (its default is medium).
- **Refusals:** every call checks `stop_reason` before reading content and enables server-side fallback (`fallbacks: "default"`), so a declined request is retried on another model automatically. A step that still fails is retried by QStash, then marked failed.
- **Prompt caching:** the stable part of each prompt (agent instructions, voice profile, insights) comes first and is cached; the per-item part (topic, sources, draft) comes last.
- **Structured outputs:** every agent returns JSON validated against its zod schema (`messages.parse`), so the pipeline never parses free text.

### 10.2 Image generation
- **Template cards** (`@vercel/og`): quote cards, stat cards, list cards in the account's brand colours. Free and reliable for text.
- **AI illustrations:** an image-generation API **[DECISION — pick in Phase 0]** via the Vercel AI SDK `generateImage` (supports several providers, so it can be swapped). Typical cost ~$0.02–0.08 per image.
- Images are posted through X's media upload endpoint (`media.write` scope) and attached to the first tweet (or to chosen thread items). X API charges for media upload, if any, are tracked like other X costs.

### 10.3 Safety and quality
- **URL inputs and sources:** fetched server-side with timeout and size cap, main text extracted (`@mozilla/readability` + `linkedom`), private/internal IPs blocked (SSRF protection). Content from web pages is treated as data, never as instructions to the agents (prompt-injection defence: sources go in clearly marked data blocks, and agents have no tools that can change settings or publish).
- **Post-processing:** weighted-length check (`twitter-text`), dedupe against the account's last 50 posts (exact and near-duplicate), avoid-list check.
- **Cost control:** tokens, web searches, images and cost are recorded per agent run; budgets per workspace (AI, images, X) pause autopilot when reached; per-user rate limit on manual generation (`@upstash/ratelimit`).

### 10.4 Estimated running cost (one account, 5 posts/day, rough)
| Item | Per day | Per month |
|---|---|---|
| Claude (research + writing + reviews + image prompts) | ~$0.50–1.00 | ~$15–30 |
| Web searches (~10/day) | ~$0.10 | ~$3 |
| AI images (~3/day; others are free template cards) | ~$0.10–0.25 | ~$3–8 |
| X API (5 posts, no links + 15 metric reads) | ~$0.15 | ~$5 (≈ $30 more if every post has a link) |
| Vercel, Neon, Upstash | free tiers | $0 |
| **Total** | | **~$25–45** |

Estimates only; the dashboard shows real spend (FR-32). Switching Writer and Image Creator to Sonnet roughly halves the Claude line.

## 11. Project layout

```
twitter-ai-automation/
├── app/
│   ├── (auth)/                     # login, signup, accept-invite pages
│   ├── (dashboard)/[workspace]/    # overview, compose, queue, drafts, posts/[id], analytics, accounts, settings
│   └── api/
│       ├── auth/[...all]/          # Better Auth
│       ├── v1/                     # REST route handlers
│       ├── jobs/{agent,publish,metrics,failed}/
│       └── cron/daily/
├── components/                     # shadcn/ui + app components
├── lib/
│   ├── auth.ts, permissions.ts
│   ├── db/{schema.ts,client.ts}    # Drizzle + Neon
│   ├── posts/                      # state machine, scheduling, publishing
│   ├── agents/                     # research, writer, reviewer, image, analyst: prompt + zod schema + run()
│   ├── pipeline/                   # step orchestration, pipeline_runs/agent_runs state
│   ├── autopilot/                  # guardrails, slot placement, emergency stop
│   ├── llm/                        # Anthropic client, cost accounting, caching helpers
│   ├── images/                     # template cards (@vercel/og), image API client, Blob storage
│   ├── generation/                 # validators (length, dedupe, avoid list), URL fetching
│   ├── x/                          # XClient, OAuth, rate-limit tracking
│   ├── jobs/                       # QStash enqueue + signature verification helpers
│   ├── analytics/, notifications/, audit/
│   └── crypto.ts                   # token encryption
├── drizzle/                        # generated migrations
├── tests/                          # unit, integration, e2e (Playwright)
├── vercel.json                     # the daily cron, function maxDuration
├── .env.example
└── .github/workflows/ci.yml
```

## 12. Open questions / decisions

1. **X API access** — *Resolved in part.* The owner has a 1-year **X Premium** subscription. Premium is the consumer subscription for the account (blue check, long posts); it **does not include API access**. API access is bought separately in the X Developer Console, which (since February 2026) uses **pay-per-use** credits for new developers instead of Free/Basic/Pro tiers. Approximate published rates (verify in the console before M2): ~$0.015 per post created, ~$0.20 per post that contains a link, ~$0.005 per post read. Example at 5 posts/day: ~150 posts/month ≈ $2–3 without links, or ≈ $30 if every post has a link; metrics at 3 reads per post ≈ $2–3. Action: create a developer account and project/app, add credits, and configure OAuth 2.0 (redirect URL from §15).
2. **Image-generation provider** — choose in Phase 0 (quality of illustrations vs cost). Claude does not generate images.
3. **ORM** — Drizzle (default) or Prisma.
4. **Auth library** — Better Auth (default) or Auth.js.
5. **Autopilot** — *Resolved:* fully automatic, with guardrails (§4.11) and 7 days of shadow mode first (recommended).
6. **Preview deployments and X** — confirm a separate dev X app for previews.
7. **Uploading your own images** — v1.1 (AI images are in v1).
8. **Commercial use** — Vercel Hobby is for personal, non-commercial projects. If this will be used for clients or a business, budget for Vercel Pro (no code changes needed).

## 13. Dashboard screens

1. **Login / signup / accept invite**
2. **Overview** — autopilot status per account (with emergency stop), today's pipeline progress, upcoming posts, pending approvals, failed posts, account health, key metrics, spend
3. **Compose / Generate** — choose account + voice, input topic/URL/prompt, view candidates, edit inline with live character count, save as draft / submit / schedule
4. **Queue / Calendar** — week/month calendar and list view of scheduled posts; drag to reschedule
5. **Drafts & Approvals** — filterable list; approve/reject
6. **Post detail** — items, revisions, status history, metrics
7. **Analytics** — date range, per account, charts and top posts
8. **Accounts** — connect/disconnect X, voice profiles, posting slots, content sources, brand (colours, logo, image style), autopilot mode and guardrails
10. **Agents** — pipeline runs per day; per post: research brief, drafts, review scores and notes, images, cost per agent; Analyst insights
9. **Settings** — workspace, members & roles, timezone, AI budget

## 14. Testing strategy

- **Unit (Vitest):** post state machine, length validation, prompt building, rate-limit logic, token encryption, permission checks.
- **Integration (Vitest):** route handlers against a real Postgres (Neon branch in CI, or a Postgres service container), with X, Anthropic, and QStash mocked via MSW. Job handlers are called directly with signed test payloads.
- **Agent tests:** each agent's prompt builder and schema validated with recorded Claude responses; Reviewer rejects known-bad samples (off-voice, unsupported claim, blocked topic); pipeline step retry does not duplicate drafts; guardrails (daily cap, gap, hours, budget) enforced; emergency stop cancels scheduled posts.
- **Quality check (manual, before live autopilot):** shadow-mode week reviewed by the owner.
- **Critical-path tests:** no duplicate publish when two publish jobs run at once; thread partial failure resumes correctly; token refresh on 401; a stale publish message after a reschedule does nothing; the daily cron running twice.
- **E2E (Playwright):** login → generate → approve → schedule against a preview deployment, with a mocked X.
- **CI (GitHub Actions):** typecheck (`tsc`), ESLint, Vitest, `drizzle-kit check`, `next build` on every PR. Vercel creates the preview deployment.

## 15. Configuration (env)

Set in Vercel project settings (Marketplace integrations add the database ones automatically):

`DATABASE_URL` (Neon, pooled), `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `TOKEN_ENCRYPTION_KEY`, `X_CLIENT_ID`, `X_CLIENT_SECRET`, `X_REDIRECT_URI`, `ANTHROPIC_API_KEY`, `AGENT_MODEL_RESEARCH`, `AGENT_MODEL_WRITER`, `AGENT_MODEL_REVIEWER`, `AGENT_MODEL_IMAGE`, `AGENT_MODEL_ANALYST` (all optional; default `claude-opus-5-5`), `IMAGE_PROVIDER`, `IMAGE_API_KEY`, `BLOB_READ_WRITE_TOKEN`, `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `QSTASH_TOKEN`, `QSTASH_CURRENT_SIGNING_KEY`, `QSTASH_NEXT_SIGNING_KEY`, `CRON_SECRET`, `RESEND_API_KEY`, `APP_URL`, `SENTRY_DSN` (optional).

## 16. Milestones

The step-by-step build order (Phases 0–11, mapped to these milestones) is in [PLAN.md](PLAN.md).

| # | Milestone | Scope | Done when |
|---|---|---|---|
| M0 | Foundation | Project, Neon, Upstash, CI, daily cron, delayed QStash job | Deployed; delayed job fires on time |
| M1 | Auth & workspaces | FR-1 – FR-3 | Sign up, workspace, invite |
| M2 | X connection | FR-4 – FR-7, FR-9a | Real X account connected; voice profile drafted from history |
| M3 | Writer + Reviewer agents | FR-8 – FR-16, FR-37, FR-38, compose UI | Manual generate → reviewed drafts |
| M4 | Workflow & scheduling | FR-17 – FR-28 | Approved posts publish on time, once |
| M5 | Research agent + daily pipeline | FR-36, §6.2, content sources, Agents page (FR-41) | Daily drafts appear in the approval queue |
| M6 | Image Creator agent | FR-39, media upload | Posts publish with reviewed images |
| M7 | Analytics + Analyst agent | FR-29 – FR-32, FR-40 | Metrics charted; weekly insights feed agents |
| M8 | Autopilot | FR-42 – FR-47 | Shadow week reviewed; live autopilot posting within guardrails |
| M9 | Notifications, audit, hardening | FR-33, FR-35, security review, docs | Production-ready v1 |

v1.1 backlog: uploading your own images, draft comments, email notifications and digest, best-time-to-post experiments, more agent types.
