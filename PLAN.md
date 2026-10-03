# Implementation Plan

Companion to [spec.md](spec.md) (v0.5). The spec says **what** we build; this file says **in what order**.

Principles:
- **Every phase ends deployed and working on Vercel.** No phase leaves the app broken.
- **Risky parts early.** Phase 1 proves the free-tier setup (Vercel Hobby + QStash delayed jobs + Neon) before features are built on it.
- **Agents one at a time.** Each agent is built, tested and tuned on its own before the next one joins the pipeline.
- **Autopilot last.** Fully automatic posting is switched on only after every agent works and a 7-day shadow run shows the quality is right.
- Each phase is one branch and one pull request, with tests, reviewed before merging to `main`.

Legend: 🧑 = something **you** need to do (accounts, keys, decisions, quality checks). Everything else I implement.

## The five agents at a glance

| Agent | Duty | Built in |
|---|---|---|
| ✍️ **Writer** | Writes tweets and threads in your voice | Phase 4 |
| 🧐 **Reviewer / Editor** | Scores every draft (voice, hook, facts, safety); sends weak ones back for rewrite | Phase 4 |
| 🔎 **Research** | Finds daily topics from your sources and the web, with facts and links | Phase 6 |
| 🎨 **Image Creator** | Makes a visual for each reviewed draft (AI image or branded card) | Phase 7 |
| 📊 **Analyst** | Learns what performs best and feeds it back to Research and Writer | Phase 8 |

Pipeline: Research → Writer → Reviewer → Image Creator → Reviewer (image) → schedule → publish → metrics → Analyst → (back to Research/Writer).

---

## Phase 0 — Accounts and prerequisites

