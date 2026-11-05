# Hermes Agent — what it is, and where it fits ProjectX

Checked 2026-11-05 against a read-only clone of `NousResearch/hermes-agent` (commit `ed1afb0`, 2026-11-04), the docs site, GitHub API and release notes. Question answered: *engine, Captain, crew bot, or neither?*

## 1. What Hermes is

| Fact | Evidence |
|---|---|
| **A single-user, persistent "assistant that learns" — plus the runtime it runs on.** One `AIAgent` loop (`run_agent.py`) reached through many front doors (CLI/TUI, gateway, ACP, API, Python lib). | [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) |
| Built by **Nous Research** (teknium et al.), raised a $90M Series B ($1.5B valuation). | [AGTP post](https://x.com/AGTPinsights/status/2116545510379008375) |
| **MIT licence**, Python (~1.06M lines incl. siblings), ~25k tests. | [LICENSE](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE), repo |
| **260.6k ★, 57k forks, 53k open issues**, pushed hourly; v0.22.7 on 2026-11-03; v0.22.5→v0.22.6 alone = 3,448 commits / 776 PRs in ~10 days. Very alive, very churny. | [GitHub API](https://api.github.com/repos/NousResearch/hermes-agent), [releases](https://github.com/NousResearch/hermes-agent/releases) |
| **Model-agnostic**: Nous Portal (OAuth subscription, 300+ models + Tool Gateway for search/images/TTS/browser), OpenAI/Codex OAuth, Anthropic (Claude Max OAuth or key), OpenRouter, DeepSeek, Gemini/Vertex, Bedrock, Azure, xAI Grok OAuth, MiniMax, Kimi, Ollama/local, any OpenAI-compatible URL. Switch with `/model`. | [Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) |
| Hosted option: **Hermes Cloud** on the Nous Portal (managed instances). | [Manage Hermes Cloud](https://hermes-agent.nousresearch.com/docs/guides/manage-hermes-cloud-with-mcp) |

## 2. Shape (memory, skills, tools, scheduling, delegation, approvals)

| Area | What exists |
|---|---|
| Identity & memory | `SOUL.md` (persona, first in system prompt), `MEMORY.md` (2,200 chars) + `USER.md` (1,375 chars) curated by the agent, FTS5 search over past sessions, Honcho user modelling, pluggable memory providers. One agent per Hermes home; "profiles" give extra agents their own home. [Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) |
| Skills | agentskills.io-compatible `SKILL.md` folders; agent writes new skills after hard tasks and edits them in use; Skills Hub. [Skills](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) |
| Tools | 70+ tools / 28 toolsets: terminal (7 backends: local, Docker, SSH, Modal, Daytona, Vercel…), files, web, browser (CDP), vision, `execute_code`, MCP client (and `mcp serve`), image/video gen, TTS. |
| Sessions | SQLite (`hermes_state.py`), branch/rewind/compress, shared across CLI, Desktop, gateway, webapp. |
| Scheduling | **Cron** (durable, fresh session per tick, delivers to any chat, "automation blueprints", no-agent script mode), `/heartbeat` (recurring prompt in one session), `/loop` (timer re-run), `/goal` (Ralph-style judge loop). [Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron), [Goals](https://hermes-agent.nousresearch.com/docs/user-guide/features/goals) |
| Sub-agents | `delegate_task`: up to 10 parallel isolated children, runs in background and **posts the result back as a message** (wake on completion), `output_schema` validation, pluggable delegation backends (route by model/profile). Public Subagent Lifecycle API for plugins. [Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) |
| Multi-agent | **Kanban**: SQLite task board shared by profiles; statuses `triage→todo→ready→running→blocked→review→done`; parent/child links; a **dispatcher inside the gateway** spawns `hermes -p <profile> chat -q` workers, retries, circuit-breaker, reviewer gate, PR completion contracts, external worker lanes via plugin. **A2A** protocol in/out; Desktop "Group Chats" with lead routing; Mixture-of-Agents provider. [Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban), [A2A](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/a2a) |
| Human-in-the-loop | Dangerous-command approval (`smart` / allowlist; cron/unattended default *deny*), DM pairing/allowlists, `clarify` tool, approvals surfaced in chat and ACP. [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) |
| Inbox/notifications | No first-class Inbox object. Delivery = cron/subagent results pushed to a chat; a community "inbox" plugin exists in the catalog (Hermes Labs). |

## 3. Interfaces

- **CLI / TUI** `hermes`, `hermes --tui`; **Desktop** (Electron, mac/win/linux) and **Webapp** (`hermes webapp`, port 9120, served by the gateway, one-use sign-in links) — this is what Oscar's Tailscale `/` → 9119 already points at (the dashboard).
- **`hermes acp`**: stdio JSON-RPC ACP server. Exposes session create/fork/cancel, prompt, streamed chunks, tool-call events, diffs, **permission requests**, auth; uses the `hermes-acp` toolset (files, terminal, web, memory, todo, skills, execute_code, delegate_task, vision) and *excludes* messaging and cron. Used by Zed/VS Code/JetBrains, Buzz, OpenBot. [ACP](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp)
- **HTTP API**: OpenAI-compatible `/v1/chat/completions` on the gateway (port 8642, SSE, tool progress inline). **TUI gateway JSON-RPC** (stdio/WebSocket) with `session.*`, `prompt.submit`, `prompt.background`, `session.steer`, approvals — the "custom host" protocol. [Programmatic integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration)
- **Gateway channels**: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Email, SMS, Teams, Google Chat, iMessage (BlueBubbles), Feishu, WeCom, DingTalk, IRC, LINE, ntfy, Home Assistant, A2A, webhooks… one process, cross-platform session continuity.
- **Web dashboard** `hermes dashboard` (9119): config, keys, MCP, pairing, cron, skills, sessions, logs, kanban, optional chat tab; themeable and pluggable.
- **Mobile**: official iOS/Android **"Hermes Mobile" beta launched at NousCon 2026-10-31**; it is a *client for your own gateway* (self-hosted or Hermes Cloud) with chat + visible tool calls, Live voice mode, cron/routines, skills/MCP/plugins, kanban, spend, SSH/Mosh terminal. Beta full (Android Firebase cap hit, TestFlight 404s), waitlist at [hermes-agent.nousresearch.com/mobile](https://hermes-agent.nousresearch.com/mobile); no store listing or date. [Kukuruza review](https://x.com/real_kukuruza/status/2116812260781838603), [issue #147269](https://github.com/NousResearch/hermes-agent/issues/147269), [Nous post](https://x.com/NousResearch/status/2117696344756252822)

## 4. Extensibility

- **Python plugins** (`~/.hermes/plugins/<name>/plugin.yaml` + `register(ctx)`): tools, hooks (`pre_llm_call` can block a turn, `post_tool_call`, `post_compaction`, kanban lifecycle…), middleware, slash commands, platforms, model/memory/context-engine/delegation/browser/image providers, locales; `plugin_storage` gives a data dir + SQLite. A plugin catalog with tiers. [Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/)
- **UI**: three separate SDKs — dashboard UI plugins (JS bundle + `manifest.json`, custom tabs, slots, FastAPI routes under `/api/plugins/<name>`), **Desktop plugin SDK** (`@hermes/plugin-sdk`: panes, pages, sidebar, status bar, commands, themes, hot reload), CLI skins. [Extending the dashboard](https://hermes-agent.nousresearch.com/docs/user-guide/features/extending-the-dashboard), [Desktop SDK](https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk)
- **So yes, you can build a product on it** without forking — but it is a Python agent plus a React/Electron desktop, with three UI plugin systems that are not the same thing, and a weekly-moving core. You own the glue; Nous owns the shape.

## 5. Can it orchestrate?

Honestly: **more than "one employee", less than a company.** Kanban + dispatcher + profiles + review gates + cron + `delegate_task` background completion is a real multi-agent kernel (spawn, track, retry, wake on completion). What it lacks for ProjectX: no Captain/Bot/Issue/Routine/Inbox model (it has board/card/profile/cron/chat), no budgets per issue, no "only the Captain talks to the human" wall (every profile is a chat bot), no business loop — you would write that as plugins and prompts, and the user-facing UI would be the Hermes Desktop/Webapp with your panes bolted in.

## 6. Hermes vs DeepSeek Harness (DSH 0.2.1-rc.5)

| Topic | Hermes Agent | DeepSeek Harness |
|---|---|---|
| Engine / plugins | Python; `plugin.yaml` + `register(ctx)`; hooks, providers, tools; 3 UI SDKs | TypeScript; everything (even the shell) is a plugin in a profile; `ctx.tools`, `ctx.storage`, `ctx.slots`, projections ([dsh-extensibility.md](dsh-extensibility.md)) |
| Custom UI | Add panes/tabs to *their* Desktop/dashboard; own chat window | Own web shell possible via `root`/`main` slots; `dsh web` serves it |
| Sub-agents / multi-agent | `delegate_task` (bg, wake on done), Kanban dispatcher + profiles, A2A, MoA | `dsh-subagent` (spawn/fork in-process), `dsh-tool-subagent-control`, experimental agent-team with task board |
| Scheduling | Cron (durable, delivery to chats), heartbeat, loop, goal | `dsh-schedule` + `dsh-tool-schedule` (reminders), goals |
| Remote / web / mobile | Gateway to 20+ chat apps, Webapp, Dashboard, HTTP API, **native mobile beta** | `dsh web` behind Tailscale (decision #19); ACP; no messaging, no mobile |
| Memory | SOUL/MEMORY/USER + session search + skills that self-write | Context files; no built-in learning loop |
| Ecosystem / maturity | 260k ★, huge community, plugin catalog, 53k open issues, breaking churn | Small, young (rc), MIT packages; small blast radius, fewer ready parts |
| Licence | MIT | MIT on installed packages (npm shows BSD-3 on an older rc) |

## 7. Verdict for ProjectX

| Option | What it means | Verdict |
|---|---|---|
| **(a) Hermes as engine instead of DSH** | Rebuild Captain/Issues/Routines/Inbox as Hermes Python plugins + Desktop panes; use Kanban as the task kernel, cron as Routines, gateway + mobile as the window. | **Not now.** Breaks decision "DSH, plugins only" and the engine boundary already built; rewrites the plan in Python; ties the product UI to Nous' Desktop and its weekly churn. Gains: chat apps, mobile, memory, 260k-★ ecosystem. Loses: our own clean UI, the DSH work to date, control of pace. |
| **(b) Hermes as the Captain's brain inside DSH** | DSH owns UI/Issues/Inbox; the Captain's reasoning runs in Hermes via ACP/JSON-RPC. | **No.** Two brains, two memories, two approval systems; the Captain *is* the thing DSH is good at (tools + storage + projections). Hermes' value (memory, chat apps) is wasted behind DSH. |
| **(c) Hermes as a crew bot via ACP** ✅ | Register Hermes behind the bot-provider interface (send · wait · tail · interrupt) exactly like `grokbot`; `hermes acp` gives permission requests, streaming, cancel, fork. Give it its own profile so its memory/skills stay separate. Good fit for research/marketing bots that benefit from its browser, web search and self-written skills. | **Recommended, after grokbot works.** Costs one adapter; Oscar keeps using the Hermes he likes. Watch-outs: ACP mode has no cron/messaging (fine — Routines are ours); 53k-issue churn means pin a version. |
| (d) Neither | Ignore it. | Wasteful: it's already on the leader and ACP-ready. |

**Bottom line for Oscar:** keep Hermes as your personal assistant and phone window to the server; let ProjectX hire it as a bot. If, after FillLight's loop runs end to end, the DSH UI is the pain point, revisit (a) with real data — Hermes' mobile + chat gateway is the one thing we cannot cheaply build. Keep the engine boundary so that switch stays possible.

Third-party reads: [blakecrosley.com/guides/hermes](https://blakecrosley.com/guides/hermes), [fast.io review](https://fast.io/resources/hermes-agent-review-2026/), [gradually.ai stats](https://www.gradually.ai/en/hermes-agent-statistics/). Related: [open-source-landscape.md](open-source-landscape.md) (Paperclip already has a `hermes` adapter — the "company around Hermes" pattern exists elsewhere).
