# ProjectX — one-page plan (v1)

_Draft 2026-11-04. Status: waiting for Oscar's yes._

## What v1 is

The official DeepSeek Harness app, with a ProjectX plugin bundle loaded into it.
Oscar opens DSH, sees five ProjectX screens, talks only to the Captain. Nothing
else changes in DSH. No fork.

## The five screens

| Screen   | What Oscar sees                                                        | Built as                                                        |
|----------|------------------------------------------------------------------------|-----------------------------------------------------------------|
| Home     | Captain chat + today's digest (Needs you / New / In progress)          | A normal DSH chat session that *is* the Captain, plus a digest card |
| Crew     | The bots: name, product, what each is doing now, last result           | Full-page panel                                                 |
| Issues   | Board of threads: Open → Working → Needs you → Done; owner bot per card | Full-page panel                                                 |
| Routines | List of "when X, do Y" jobs, next run time, on/off                     | Full-page panel over DSH's built-in scheduler                   |
| Inbox    | Only things that need Oscar: approve / answer / decide                  | Full-page panel + a badge count                                 |

Full-page panels use DSH's `main` slot (same mechanism as its own Plugins page).
We do **not** replace DSH's layout in v1. That is tested in step 2 and only
matters if the panels feel cramped.

## The plugin bundle (one repo, three packages)

```
ProjectX/
  packages/
    projectx-profile/   the bundle: one cordis.patch.yml that turns the others on
    projectx-host/      brain + memory (runs inside DSH's host process)
    projectx-client/    the five screens (runs in DSH's web GUI)
```

**projectx-host** owns:
- Storage (via `ctx.storage`): `bots`, `threads`, `routines`, `inbox`.
- Model tools the Captain uses: `issue_create/list/update/assign`,
  `bot_list`, `bot_send` (hand a thread to a bot), `inbox_post`, `digest_update`.
- Bot providers behind one interface `{ send, wait, tail, interrupt }`:
  `grokbot` CLI first; OpenBot later; DSH's own Agent Teams for in-app helpers.
- A "projection" (live state) the client reads — same pattern Agent Teams uses.

**projectx-client** owns the five panels and the inbox badge. It never calls
tools; it reads the projection and sends actions to the host.

**projectx-profile** is 30 lines of YAML: load host, load client, set the
Captain's system prompt and the default model.

## What we reuse instead of build

| Need           | DSH already has                     | We add                                  |
|----------------|-------------------------------------|-----------------------------------------|
| Chat with Captain | Sessions + chat UI               | Captain persona + tools                 |
| Scheduling     | `dsh-schedule` (cron/daily/weekly)  | Routines panel that lists/toggles them  |
| Helper agents  | Agent Teams (spawn, task board)     | Nothing; Captain uses it for small jobs |
| Persistence    | `ctx.storage` schema domains        | Our four record types                   |
| Plugin loading | `dsh plugin add <local path>`       | Nothing                                 |

## Build order (each step is a working thing)

1. **Hello plugin** — one panel that says hello, loaded from this folder.
   Proves local loading. (Also try the `root` slot once, to answer "fork?" forever.)
2. **Issues board** — `issue_*` tools + Issues panel. Captain can say "I opened
   FL-3: fix paywall copy" and it appears on the board.
3. **Inbox** — `inbox_post` + panel + badge. Captain can ask Oscar a question
   without burying it in chat.
4. **Bot provider: grokbot** — `bot_send` hands a thread to "FillLight Bot",
   result comes back to the thread and the Captain is woken.
5. **First routine** — weekly X scout for ASO / Apple Ads / TikTok tactics →
   digest in Inbox. This is the first real FillLight value.

Steps 1–3 need no external accounts. Step 4 is read-only on Grok Bot until
Oscar approves a send.

## Rules baked in from day one

- Human approval tool gate for: spend, publish, price change, merge.
- Secrets live in DSH's credential store / Keychain, never in our storage.
- Everything is MIT and written as if public.

## Not in v1

Phone access, more than one product, replacing DSH's sidebar, OpenBot, Stripe.
