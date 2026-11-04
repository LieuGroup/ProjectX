# Grok Bot (grokbot CLI) and X marketing presence

Date: 2026-11-04. Everything here comes from read-only commands only: `--help`, `whoami`, `agents`, `team`, `computers`, `routines-list`, `skills-list` and `mcp-list`, plus `xurl` GET requests. I created, ran, changed and sent nothing.

## 1. What grokbot is
- `/opt/homebrew/bin/grokbot` is a symlink to `/opt/homebrew/lib/node_modules/grokbotagent-cli/dist/cli.js`, a Node script.
- It is npm package `grokbotagent-cli` v0.5.1. There is no README.
- It ships two skills: `skills/grokbot/SKILL.md` and `skills/grokbot-inbox/SKILL.md`.
- `whoami` shows it uses the Grok Bot desktop app's login (app 0.78.4; the CLI's command catalog was checked against 0.75.0).
- The CLI's own tagline is "Orchestrate your GrokBot workforce from any terminal". It lets an external coding agent (Claude Code, Cursor, us) brief the bots and receive work back from them.

## 2. How a bot is built (from help text, SKILL.md and the account's real data)

| Part | How it works |
|---|---|
| Identity | Each bot has a name, a title ("HeardBack growth owner"), a description, and instructions (its brief). `hire` creates one with its own `/workspace/staff` folder. Every bot on this account runs on the `temporal` harness. There is one **primary Bot** ("Grok Bot"); the rest are staff. |
| Computer | "always-on assistants, each with its own cloud computer, memory and connectors" (SKILL.md). One cloud "Box" is shared by all the bots ("the cloud computer is the same for every bot"). `vnc` opens the box desktop. `computers` lists registered local machines (this Mac, `LC3Y34J661X`), which bots can use for local tools after you approve them. `worker-start` can run a self-hosted cursor-agent worker. |
| Skills | One skill library for the whole Box (SKILL.md files). It includes grokbot-managed skills (booking, shopping, `site-playbooks-*`, `add-connector`, `export-bot-template`) plus skills you import (`skills-push`). Changing the library affects every bot. |
| Connectors | Box MCP servers: Postiz, Cloudflare, Notion, zernio, **X**, PostHog, RevenueCat, Slack, Shadcn, Finance, GitHub, Origin. |
| Routines | A routine is a saved prompt plus a trigger. Triggers can be a cron schedule (with a time zone), a webhook (key kept in the Keychain), email to the bot's inbox (with a sender filter and auth check), GitHub events (ci-failed, pr-opened…), or a JSON trigger (Slack, Teams, Linear, Sentry, PagerDuty, Origin, or a group of these). You can enable, disable, fire and update them. |
| Memory | Each bot has its own memory. Team Bots have *shared* memory and skills, and changes go through `review` / `review-resolve`, where the owner or a manager approves. There are files shared with the team and private ("Only you") files. |
| Human in the loop | Approval cards (`approvals`, `approve`, `deny`), question widgets (`answer`), and bots marked protected (`main`/`test`) that need `--force-protected` before changes. |
| Messaging | `ask` waits until the "Done Rule" settles. `send` returns a job ID you can resume (and queues the message if the bot is busy), followed by `wait`, `tail`, `interrupt`, `reply` and `edit`. Each message carries a header naming the caller's project, runtime and session. |

## 3. Bots working together
- **Rooms** (`room-create`, `room-members`): several bots answer one message.
- **Inbox**: a bot leaves an item for the owner's assistants with `inbox-post`. Item types are bug, task, question, result and decision, with a JSON schema. The local agent then runs `inbox`, `inbox-show`, `inbox-fetch`, `inbox-reply` and `inbox-done`. `inbox wait` / `inbox check` are meant for a hook before the agent stops, so it can't end its turn with undelivered items.
- **Primary Bot as manager**: `setup-bots` / `update-bots` go through the primary Bot.
- **Hand-off to Cursor**: the `cloud-agents` / `cloud-agent` commands, and LilOS Dev bots that "Build slices in Cursor cloud agents… one draft PR per task".

