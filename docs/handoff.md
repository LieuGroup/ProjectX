# Handoff — where we are

_Last updated: 2026-11-04 by the founding session (DSH, default-workspace)._

## Where we are

- Day 0. Research finished, decisions made, repo created. **No code yet.**
- This folder (`~/Desktop/Oscar/ProjectX`) is a git repo with one commit.
  Not pushed to GitHub yet (Oscar will create the repo under LieuGroup).
- Decisions and the concept table live in `docs/decisions.md`. Research in
  `docs/research/`.

## Why we moved here

The founding conversation ran in a DSH session whose working directory was
`~/Documents/deepseek-harness/default-workspace`. DSH has no "move session"
feature (a session's cwd is fixed in its log header), so the work continues
in a fresh session rooted in this folder, with these docs as its memory.

## Immediate next steps (in order)

1. **One-page plan** — screens (Home/Captain, Crew, Issues, Routines, Inbox)
   and the plugin layout (which DSH packages we create, what each owns).
   Present it to Oscar, get a yes.
2. **Prove the plugin path** — build a tiny "hello" DSH plugin, load it into
   the official app from this folder (`dsh plugin add <local path>` or the
   Plugins page), confirm it can (a) add a full-page panel and (b) whether it
   can replace the main layout via the `root` slot. This decides
   "plugin vs fork" for the UI for good. See
   `docs/research/dsh-extensibility.md` for the APIs and template packages.
3. **Read `dsh-superagent-prototype`**
   (https://github.com/deepseek-ai/dsh-superagent-prototype) to see how much
   Router/Worker/Thread/Trigger is already implemented and reusable.
4. **Smallest working piece** — a Captain that talks to Oscar and keeps an
   issues board (create / assign / status), stored with `ctx.storage`.
5. **Bot provider** — wrap the `grokbot` CLI (send → job id → wait/tail) so the
   Captain can hand a thread to a Grok Bot bot. Then the first FillLight
   routine: weekly X scout for ASO / Apple Ads / TikTok tactics → digest in
   Inbox.

## Open items that need Oscar

- Stripe read access (run `stripe` outside the sandbox, or a read-only key).
- App Store Connect vendor number (for sales/downloads via `asc`).
- Rotate the X bearer token that was pasted into chat. App-level token is
  enough for search, which is the main use.
- Confirm git author name/email (first commit used a placeholder).
- Create the GitHub repo under LieuGroup and give the URL.
- `~/Desktop/Oscar/` also contains `Projectx` and `projectx` folders —
  check whether they're leftovers before deleting.

## Tools available on this Mac

`dsh`, `grokbot` (Grok Bot CLI), `asc` (App Store Connect, signed in),
`stripe` (blocked in sandbox), `xurl` (X API, app-level token), `rea`
(reverse engineering), `claude`. Grok Bot account has 10 bots, one per
product; Heardback has routines (daily growth, weekly results, Threads 4x/day).
