# Open-source landscape — what ProjectX can borrow or build on

_2026-11-04. Stars/licence/activity from the GitHub API on that day. Read-only survey; nothing cloned._

Question: is there an open-source project that already does "Captain + crew of bots running a
*business* loop, with Issues / Routines / Inbox / Crew", that we should build on instead of
writing DSH plugins? Short answer: **one project (Paperclip) is a near-exact concept match, but it
is its own engine, not a DSH plugin.** Everything else is a coding-agent tool, an agent framework,
or a plain issue tracker.

## Sceptic's lens

- "Cron job with a marketing budget" = a scheduler that fires a prompt. Routines alone are not orchestration.
- Real orchestration = a manager that *decides* who does what, tracks work items with status and owner,
  routes questions/approvals to a human, and wakes workers on events, not only on timers.
- Coding-only tools (worktree + diff + PR) are the wrong shape for ASO / Apple Ads / TikTok work.

## Survey

| Name | What it is | Licence | Stars / activity | Reusable for ProjectX | Why not base on it |
|---|---|---|---|---|---|
| [Paperclip](https://github.com/paperclipai/paperclip) | Node server + React UI that "runs an AI company": org chart of agents, issues, goals, routines, approvals, inbox, budgets. "If OpenClaw is an employee, Paperclip is the company." | MIT | 111k ★, pushed today, 6.5k open issues, 18k forks (created 2026-03) | **Concept + data model**: heartbeat runtime (agents wake on `timer` / `assignment` / `on_demand` / `automation`, wakeups coalesced), adapter interface (`execute()` per agent runtime; built-ins for Claude Code, Codex, Hermes, OpenClaw gateway, `process`, `http`), Postgres schema for `issues`, `routines`, `approvals`, `heartbeat_runs`, `goals`, `inbox_*`. Pages: Issues, Inbox, Approvals, Agents, Goals, Costs. Already has a `grok-local` adapter. | It *is* an engine (Postgres, own runner, 130 MB of TS, ~170 schema tables). Adopting it violates "DSH plugins, no fork" and we would run two brains. Huge, fast-moving, 6.5k open issues = churn. Borrow the shape, not the code. |
| [dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) | DeepSeek's proactive agent for one GitHub repo, shipped as a **DSH bundle**. Source → Trigger → Thread → Eval → Ticket → Assignment → Executor; web panel for "what needs a person". | MIT | 17.8k ★, pushed 2026-11-03; explicitly "research prototype" | **Code**: it is already DSH plugins. Domain model (Source/Trigger/Thread/Ticket/Mandate/Channel/Policy) maps to our Captain/Issue/Inbox; "Eval" is our Captain's dispatch step; panel = Inbox. Cookbook shows how to add a Source (we'd add Grok Bot / App Store Connect / X). | Coding-centric (GitHub + Feishu sources, PR output); no Crew page, no Routines UI, no cost tracking; prototype quality by its own admission. Still the best *starting code*. |
| [OpenClaw](https://github.com/openclaw/openclaw) | Personal AI assistant daemon ("Gateway") reachable from Discord/Telegram/WhatsApp/iMessage/etc.; models and harnesses are plugins; cron + heartbeat built in. Foundation-stewarded. | MIT | 394k ★, pushed today | **Concept**: gateway/typed-WS architecture, heartbeat that defers while cron runs, channel adapters for phone access. Could be a *bot provider* later (Paperclip already drives it via `openclaw_gateway`). | Single-assistant shape: no board, no manager, no work items. It's an employee, not a company. |
| [Hermes Agent](https://github.com/NousResearch/hermes-agent) | Nous Research's self-improving agent: skill creation from experience, memory, cron scheduler with delivery to Telegram/Slack/etc., subagents, 6 terminal backends (local/Docker/SSH/Modal/Daytona). | MIT | 260k ★, pushed today | **Concept**: "closed learning loop" (skills written after complex tasks) is what we want for ASO tactics; cron-with-delivery = Routine → Inbox. Could be a bot provider via its HTTP API (Paperclip has `hermes_gateway`). | Python, single-agent, no issues board or manager. Its cron is exactly "cron job with a budget". |
| [OpenHands](https://github.com/OpenHands/OpenHands) | Full coding-agent platform (sandboxed runtime, web UI, SDK), formerly OpenDevin. | MIT | 92.6k ★, pushed today | Sandbox runtime pattern; nothing for marketing. | Pure software-engineering agent; heavy Docker footprint; our bots already have hands (Grok Bot). |
| [Vibe Kanban](https://github.com/BloopAI/vibe-kanban) | Kanban board that launches 10+ coding agents in git-worktree workspaces; review diffs, preview, PR. | Apache-2.0 | 28.6k ★, pushed 2026-11-03, **sunsetting** (community-maintained from now) | **UI**: issue board + "workspace per issue" + inline review are the right shape for our Issues page; Rust+React, clean. | Sunset; coding-only (branch/terminal/dev-server per task); no manager agent, no routines, no inbox. |
| [Backlog.md](https://github.com/MrLesk/Backlog.md) | Markdown-native task board for human+agent collaboration: one `.md` per task, acceptance criteria, CLI + TUI + small web UI, MCP server. | MIT | 7.3k ★, pushed 2026-10-29 | **Data model**: tasks as plain markdown files in the repo (`FL-12.md` with status/assignee/AC) — a zero-infra store for Issues that fits `ctx.storage` or git. The "review spec → plan → code" gates = our approval gates. | Solo-dev coding tool; no agents running, no scheduling, no inbox. Borrow the file format only. |
| [Claude Squad](https://github.com/smtg-ai/claude-squad) | TUI managing multiple Claude Code / Codex / Gemini sessions, one git workspace each. | AGPL-3.0 | 8.7k ★, last push 2026-08 (slowing) | Pattern for running N CLI agents in tmux-like isolation and tailing them. | AGPL (MIT repo cannot absorb code), coding-only, no concept of work items. |
| [Open SWE](https://github.com/langchain-ai/open-swe) | LangChain's "software factory" on Deep Agents: issue → plan → sandbox → PR → review → CI babysit, with schedules. | MIT | 11.2k ★, pushed today; "not accepting issues/contributions", needs LangSmith Managed Deep Agents | The investigate→implement→validate→deliver→review loop diagram is a good template for our *business* loop phrasing. | Depends on LangChain's paid runtime; coding-only; closed to contributors. |
| [Agent Zero](https://github.com/agent0ai/agent-zero) | General agent with its own Linux desktop in Docker; hierarchical sub-agents; web UI. | MIT | 19.7k ★, pushed 2026-11-01 | Sub-agent hierarchy prompt design; "agent owns a computer" ≈ OpenBot. | Monolithic Python app, chat-shaped UI, no board/routines/inbox. |
| [OpenBot](https://github.com/CopilotKit/openbot) | "AI coworkers that each get a computer": Docker-compose template, Postgres, AG-UI protocol, per-coworker browser/files/tools, human take-over. Explicitly "a template, not a product". | MIT | 10.1k ★, pushed today, beta | Already in our plan as a later bot provider (decision #3). Approval-before-action + takeover pattern. | Template meant to be forked and owned; it is an *employee runtime*, not the company. |
| [Huly](https://github.com/hcengineering/platform) / [Plane](https://github.com/makeplane/plane) | Open-source Linear/Jira clones (Huly: all-in-one incl. chat; Plane: issues/cycles/docs). | EPL-2.0 / AGPL-3.0 | 28k ★ / 62k ★, both pushed this week | **UI reference** for Issues board (keys like FL-12, states, triage, cycles). Plane has a REST API we could sync to if Oscar ever wants a "real" tracker. | Both are multi-service platforms (Mongo/Postgres/Elastic/Redis); AGPL/EPL unfriendly to MIT; zero agent awareness. Overkill for one person. |
| Frameworks: [CrewAI](https://github.com/crewAIInc/crewAI) (MIT, 60k), [Mastra](https://github.com/mastra-ai/mastra) (Apache + `ee/` dirs, 29k), [Letta](https://github.com/letta-ai/letta) (Apache, 25k), [Julep](https://github.com/julep-ai/julep) (Apache, 6.6k) | Libraries for building agents: role-based crews, TS workflows, stateful memory, durable flows. | see left | all active | Vocabulary only: CrewAI "role/goal/backstory" per bot; Letta's memory blocks for bot memory; Julep's crash-and-resume flows for long Routines. | They are *inside-the-agent* frameworks; DSH is already our agent runtime. Adding a second runtime is the LilOS mistake. |
| Dropped | [Daytona](https://github.com/daytonaio/daytona) repo **archived** (2026-07); Conductor (conductor.build) is a closed Mac app, not OSS; [Kilo Code](https://github.com/Kilo-Org/kilocode) orchestrator is a VS Code mode (boomerang subtasks), coding-only; [Superagent](https://github.com/superagent-ai/superagent) pivoted to prompt-injection firewall; [AgentOps](https://github.com/AgentOps-AI/agentops) is a Python tracing SDK (cost tracking idea only, last push 2026-06); Plandex coding agent, last push 2025-10. | | | | |

## Top 3 to look at deeper, and why

1. **dsh-superagent-prototype** — the only candidate that is *already DSH plugins*. Read `docs/001-domain.md`
   → `007-architecture.md` and the Source cookbook. Decide: fork its Ticket/Thread/Eval/Panel code into our
   plugin set (renaming to Issue/Captain/Inbox) vs. write fresh and copy only the shape. Risk: coding-only
   assumptions baked into prompts and the panel.
2. **Paperclip** — the concept match. Steal the *design* not the code: heartbeat wake reasons, wakeup
   coalescing, adapter `execute()` contract (our "bot provider" interface should look like this, including
   its `grok-local` and `process` adapters), the `issues`/`routines`/`approvals`/`inbox` schema, and the
   page list (Dashboard, Issues, Inbox, Approvals, Agents, Goals, Costs). It also proves there is appetite
   for "agents at work" tooling — and that we should not compete with it but stay DSH-native and personal.
3. **Backlog.md** — simplest viable Issue store: one markdown file per issue, in git, readable by any
   agent and by Oscar on his phone via GitHub. Pairs well with DSH `ctx.storage` for status/index. If we
   adopt its front-matter format we get a CLI and a board for free while our own UI is being built.

## Does anything match outright? No — here's why

Nothing ships "manager agent + crew + business loop" *as a DSH plugin*. Paperclip is the full idea but
as a standalone Postgres product with its own runner, so building on it means abandoning decision #1
(DSH, plugins only) and running two orchestration brains. Everything else is one of: a single powerful
employee (OpenClaw, Hermes, Agent Zero, OpenBot), a coding-only factory (Vibe Kanban, Open SWE,
OpenHands, Claude Squad, Kilo), a plain tracker (Huly, Plane, Backlog.md), or an inside-the-agent
framework (CrewAI, Mastra, Letta, Julep). None has marketing (ASO/Ads/TikTok) as a first-class job.
The marketing loop will be ours to build; the orchestration *shape* can be copied from Paperclip and the
DSH-native *code* from dsh-superagent-prototype.

## Follow-ups for the plan

- Add to `docs/decisions.md`: "borrow Paperclip's heartbeat + adapter design; start from
  dsh-superagent-prototype's domain code; Backlog.md-style markdown issues" once Oscar agrees.
- Licence check before copying any code: only MIT/Apache sources (Paperclip, dsh-superagent, Backlog.md,
  Vibe Kanban OK; Claude Squad, Plane = AGPL, Huly = EPL — concepts only).
