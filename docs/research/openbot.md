# OpenBot (CopilotKit) as a second bot provider

> **Correction 2026-11-04 (evening):** this note studied the WRONG OpenBot.
> The one Oscar runs on `oscar-dev-server-leader` is **nightly-labs/openbot**
> (v0.60, PolyForm Noncommercial, Synthetify Labs), not CopilotKit/openbot.
> nightly-labs/openbot is a local-first desktop/web/phone workspace for
> persistent agents whose brains are ACP agents; on the leader it is wired to
> `dsh --profile acp` through `~/.local/bin/dsh-cpa-acp`, so every OpenBot
> agent there is a DSH session. Oscar's verdict: OpenBot only solves "DSH has
> no remote access"; it is a remote window onto DSH, **not** a base for
> ProjectX and not a product direction. Decision #2 ("OpenBot later as a bot
> provider") is therefore dropped; see decisions in `docs/projectx.html`.
> The rest of this file is kept for the record only.

Date: 2026-11-04. Source: clone of https://github.com/CopilotKit/openbot at commit `f3dfc9b` (2026-11-03), README, `docs/`, `server/src`. Read-only; nothing was run.

## 1. What it is
- A **self-hosted agent platform template**, not a hosted product: "A template, not a product… no hosted version… nothing published as a package" (README). You clone it, replace `examples/fintech` with your own bots, and run it.
- Shape: React app (3010) + **Bun/Hono API server** (3001) + PostgreSQL/pgvector + a **routine worker** + a **supervisor** that gives each person/bot a Docker "computer" (Chromium + `/workspace` + shell, port 4100).
- A Bot = any **AG-UI endpoint** (LangGraph, Mastra, CrewAI, Pydantic AI, hand-written…) or a `built-in` bot with a system prompt. OpenBot supplies governance (CEL policy gateway, audit, approvals), the browser/computer, routines and channels; **you supply the brain** (model key + optional agent endpoint).
- Durable threads/memory live in **CopilotKit Intelligence** — a hard dependency. Server refuses to start without `INTELLIGENCE_API_URL/_GATEWAY_WS_URL/_API_KEY`. Managed cloud (free Developer plan) or self-hosted with a licence (local self-host is a macOS/Docker-Desktop preview).
- **No CLI** for driving bots. `scripts/` only has start/stop/setup helpers. The HTTP API exists but is built for the browser session (Better Auth cookies; Google/Microsoft/Okta/SAML). The only no-login mode is `OPENBOT_SINGLE_USER=true`, which **refuses to start on a non-loopback address**. No personal-access/API token for machine callers (only SCIM and Recorder bearers).
- Useful HTTP routes an orchestrator could use (all under `/api`, all `requireUser`):
  - `POST /delivery/conversations/:id/messages` `{agentId, text}` → **202 queued** (async; no reply in response); `GET /delivery/conversations/:id/history` to read back.
  - `POST /copilotkit/*` AG-UI run route — returns at gateway-join `{threadId, runId, joinToken}`, "a caller learns that a turn STARTED and nothing else" (`toolkit/src/run-turn.ts`); the reply arrives over a WebSocket.
  - `POST /responsibilities` (lasting goal), `POST /responsibilities/:id/run`, `GET /responsibilities/:id/runs` → run status `succeeded|waiting|failed|skipped` with `replyText` — the cleanest "send task → poll → result" path.
  - `GET/POST /channels`, `GET /tasks`, `GET /approvals`, `POST /approvals/:id/decision`, `POST /approvals/questions/:id/respond`, `POST /routines/:id/run`, `GET /routines/:id/runs`.
  - `POST /events/triggers/:triggerId` — secret-authenticated public webhook ingress (webhook/GitHub/Linear/Sentry/PagerDuty/email/Slack).
  - No cancel/interrupt route found for a running turn.

