# Implementation Plan

Companion to [spec.md](spec.md) (v0.4). The spec says **what** we build; this file says **in what order**.

Principles:
- **Every phase ends deployed and working on Vercel.** No phase leaves the app broken.
- **Risky parts early.** Phase 1 proves that the free-tier setup (Vercel Hobby + QStash delayed jobs + Neon) works before features are built on it.
- **First usable version after Phase 6:** you can generate, approve, schedule and auto-publish posts. Phases 7–9 add analytics, notifications and polish.
- Each phase is one branch and one pull request, with tests, reviewed before merging to `main`.

Legend: 🧑 = something **you** need to do (accounts, keys, approvals). Everything else I implement.

---

## Phase 0 — Accounts and prerequisites

**Goal:** have every service account ready so building isn't blocked later.

| Who | Task | Needed by |
|---|---|---|
| 🧑 | Create a free **Vercel** account (Hobby) and connect it to the GitHub repo | Phase 1 |
| 🧑 | Create a free **Neon** account (or add Neon from the Vercel Marketplace — easier) | Phase 1 |
| 🧑 | Create a free **Upstash** account; create one Redis database and enable QStash (also available from the Vercel Marketplace) | Phase 1 |
| 🧑 | Create an **Anthropic** API account, add a small amount of credit, create an API key | Phase 4 |
| 🧑 | Create an **X developer** account, a Project and an App; add pay-per-use credits; turn on OAuth 2.0 (I'll give you the callback URL) | Phase 3 |
| 🧑 | Optional: a free **Resend** account for emails (password reset, invites) | Phase 2 |

**Done when:** all keys are saved in Vercel → Project → Settings → Environment Variables (I'll give you the exact list).

---

## Phase 1 — Foundation (spec M0)

**Goal:** an empty but real app, deployed, with the background-job machinery proven.

Tasks:
1. Create the Next.js (App Router) + TypeScript project; add Tailwind CSS and shadcn/ui.
2. Set up ESLint, Prettier, Vitest and strict TypeScript.
3. Add Drizzle ORM connected to Neon; first migration; run migrations during the Vercel build.
4. Add Upstash Redis and QStash clients, with QStash signature checking on `/api/jobs/*`.
5. Add `vercel.json` with the single daily cron (`/api/cron/daily`), protected by `CRON_SECRET`.
6. Add a `/api/health` endpoint that checks the database and Redis.
7. **Proof of concept:** a test job that QStash calls 2 minutes after it is enqueued, and writes a row to the database.
8. GitHub Actions CI: typecheck, lint, tests, build.
9. `.env.example` and a short README "how to run locally".

**Done when:** the app is live at `<name>.vercel.app`, `/api/health` is green, the delayed test job fires on time in production, and CI passes.

---

## Phase 2 — Login and workspaces (spec M1)

**Goal:** only you (and people you invite) can use the app.

Tasks:
1. Better Auth: sign up, log in, log out, password reset.
2. Workspaces with roles (Owner, Editor, Contributor, Viewer) and invitations.
3. One permission helper used by every API route (workspace scoping).
4. App layout: sidebar, workspace switcher, empty pages for each section.
5. Tests for the permission rules.

**Done when:** you can sign up, see your workspace, invite a second email, and that person sees only what their role allows.

> If this is only for you, roles and invitations can be kept minimal here and expanded later — tell me and I'll shorten this phase.

---

## Phase 3 — Connect X accounts (spec M2)

**Goal:** the app can act on your X account safely.

Tasks:
1. OAuth 2.0 (PKCE) "Connect X account" flow and callback.
2. Encrypt access/refresh tokens before saving (AES-256-GCM).
3. `XClient` wrapper: on-demand token refresh with a Redis lock, rate-limit tracking from X response headers, cost tracking per call.
4. Accounts page: connected accounts, status (active / needs re-auth), disconnect.
5. Check whether long posts work through the API for your Premium account; set `max_post_length` accordingly.
6. Tests with a mocked X API.

**Done when:** you connect your real X account, see your username and avatar, and can disconnect and reconnect it. (No posting yet.)

🧑 Needs: X developer app from Phase 0.

---

## Phase 4 — Voice profiles and AI generation (spec M3)

**Goal:** the AI writes drafts that sound like you.

Tasks:
1. Voice profile editor: tone, audience, topics, topics to avoid, example tweets, hashtag/emoji rules.
2. `LLMClient` (Vercel AI SDK + Claude) with structured output (zod schemas).
3. Generation job (`/api/jobs/generate`): build the prompt, call Claude, validate length, check duplicates and the avoid list, save candidates, record tokens and cost.
4. Link input: fetch the article safely (timeouts, size cap, block internal addresses) and extract its text.
5. Compose page: pick account and voice, enter topic / link / prompt, choose tweet or thread, see candidates, edit with a live character counter, save as draft.
6. Rewrite actions (shorter, punchier, more formal, translate…).
7. Monthly AI budget check and per-user generation rate limit.

**Done when:** you type a topic, get several good drafts in your voice within ~30 seconds, edit one, and save it as a draft.

🧑 Needs: Anthropic API key. You'll also fill in your first voice profile and judge the draft quality — we may tune prompts here.

---

## Phase 5 — Drafts and approval workflow (spec M4, part 1)

**Goal:** a clear lifecycle for every post.

Tasks:
1. Post state machine in one module: draft → pending approval → approved → scheduled → publishing → published (+ failed, cancelled), with role checks on each step.
2. Revision history for every edit.
3. Drafts & Approvals page: filter by status/account, approve, reject, send back to draft.
4. Post detail page: text, history, status timeline.
5. Unit tests for every allowed and forbidden transition.

**Done when:** a draft can be submitted, approved, sent back, edited, and every change is recorded.

---

## Phase 6 — Scheduling and auto-publishing (spec M4, part 2) ⭐ first usable version

**Goal:** approved posts go out on X at the right time, exactly once.

Tasks:
1. Schedule a post: pick a date/time (in your timezone) or "next free slot"; posting-slot settings per account.
2. On schedule: create a delayed QStash message for posts within 7 days; save its ID. On reschedule/cancel: delete it and, if needed, create a new one.
3. Publish job: check the message still matches the post, atomically claim the post, publish the tweet or thread, save each tweet ID immediately, resume threads after partial failure.
4. Retries: X busy/down → QStash retries with backoff; login revoked → mark account "needs re-auth", fail the post. Failure callback marks posts failed.
5. Daily cron: set timers for posts now within 7 days, fix stuck posts, publish any missed posts, keep X tokens fresh.
6. "Publish now" and "Retry" buttons.
7. Queue page: calendar (week/month) and list views; drag to reschedule.
8. Overview page: upcoming posts, pending approvals, failed posts.
9. Tests: two publish jobs at once never post twice; old timer after reschedule does nothing; thread resumes correctly; daily cron safe to run twice.

**Done when:** you schedule a real post 5 minutes ahead and a thread for tomorrow; both appear on X on time, once each; cancelling a scheduled post stops it.

🧑 Needs: X API credits (each real post costs ~$0.015, or ~$0.20 with a link).

---

## Phase 7 — Analytics and cost tracking (spec M5, part 1)

**Goal:** see how posts perform and what the app costs.

Tasks:
1. On publish, enqueue three delayed metric jobs (+1 h, +24 h, +7 days).
2. Metrics job: read likes, reposts, replies, quotes, bookmarks and views from X; store a snapshot.
3. Analytics page: date range, per account totals, engagement chart, top posts.
4. Cost panel: AI spend and X API spend this month; optional monthly X budget that blocks new scheduling when exceeded.

**Done when:** a post published in Phase 6 shows its numbers after 1 hour, and the cost panel matches the X and Anthropic consoles roughly.

---

## Phase 8 — Notifications and audit log (spec M5, part 2)

**Goal:** you find out when something needs attention.

Tasks:
1. In-app notifications: post failed, account needs re-auth, post awaiting approval; unread badge.
2. Audit log of important actions (connect/disconnect, approve, schedule, publish, delete, role changes); owner-only page.

**Done when:** a forced failure (e.g. a revoked X login) shows a notification within one job run, and the audit log lists your recent actions.

---

## Phase 9 — Hardening and launch (spec M6)

**Goal:** safe and reliable for everyday use.

Tasks:
1. Security review: permission checks on every route, secrets never logged, encryption, link-fetching protections, input validation.
2. End-to-end test (Playwright): login → generate → approve → schedule, with X mocked.
3. Error monitoring (Sentry, optional, free tier).
4. Check free-tier usage (Vercel, QStash, Neon, Upstash) against limits.
5. Docs: setup guide, environment variables, "what to do when…" runbook (X login revoked, post failed, budget reached).

**Done when:** all checks pass and you've used it for real for a few days without manual fixes.

---

## Later (v1.1 backlog)

- Image/video upload (Vercel Blob)
- Comments on drafts
- Email notifications (Resend)
- Auto-approve for trusted accounts (only if you decide to allow it)
- Best-time-to-post suggestions from your analytics

---

## Summary

| Phase | What you get | Your action needed |
|---|---|---|
| 0 | Service accounts ready | Create accounts and keys |
| 1 | Live empty app, background jobs proven | Vercel, Neon, Upstash |
| 2 | Login, workspaces, roles | — |
| 3 | X account connected | X developer app |
| 4 | AI drafts in your voice | Anthropic key, voice profile |
| 5 | Draft → approve workflow | — |
| 6 | **Scheduled auto-publishing (first usable version)** | X API credits |
| 7 | Analytics and costs | — |
| 8 | Notifications and audit log | — |
| 9 | Hardened, documented v1 | Real-world trial |
