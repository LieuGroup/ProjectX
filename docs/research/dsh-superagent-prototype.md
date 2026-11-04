# dsh-superagent-prototype — what it is, and whether ProjectX should use it

Source: `git clone https://github.com/deepseek-ai/dsh-superagent-prototype` → `/tmp/dsh-superagent-prototype` (commit `8c32a9d`, 2026-10-31), plus the article [Exploring Super-agent Design](https://x.com/DeepSeekHarness/status/2116857257245987236) (X truncates it; full mirror at [offloop.org](https://offloop.org/learn/exploring-deepseek-super-agent-design)). Paths below are relative to the clone.

**In one line:** a MIT-licensed, ~15k-LOC DSH *bundle* that turns one DeepSeek Harness profile into a proactive coding agent for **one GitHub repo** (+ optional Feishu chat). A "main Session" (Router) judges incoming events and opens Tickets; each Ticket gets its own worker Session + git worktree. It is explicitly "research code, not production" (`README.md:3-5`).

## 1. What it implements (vs. the article's design)

| Concept | Status | Evidence |
| --- | --- | --- |
| Router / Worker roles | **Exists** | Two agent presets `dsh-superagent-router` / `-worker` in `cordis.patch.yml:20-193`; `ROUTER_PRESET`/`WORKER_PRESET` in `src/app/sessions.ts:48`. Router has read-only fs, no subagents/web; Worker has full work tools. |
| Threads + subscriptions | **Exists** | Domain in `docs/001-domain.md`; `src/core/intake.ts`, `src/core/tickets.ts` (fan-out of Triggers to subscribed Tickets, many-to-many). |
| Triggers (events) + policy gate | **Exists** | `src/core/engine.ts` `receive(trigger)`; deterministic gate + LLM assessment in `src/policy.ts`, `src/app/assessment.ts`. Sources: GitHub polling (`src/sources/github.ts`), Feishu (`src/sources/feishu*.ts`). No webhooks — polls every N seconds. |
| Inline-vs-spill (long commands go to background) | **Exists (via config)** | Worker/Router shell `timeoutMs: 5000` so foreground commands spill to background jobs after 5 s (`cordis.patch.yml:38-56`, `:69-74`); job end wakes the worker. Router is guarded against `sleep`/long waits (`src/app/sessions.ts:70-86`). |
| Watchdog | **Partial** | In-process only: 30-min background limit kills runaway jobs (`src/executor/dsh.ts:32,634-656`); transient-error retries w/ backoff (`:548-576`); health board (`src/infra/health.ts`); restart recovery (`resume()` `:524-537`). The article's *external* watchdog is just a `scripts/dsh-superagent status/restart` CLI + a suggested hourly DSH schedule in another profile (`docs/006-deployment.md:135-141`). No stuck-detection of a worker that is "alive but not progressing". |
| Inbox (human attention queue) | **Partial** | Ticket stages `person/answer`, `person/review`, `blocked`, `failed` (`docs/003-execution.md:9-18`, `src/core/stage.ts`) and an "Overview: Tickets that need a person" panel view (`src/app/views/overview.ts`). It is a list, not a unified inbox; the human answers *in GitHub/Feishu*, not in the panel. It also uses DSH's own agent *inbox* (`agent.steer/followup/inject`) as the delivery mechanism, which is a different thing. |
| Durable state / effects queue | **Exists** | SQLite via `node:sqlite` (`src/app/store.ts:60`), durable effect queue with retry (`src/app/effects.ts`), 2-s pass loop (`src/app/passes.ts:28`). |
| Routines (cron-style recurring work) | **Missing** | Nothing scheduled by the plugin itself; it is purely event-driven. |

## 2. Packaging & DSH surface

- **Bundle** with `dsh.bundle.patch: cordis.patch.yml` and a `./client` web export (`package.json:30-46`). Pins `dsh ^0.2.1-alpha.2`, cordis `4.0.5-alpha.1`. 23 peer deps on `@deepseek-ai/dsh-*`.
- **Host services injected** (`src/index.ts:8`): `agents`, `sessionController`, `sessionPersistence`, `workspaceRegistry`, `permissionPresets`, `tools`, `systemPrompt`, `skills`, `webServer`, `llm`, `subprocess`; plus `settings`, `agentPresets`, `jobs`, `subagents` via `ctx.inject` (`src/executor/background.ts:2`).
- **Not used:** `ctx.storage` (own SQLite instead), `ctx.agentTeams`, `schedule`. Subagents are only *observed/stopped*, never spawned by the plugin (workers may spawn them via the `subagent` tool).
- **UI: yes.** React panel (~4.4k LOC in `src/client/`) mounted with `ctx.slots.inject('main'|'sidebar.panellist'|'plugins.row.config')` (`src/client/index.tsx:53-56`): overview, tickets, threads, sources, settings, prompts editor, zh/en. Served from `/orchestrator/api` on the profile's web server (`src/app/http.ts`).
- Needs a **dedicated DSH profile** (changes process-wide PATH/git config) and runs workers with `danger-full-access` (`src/app/sessions.ts:152`).

## 3. How a Worker starts and how the Router is woken

1. Router accepts → `Engine.decide` creates Ticket + Assignment → `DshExecutor.start` enqueues a durable `launch` effect (`src/executor/dsh.ts:264-266`).
2. `launch` handler: reserve a worker slot (`maxWorkers`, default 3, +1 for human messages), run the `prepare` shell command (git worktree), `sessions.ensure()` → `sessionController.create({ agentPreset: 'dsh-superagent-worker' })` (`src/app/sessions.ts:138-155`), then push the assignment JSON into the worker via `agent.followup()` (`src/app/delivery.ts:66-85`, idempotent by message id).
3. Later inputs (comments, CI) are **steered** into the running worker (`agent.steer`).
4. **Wake-up of the Router** is not polling: the worker calls the `orchestrator_report` tool (`src/app/tools/executor.ts:74-103`) → `Engine.report` + a `relay` effect → `Sessions.prompt(masterId, …, 'steer')` into the Router Session (`src/app/sessions.ts:104`). Additionally, the executor listens to `agent/status` → `idle` (`src/executor/dsh.ts:479-487`), waits for background jobs/subagents to go quiet (`untilQuiet`), then files a `waiting` report + relay (`:601-672`). Low-priority facts are `inject`ed without waking the Router (`notice` effect).

## 4. Licence, activity, size

| | |
| --- | --- |
| Licence | MIT (`LICENSE`, `package.json:4`) |
| Commits | **1** squash commit, 2026-10-31 ("add the dsh-superagent research prototype"); GitHub `pushed_at` 2026-11-03 |
| Popularity | ~17.8k stars / 1.9k forks / 0 open issues after 4 days (issues likely disabled) |
| Size | `src/` 15.2k LOC (core 2.1k, client 4.4k, executor 0.8k, rest app/sources); `tests/` 15.0k LOC, 59 vitest specs; docs+prompts 2.3k lines |
| Scope | One GitHub repo per deployment; Sources hard-wired to GitHub/Feishu in several generic modules (`docs/007-architecture.md:275-286`) |

## 5. Verdict: **(b) copy its patterns, do not depend on it**

| Option | Why / why not |
| --- | --- |
| (a) Depend on it | ❌ `private: true`, not on npm, single squash commit, no roadmap, pinned to alpha DSH versions, assumes GitHub+Feishu and a coding worker. Our loop is marketing-heavy (ASO, Ads, TikTok) and multi-product; we'd be fighting its domain model (Ticket≠Issue, no Routines, no Inbox-as-UI). |
| **(b) Copy patterns** | ✅ It answers, with working code and tests, the exact DSH questions we had: how to create role-preset Sessions, how to steer/followup/inject messages idempotently, how to detect "worker idle" incl. background jobs, 5-s spill config, durable effects queue, slot-based UI panel. MIT lets us lift files with attribution. |
| (c) Ignore | ❌ Would mean re-discovering the DSH event/API surface ourselves. |

**What to lift (in order):** `src/app/delivery.ts` (idempotent delivery), `src/executor/background.ts` + `dsh.ts` `install()/workerIdle()` (wake-up logic), the two presets in `cordis.patch.yml`, `src/app/effects.ts` (durable queue), `src/client/index.tsx` slot registration. **What to build ourselves:** Captain/Bot/Issue/Thread model, Routines (use DSH `schedule`), a real Inbox page, bot-provider interface (grokbot CLI), approval gates for money/publishing. **Open gap in both the article and the prototype:** a true external watchdog — plan one as a tiny separate process/launchd job.
