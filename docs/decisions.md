# ProjectX — Decision Log

Working name: **ProjectX** (under LieuGroup). Open source, MIT.

This file is the written record of the founding conversation (2026-11-04).
CEO: Oscar (LieuGroup). CTO: the agent. Rule: CEO brings problems and picks
between options; CTO owns the technical side and explains in plain English.

## What we are building

A "company factory" / super app for one person. One place where Oscar runs
his products (FillLight, Heardback, QuickBook360, …) through a **Captain**
(manager agent) and a **crew** of bots, with an issues board, routines and an
inbox — instead of juggling many chat sessions and many tools.

The loop is a *business* loop, not only a code loop:

    research market → ship → market → measure → fix conversion → repeat

First focus is **dark marketing** for FillLight: ASO, Apple Ads, TikTok
creatives, learned from indie developers on X.

## Core concept (the thing LilOS got wrong)

| Word     | Meaning                                              | Equivalent         |
|----------|------------------------------------------------------|--------------------|
| Bot      | *Who* does work. Permanent, named, has memory.       | Employee           |
| Thread / Issue | *What* is done. Temporary, has status.         | To-do item         |
| Routine  | *When* it happens by itself.                         | Calendar           |
| Inbox    | What needs the human.                                | Your desk          |
| Captain  | Manager bot. Talks to you, dispatches, never does the work. | Chief of staff / DSH "Router" |
| Session  | Technical detail; never shown to the user.           | Employee notebook  |

Rules: a bot owns many threads; a thread belongs to exactly one bot; only the
Captain and the inbox talk to the human.

## Decisions

| # | Decision | Choice | Why |
|---|----------|--------|-----|
| 1 | Engine | **DeepSeek Harness (DSH)**, as plugins, no fork | MIT; everything-is-a-plugin; Agent Teams + super-agent prototype already exist; local plugins load into the official app |
| 2 | Engine boundary | Keep a clean boundary so ACP / other engines can be plugged in later | Future plug-and-play |
| 3 | Hands (bots with real tools) | **Grok Bot now** via `grokbot` CLI; **OpenBot later** via its CLI. Both behind one "bot provider" interface | Oscar's data already flows through Grok Bot (PostHog, RevenueCat, Postiz, X). Switching becomes config, not rewrite |
| 4 | Grok Bot plan | $100 tier is enough | Low usage |
| 5 | First product | **FillLight** | It makes money now |
| 6 | Not continuing | LilOS / Nuncio | Built engine + product at once; too much time |
| 7 | Licence | MIT, open source under LieuGroup | — |
| 8 | Human approval always required for | spending money, publishing publicly, changing prices, merging | Safety |
| 9 | Secrets | Never in chat; vault + read-only keys first | Oscar has pasted keys into chats before |

## Research done (see `research/`)

- `dsh-extensibility.md` — plugin vs fork verdict: all of Captain, Issues,
  Threads, Board UI are plugins; replacing the main layout is probably a plugin
  (untested).
- `capy.md` — how Capy Captains work (Captain = thread with crew; AGENTS.md /
  CREW.md / NOTES.md / DIGEST.md; wakes when crew finishes).
- `claude-cursor.md` — Claude Projects (knowledge container) vs Cursor
  Projects (coordinator with isolated worktree agents).
- `grokbot-and-x.md` — Grok Bot bot model (identity, routines = prompt +
  trigger, rooms, typed inbox, approvals); Oscar's 10 bots; X status.
- `business-data.md` — App Store Connect: 22 apps, 5 live, 4 rejected;
  FillLight 1.1.5 live Oct 28, $0 last 28 days. Stripe blocked by sandbox.
- `user-context.md` — products, stack, how Oscar works, recurring pains.

External references:
- DSH super-agent article (Router/Worker, triggers, threads, inline & spill,
  watchdog): https://x.com/DeepSeekHarness/status/2116857257245987236
- Prototype plugin: https://github.com/deepseek-ai/dsh-superagent-prototype
- OpenBot: https://github.com/CopilotKit/openbot
- T3 Code remote architecture (for phone access later):
  https://github.com/pingdotgg/t3code

## Version 1 scope (FillLight, dark marketing)

Routines:
1. Weekly scout — search X for indie-dev ASO / Apple Ads / TikTok tactics;
   1-page digest to inbox.
2. Daily numbers — installs + revenue via Grok Bot connectors; flag drops;
   auto-open a "fix" thread.
3. Creative pipeline — hooks → script → Postiz draft → approval → publish.
4. Store watch — App Store Connect status, rejections, reviews → threads.

Screens (names TBD): Home (Captain chat + digest), Crew (bots), Issues
(threads board), Routines, Inbox.

## Open items

- Stripe read access (run outside sandbox, or read-only restricted key).
- App Store vendor number for sales/downloads.
- X: regenerate the bearer token that was pasted in chat; app-level token is
  enough for search (which is the main use).
- Untested: loading a local plugin into the official DSH app; replacing the
  main layout via the `root` slot.
- Read `dsh-superagent-prototype` to see how much Router/Worker/Thread exists.

## Next step

Write the one-page plan (screens + plugin layout), then build the smallest
working piece: a Captain that talks to Oscar and keeps an issues board, loaded
as a local plugin into the official DSH app.
