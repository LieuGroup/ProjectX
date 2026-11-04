# User context: Oscar's products and agent workflow

Sources: `~/.claude/projects` (about 95 dirs), user messages and `cwd` fields sampled with python, plus repo listings, `git remote` and the latest commit, all read-only. There is no `~/.claude/CLAUDE.md`. No secrets are copied here. Window: about 2026-10-15 to 2026-11-04. The user writes in English and Vietnamese and lives in Adelaide (AU market).

## Products
| Product | What | Stack | Repo / path | Status (from transcripts) |
|---|---|---|---|---|
| **Heardback** | AU job-application SaaS: upload a CV and a job ad, get a free tailored preview, pay to unlock; SEO guides on SEEK (heardback.com.au/guides) | Next.js, Prisma, Stripe, Inngest, PostHog, Vercel Blob, OpenRouter/Gemini (zero-data-retention promise), Docker compose (app + marketing) | `~/Desktop/Oscar/heardback`, github oscarlehuu/heardback | Live website; work on SEO titles (PR #91), a free first-pack code (#74), and a "Heardback Bot" Threads warm-up automation |
| **QuickBook360** ("QuickBook") | Booking SaaS | Next.js, Convex (+auth, rate-limiter), shadcn, Playwright e2e, Docker; dev service on oscar-dev-server port 3360, Inngest | `~/Desktop/Oscar/QuickBook360`, oscarlehuu/quickbook360 | Active; last commit 2026-10-22 |
| **QuickShift** | Shift/roster product (exact type unclear) | Not found locally (older path `/Users/oscar/Desktop/projects/oscarprojects/quickshift` on another Mac) | ? | Portfolio report lists "QuickShift 0.3.9" live |
| **FillLight** | iOS app with a yearly paywall, on RevenueCat | Native iOS (`ios/`), Convex backend + auth, AWS SES, marketing `site/` | `~/Desktop/Oscar/FillLight` (+ version-copy dirs), oscarlehuu/FillLight | 1.1.4 was rejected; 1.1.5 yearly paywall merged 10-29; RevenueCat 28-day revenue is $0 with 1–2 trials; heavy TikTok marketing |
| **LilOS** | His own "agentic + coding + productivity in one" harness with an orchestrator over ACP engines (Devin, Claude Code via claude-agent-acp, Hermes) | Bun monorepo, TypeScript, Biome, Vitest, Playwright e2e on sharded CI | `~/Desktop/Oscar/LilOS`, Nuncio-hq/LilOS; T3 worktrees `~/.t3/worktrees/LilOS/t3-*` | Most-active project (about 2,000 messages); issues around #846–#901 by 10-28 |
| **FootyPresident** | AFL management game | Swift prototype; Gemini/Veo media through flowctl | `~/Desktop/Oscar/FootyPresident`, LieuGroup/footy-president | In App Store review; TikTok @footypresident launched 10-26 |
| **QRit** | Client project ("Sam") | pnpm/turbo, Supabase, Deno | `~/Desktop/Oscar/SamProjects/QRit`, oscarlehuu/qrit | Handed off and invoiced 10-27; the user shut it down on 10-28 |
| Others | OpenRork (app-building and marketing agent idea: ACP, SwiftUI first, at most 3 code agents), ManeBooth, Oathwise, GymJoy, Glowmax, StyleMaxx, Wrap, Bible, AppStoreAPI, Grokbot-cli | mostly iOS apps (Expo/SwiftUI) | `~/Desktop/Oscar/*` | An "app-studio-ops" Notion page tracks downloads, revenue and trials for each app |
| AdelaideUni | HPC/SLURM GPU work (self-hosting DeepSeek, Qwen, MiMo) | SLURM | `~/Desktop/AdelaideUni` | Ongoing |

## How he works with agents
- **Many engines**: Claude Code in T3 Code (worktrees + scratch threads), Devin, Capy, Hermes/OpenClaw, Grokbot (per-product bots: "FillLight Bot", "Heardback Bot"), Herdr, Amp, cliproxyapi.
- **Lead delegates by issue** (`delegate-task` / `thread-send`, ids like `lilos-<issue>-<engine>-<role>`) with numbered review rounds.
- **GitHub issue → PR → review → merge** with terse approvals ("merge it now", "ok go"); he sometimes holds a merge for his own review.
- **Q&A spec rounds** (Q1…Q24 "yes as recommended").
- **Remote dev**: wants Linux `oscar-dev-server-leader` as the coding box, with the Mac for iOS builds and simulators only; controls things from his phone over Tailscale/HTTPS.
- **Marketing loop**: TikTok/Threads hooks (Apify), Veo/Gemini/HPC video, Postiz drafts and analytics, xurl for X research, captions and email drafts.

## Recurring pains
- Context exhaustion (many "continued from previous conversation" sessions); context lost between tools.
- Tool sprawl; looking for one product for agentic, coding, productivity, GUI and mobile.
- CI failures after merges (16 e2e specs), conflicts between parallel PRs, worktree and disk clutter.
- App Store rejections; weak monetisation.
- Secrets pasted into chat (an X token, pairing codes); a vault is needed.
- Hard rules: "don't start drama 3, do not ever do that", read-only tasks, no merge without review.

## Implications for a "business factory" coordinator
1. A portfolio **Captain** with a registry per product (repo, stack, deploy, store status, revenue from RevenueCat, App Store Connect, Stripe and PostHog).
2. **Issues → Threads**: worktree per issue, engine per issue, automatic review rounds, CI watch, human gate for merge, spend, posting and messaging.
3. **Marketing as first-class issues**: a creative pipeline with an analytics feedback loop.
4. **Remote and mobile approvals**, running on the Linux leader.
5. **Durable memory and decision logs per product** (reuse AGENTS.md / DECISIONS.md / plans/).
6. Policy guardrails and secret handling built in.