**This account's roster** (all idle): Grok Bot (primary), two test bots, Side Hustle CEO, FillLight Bot (iOS), HeardBack Bot (growth owner), LilOS Dev and LilOS Dev 2, Selah Bot (iOS growth and data), and Footy President Bot (marketing). The pattern is **one owner bot per product, with goal-based instructions** (for example, "Goal: paying customers").

## 4. How marketing automation is done (the HeardBack Bot's real routines)
All of these run on cron in Australia/Adelaide time:
- **Daily growth session** (Mon–Fri 10:15): first a production health check (synthetic upload; checks that the Stripe checkout link appears, never pays). If the check fails, fixing it is the whole session. The routine follows an Operating rules page in Notion.
- **Weekly results** (Mon 9:15): compares the last 7 days with the previous 7. Sources: Cloudflare traffic, Search Console, the PostHog funnel, and revenue from the app database.
- **Threads posts** (4 times a day) and **Threads engage** (4 times a day, about 30 minutes each): these drive the Threads account @heardback2026 through the **Aside CLI on Oscar's Mac** (machine ae968aee…). Approvals from Oscar are dated inside the prompts.

**X posting/monitoring would follow the same pattern.** A cron routine (or several) would use the X MCP connector, Postiz/zernio for scheduling, or Aside driving the browser. Separate routines would handle posting, engagement and a weekly metrics roll-up. Results, drafts that need approval, and anything blocked would go to the inbox or approval cards.

## 5. What we can copy for our coordinator
1. **Bot = brief + goal + owned product**, plus a persistent workspace folder and memory. Our coordinator would keep one roster of named, long-lived agents.
2. **Routines as data**: `{name, prompt, trigger}`, where the trigger is a union type (cron+TZ, webhook, email, git event, or a group). They can be enabled, disabled, fired, listed with their next run time, and created paused.
3. **Asynchronous jobs**: `send` returns a job ID, followed by `wait`/`tail`/`interrupt`; messages queue while the agent is busy; a "Done Rule" decides when a turn is complete.
4. **Typed inbox back to the coordinator** (bug, task, question, result, decision) and a stop-hook that refuses to finish with undelivered items.
5. **Safety**: approval cards, protected or test agents, review queues for changes to shared memory and skills, and `--yes` only after explicit owner approval. Secrets stay in the Keychain and are never printed.
6. **Health check before growth work**, and weekly comparisons over the same 7-day windows.
7. **Rooms** for asking many agents at once. We could also connect to Cursor cloud agents for code work.

## 6. X account (xurl)
- **Blocked:** `xurl` has only an app-only **bearer token** (`xurl auth status`: oauth2 none, oauth1 –). `/2/users/me` returns 403 "OAuth 2.0 Application-Only is forbidden". So I **could not see the user's account or their own posts**. To see them, the user needs to run `xurl auth oauth2`. Recent search does work.
- I ran recent searches for the product names (heardback, filllight, "footy president", selah app, lilos) over roughly the last 7 days. They found **no posts from or about the user's products**; the hits were unrelated uses of the same words.
- This fits the routines: HeardBack's social activity currently runs on **Threads**, not X. **[I]** The user's presence on X looks to be little or nothing.

### Practical advice found on X (small sample from recent search; treat as anecdotal)
- Submit to GitHub "awesome" lists and directories for early backlinks and domain authority. Each of these one-off listing jobs suits a bot.
- B2C apps and single-channel lead-gen products are said to beat horizontal B2B SaaS for solo builders. Pick one channel and go deep.
- Building in public plus actively engaging in the indie community (replies, connecting) is how small accounts grow.
- A useful sceptical take: "most 'ai agents' are cron jobs with a marketing budget". Measure results (the weekly results routine already does this), not just activity.

**[I] My own suggestions for X automation (not from any source):** draft posts from real product events (releases, metrics, user stories), and keep a person approving posts at first. Spend time on replies to relevant accounts, not only scheduled posts. Track the follower → site → install/checkout funnel in PostHog. Respect X's automation rules (no bulk follow or like). Note that the HeardBack Threads routine was told to do more following.