## 2. Grok-Bot-like concepts (mapping)
| Grok Bot | OpenBot | Notes |
|---|---|---|
| Named persistent bot + memory | ✓ `agents`/`agent_profiles`, `/bots/:id/memory`, Learning containers | memory needs Intelligence |
| Routines (cron) | ✓ `/routines`, created by asking the bot; 15-min floor, 20 cap, worker process | plus **Responsibilities** = goal + triggers |
| Webhook/email/GitHub triggers | ✓ `responsibilitySources`: slack, github, webhook, linear, sentry, pagerduty, email | |
| Connectors | Google Drive, Notion, Parallel/Exa/Firecrawl; **Composio** (hundreds of apps, incl. X/Twitter) and Arcade | **No first-class PostHog, RevenueCat, Postiz**; would go via Composio or a custom MCP server |
| Approvals / HITL | ✓ strong: CEL deny-before-allow policy, `/approvals`, question cards, "take the wheel" on login/2FA walls, audit trail | |
| Inbox | ✗ no equivalent. Nearest: `/tasks` (check-in items), proactive suggestions, notifications to Slack/Teams/SMS | |
| Rooms / bot-to-bot | ✓ group channels, `message_bot` handoff tool (typed, granted per bot) | |
| Shared cloud computer | ✓ one Docker computer per person, or per Isolated bot; shell + browser + files | heavier than Grok's shared Box |

## 3. Running on a Linux server
- Yes: `Dockerfile` is Ubuntu 24.04 + Bun 1.4.3 + Node 24; published image `ghcr.io/copilotkit/openbot:latest` with `EMBEDDED_POSTGRES=on`. Helm chart in `charts/`.
- **Needs Bun, not Node 22** (server uses Bun's WebSocket; README: Bun 1.4+). Needs Docker for Postgres and bot computers, a model key (OpenAI/Anthropic/Google), `KEY_ENCRYPTION_KEY`, and the CopilotKit Intelligence project key.
- Sizing (docs/deployment.md): 2 GB RAM minimum, 4 GB for several bots; ~1 GB per concurrent browser.
- Driving from another process: single-user mode only on loopback, so the orchestrator must run on the same host (or use a real identity provider and a cookie session).

## 4. Licence, maturity, team
- MIT © CopilotKit. 10,069 stars, 1,241 forks, 69 open issues; created 2026-08-17; last push 2026-11-04; v0.2.0; README badge **Beta**, "expect rough edges… expect things to move". 769 test files, 430 KB changelog — very fast churn (27 commits on 2026-11-02 alone). Main committer David McKay (CopilotKit). Commercial angle: "we can build this for you" services.

## 5. Verdict for ProjectX
**Later, not now.** It is the closest open-source analogue to Grok Bot (bots + computer + routines + approvals), but:
1. No CLI and no machine token: the adapter would need a loopback single-user deployment or a scripted cookie login.
2. Send is fire-and-forget (202 / join); `wait`/`tail` need the channel WebSocket or polling `responsibilities/:id/runs`.
3. Hard dependency on CopilotKit Intelligence (cloud or licensed self-host) — a second vendor in the loop.
4. Heavy footprint (Docker, Postgres, Chromium per bot) and beta churn; API paths may move between weeks.
5. OpenBot gives no brain: we would also have to host an AG-UI agent (or use `built-in` prompt bots on OpenAI).

What the adapter would do (when revisited): `send` → `POST /api/responsibilities` (or `/delivery/conversations/:id/messages` for an existing thread); `wait` → poll `GET /responsibilities/:id/runs` until `succeeded|failed|waiting`; `tail` → subscribe to the channel WebSocket or read `conversations/:id/history`; `interrupt` → not available via HTTP today (pause the responsibility instead). Approvals map 1:1 to our Inbox via `GET /approvals` + `POST /approvals/:id/decision`.

Recommendation: keep the bot-provider interface engine-neutral, re-check OpenBot when it ships (a) a stable public API/token and (b) a v1.0; meanwhile Grok Bot stays the only real-tool provider.