| Who | Task | Needed by |
|---|---|---|
| 🧑 | Free **Vercel** account (Hobby), connected to the GitHub repo | Phase 1 |
| 🧑 | Free **Neon** and **Upstash** (Redis + QStash) — easiest via the Vercel Marketplace | Phase 1 |
| 🧑 | **X developer** account: Project + App, pay-per-use credits, OAuth 2.0 on (I'll give the callback URL) | Phase 3 |
| 🧑 | **Anthropic** API key with some credit | Phase 4 |
| 🧑 | Choose an **image-generation provider** and get its API key (I'll give a short comparison with sample images) | Phase 7 |
| 🧑 | Optional: **Resend** for emails | Phase 2 |

---

## Phase 1 — Foundation (spec M0)

1. Next.js + TypeScript + Tailwind + shadcn/ui; ESLint, Prettier, Vitest.
2. Drizzle ORM + Neon; migrations run during the Vercel build.
3. Upstash Redis and QStash clients; QStash signature checks on `/api/jobs/*`.
4. `vercel.json` with the daily cron, protected by `CRON_SECRET`; `/api/health`.
5. **Proof:** a delayed QStash job that fires 2 minutes later in production and writes to the database.
6. GitHub Actions CI; `.env.example`; README "run locally".

**Done when:** live on `<name>.vercel.app`, health green, delayed job fires on time, CI passes.

---

## Phase 2 — Login and workspaces (spec M1)

1. Better Auth: sign up, log in, password reset.
2. Workspaces, roles, invitations; one permission helper used by every route.
3. App layout with sidebar and empty section pages.

**Done when:** you can sign up and see your workspace. *(If it's only you, I'll keep roles/invites minimal.)*

---

## Phase 3 — Connect X and voice profile (spec M2)

1. OAuth 2.0 "Connect X account" (including `media.write` for images later); tokens encrypted.
2. `XClient`: on-demand token refresh, rate-limit tracking, cost tracking.
3. **Voice profile from your history:** read your recent posts and let Claude draft the profile; you review and edit it.
4. Check long-post support for your Premium account.
5. Accounts page.

**Done when:** your real X account is connected and has a voice profile drafted from your own tweets.

🧑 X developer app. Review the drafted voice profile.

---

## Phase 4 — Writer ✍️ and Reviewer 🧐 agents (spec M3)

1. Agent framework: one module per agent (prompt, zod output schema, model + effort config), `agent_runs` logging with tokens and cost, Anthropic SDK client with prompt caching and refusal fallback.
2. **Writer agent:** tweet or thread from a topic, link or prompt; follows the voice profile and length rules; also writes an image brief.
3. **Reviewer agent:** scores voice, hook, clarity, factual support and safety; pass/fail; edit notes; up to 2 rewrite loops with the Writer.
4. Compose page: enter topic/link/prompt → Writer → Reviewer → drafts with scores and notes; edit with live character counter; save.
5. Budget check and rate limit.
6. Tests with recorded Claude responses; Reviewer must reject a set of known-bad samples.

**Done when:** you type a topic and get reviewed drafts in your voice; weak drafts are visibly rewritten.

🧑 Anthropic key. Judge draft quality — we tune prompts here.

---

## Phase 5 — Approval workflow, scheduling and publishing (spec M4) ⭐ first usable version

1. Post state machine with history; Drafts & Approvals page.
2. Posting slots per account; schedule / next free slot.
3. Delayed QStash publish job, atomic claim (never posts twice), thread resume, retries, failure handling.
4. Daily cron: future timers, stuck posts, missed posts, token freshness.
5. Queue calendar, Overview page, Publish now / Retry.

**Done when:** a post scheduled 5 minutes ahead and a thread for tomorrow both appear on X on time, once each.

🧑 X API credits.

---

## Phase 6 — Research agent 🔎 and the daily pipeline (spec M5)

1. Content sources per account: RSS feeds, websites, keywords; niche description.
2. **Research agent:** Claude web search (capped per run) + your sources → daily content brief; every fact has a source link; skips recent and blocked topics.
3. **Pipeline engine:** `pipeline_runs` + one QStash job per step; retries don't duplicate work; the daily cron starts tomorrow's run for each account.
4. Research → Writer → Reviewer, end to end; results land in the **approval queue** (no autopilot yet).
5. **Agents page:** per day and per post — brief, drafts, scores, notes, cost per agent. "Run now" button.

**Done when:** each morning the approval queue holds reviewed drafts on fresh topics, with sources you can check.

🧑 Add your content sources; review a few days of output.

---

## Phase 7 — Image Creator agent 🎨 (spec M6)

1. Brand settings: colours, logo, image style.
2. Template cards (`@vercel/og`): quote, stat, list.
3. AI illustrations via the chosen provider; images stored in Vercel Blob; alt text.
4. **Image Creator agent:** picks template vs AI image, writes the prompt, generates.
5. **Reviewer image check:** matches the post, no garbled text, no real people or logos, safe; one regenerate, else text-only.
6. X media upload; image attached to the post; Blob cleanup after 30 days.

**Done when:** posts publish on X with matching, reviewed images.

🧑 Image provider key; set brand colours/logo; judge image quality.

---

## Phase 8 — Analytics and Analyst agent 📊 (spec M7)

1. Metrics jobs at +1 h, +24 h, +7 days; analytics page; cost panel (AI, images, X).
2. **Analyst agent** (weekly): best topics, hooks, formats, times → insights saved and fed into Research and Writer prompts; suggested posting slots; suggested voice-profile tweaks (you approve those).

**Done when:** analytics show real numbers and the first weekly insights visibly change what the agents produce.

---

## Phase 9 — Autopilot (spec M8) 🚀 fully automatic

1. Per-account autopilot mode: **off / shadow / live**.
2. Guardrails: posts per day, minimum gap, allowed hours, score threshold, blocked categories, no posts on very fresh news, monthly budgets.
3. System approval + automatic slot placement for Reviewer-passed posts.
4. **Emergency stop** (all accounts / one account): pause and cancel the next 24 hours.
5. **Shadow mode:** everything runs, posts go to the queue marked "would have posted".
6. Daily digest: posted, rejected (and why), spend.
7. Reminder to turn on X's "automated account" label.

**Done when:** a 7-day shadow run is reviewed by you and approved, then live autopilot posts within the guardrails for a week without manual fixes.

🧑 Review the shadow week; set guardrails; turn on the automated-account label in X; switch to live.

---

## Phase 10 — Notifications and audit log (spec M9, part 1)

1. In-app notifications: post failed, account needs re-auth, budget reached, autopilot paused, pending approvals.
2. Audit log including agent actions and autopilot changes.

---

## Phase 11 — Hardening (spec M9, part 2)

1. Security review: permissions, secrets, encryption, link fetching, prompt-injection defences for web content.
2. End-to-end tests; free-tier usage check; optional Sentry.
3. Docs: setup guide, environment variables, runbook (X login revoked, bad post went out, budget reached, emergency stop).

---

## Later (v1.1)

Uploading your own images · comments on drafts · email notifications and digest · posting-time experiments · more agent types.

---

## Summary

| Phase | What you get | Your action |
|---|---|---|
| 0 | Accounts ready | Create accounts and keys |
| 1 | Live empty app, background jobs proven | Vercel, Neon, Upstash |
| 2 | Login, workspace | — |
| 3 | X connected, voice profile from your tweets | X developer app; review profile |
| 4 | ✍️ Writer + 🧐 Reviewer agents | Anthropic key; judge drafts |
| 5 | ⭐ Approve → schedule → auto-publish | X API credits |
| 6 | 🔎 Research agent + daily pipeline | Content sources |
| 7 | 🎨 Image Creator agent | Image provider; brand |
| 8 | 📊 Analytics + Analyst agent | — |
| 9 | 🚀 Autopilot (shadow → live) | Review shadow week; go live |
| 10 | Notifications, audit log | — |
| 11 | Hardened, documented v1 | — |
