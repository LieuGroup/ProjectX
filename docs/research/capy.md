# Capy "Captains": reverse-engineering notes

Target: `/Applications/Capy.app`, the Electron desktop app (renderer bundle-report says build `ffebce62…`, profile desktop, channel stable).

Sources:
- https://capy.ai/llms-full.txt
- https://docs.capy.ai/captains.md
- https://docs.capy.ai/desktop.md
- https://docs.capy.ai/llms.txt (API index)

Legend:
- **[O]** observed in the shipped bundle (static strings and schemas). Static analysis only; nothing was executed.
- **[D]** stated in the official docs.
- **[I]** my inference.

Method limits:
- `app.asar` (3.2 MB, the main process) did **not** extract (`npx @electron/asar extract` exited 1), and I did not run the REA `analyze_javascript_application` pass.
- All code evidence comes from ripgrep over the unpacked renderer at `Contents/Resources/renderer/assets/*.js` (about 1000 minified Vite chunks) plus `strings` on `app.asar` and the `kappu-macos` binary.
- `/tmp/capy-re` holds no extracted content.

## 1. Big picture: the brain is in the cloud

- [D] "A thread is a durable conversation stored on Capy's servers." Captains run in the cloud by default; the machine sleeps between turns.
- [O] The renderer is a TanStack Router web app that mirrors capy.ai (routes like `/_app/project/$projectId/thread/$threadId`, `/_app/issues`, `/_app/pulls`). It talks to `https://api.capy.ai` plus realtime `wss://realtime-production.cloud.capy.ai`.
- [O] `app.asar` contains `wss://device-tunnel-production.cloud.capy.ai/tunnel`, which links the desktop to the cloud as a paired "device" so cloud-orchestrated threads can run in a local folder.
- [O] There is a bundled Rust/Tokio helper, `Resources/kappu-macos/kappu-macos` (universal Mach-O). Its strings show an accessibility/computer-use API (`status, apps, observe, run_actions, launch, quit, stop`; click, drag, scroll, key, setValue, selectText, screenshot).
- [I] Orchestration (Captain logic, issues, digest) runs server-side. The desktop is a client plus an execution device; it is not the brain.

## 2. Data model

**Thread / Jam**
- [O] The internal name for a thread is **Jam**: `jamId`, `captainJamId`, `JamDetail`, `CreateJamRequest`, `CreateSideThreadRequest`, `CreateForkRequest`.
- [O] `ThreadCard` fields: `id, primaryMachine, title, titleCustom, status, archived, threadNumber, userId, takenFromUserId, origin, projectId, slackChannel, linearIssue, pinned, member, subscribed, personal, tagIds, lastModelId, pulls[], createdAt, updatedAt, …, captain?: ThreadCaptain, crewCount?, automation?`.
- [O] `PrimaryMachine` is either `{kind:'cloud', machineId, size}` or `{kind:'device', machineId, name, access, ownerId, ownerName}`.
- [O] `JamDetail` adds:
  - `parentJamId` (subagents)
  - `runStatus` (`working|waiting|idle|done|failed`)
  - `wakePendingAt`, `todos[]`, `model`, `servedModel`, `lastReadEventId`, `slack`
  - `membership{joinPolicy, members, viewerRole}`
- [O] Sidebar status labels: `Working, Waiting, Needs you, Read, Failed`.
- [O] There is an "Ask" model (`OpenAsk`, `AskGroup`, `AskDismissRequest`): questions threads raise for humans.

**Captain**
- [O] A Captain is a **Jam that carries a captain record**, not a separate entity type. `ThreadCaptain = {jamId, name, avatar}`. `crewCount` sits on the thread card.
- [O] User preferences include `captainOrder` for sidebar ordering.
- [O] There are agent tool contracts `CaptainCreateTool` (output `{id, code, name}`), `CaptainListTool`, `CaptainGetTool` and `CaptainProfileUpdateTool`, alongside `ThreadListTool`. These are tools exposed to agents.
- [D] Captain attributes: name, face/avatar, alias, project, machine ("Run on"), model, access (whole org or specific people), owner, and "Crew runs on" (cloud or one paired device).
- [D] A Captain always acts as its owner, using the owner's GitHub access and keys.
- [D] **Crew**: the threads a Captain runs. Every crew thread must be in the Captain's project, may not be a subagent or a Captain, and can belong to only one crew at a time. Crew threads inherit the Captain's access.
- [O] Usage reporting has `CrewUsageThread{jamId,title,archived}` in `ThreadUsage.crew`.

**Captain Drive** [D]
- `AGENTS.md`: the Captain's own rules.
- `CREW.md`: rules every crew thread reads on every turn.
- `NOTES.md`: the plan.
- `DIGEST.md`: the digest.
- `skills/`, `reports/`, `renders/`, `handoffs/`.

[O] The Drive appears as `CaptainDriveFileView` / `CaptainFilesView`, keyed by `captainJamId`.

**Issue** [O] (`IssueSummary`)
- Fields: `id, code ("PREFIX-n"), number, projectId, title, status, stateId, priority, dueAt, parentIssueId, visibility, accessJamId, assignee, assignees[], labels[], blockedBy[], blocking[], blockingCount, subIssueCount, closedSubIssueCount, openBlockers, pulls[], viewer, createdAt, updatedAt, closedAt`.
- Issue list query params: `assigneeIds`, `assigneeJamIds`, `assigneeCaptains`, `creatorJamIds`, `creatorCaptains`, `captain=<jamId>`, `scope=for_you`, `mentioned`. A `mentionCount` endpoint also takes `captain`.
  - [I] Issues can be assigned to **people or to threads/Captains (jams)**, and Captains can create them.
- [D] API: per-org issue states (custom, ordered, with a default), labels, prefix, relations (blocks, parent/sub), comments, timeline, and links to PRs or threads.

**Project** [O]: has repositories, dev-environment builds, env vars, MCP servers, settings, and drive/volume.

## 3. Lifecycle

1. [D] **Create** a Captain three ways:
   - "New Captain" dialog;
   - "Promote to Captain" on a thread, which keeps its history, PRs and machine (promoting a folder takes its threads into the crew);
   - by asking any agent, which calls `CaptainCreateTool` [O].
   - In the first message the user states its purpose, and the Captain writes that into `AGENTS.md`/`NOTES.md`.
2. [D] **Spawn**: the Captain starts crew threads; each is a full thread with its own subagents and VM or worktree. Users can also add threads by drag-and-drop, "Move to", "Add to Captain" (on folders), or a chat request ("Take CAPY-412 into your crew"). [O] `captainDrag` / `captainDropActive` and the palette strings `Add to Captain` and `Remove from crew` confirm this in the code.
3. [D] **Assign/track**:
   - The Captain is woken whenever a crew thread finishes a turn or needs something. It reads the report and updates `NOTES.md`.
   - It rewrites `DIGEST.md` (sections Needs you / New / In progress) whenever crew state changes.
   - It pastes Needs-you items into its chat and @-mentions the owner, which sends a notification.
   - [O] The `issues_needs_you` section in the Captain's view and the `captain` issue filters imply the Captain also tracks issues [I].
4. [D] **Review/merge**:
   - Crew threads open PRs. The Review agent posts structured findings, and a Captain triages them: it marks false positives and sends real issues back to a build thread.
   - The user replies "merge capy#3780" and the Captain does it.
   - The PR page can be filtered by Captain [O: `captainIds` in `pulls-*.js`].
5. [D] **Wake**:
   - by automations (cron, webhook, GitHub/Slack events);
   - from Slack with `@Capy captain=<name>` (interrupts unless `!queue` or `!steer` is given);
   - through the API, by messaging the Captain's thread id. The API also has `GET /captains/{id}/crew` and `/digest`.
6. [D] **End**:
   - The Captain archives or releases crew threads itself when their work is done.
   - Demote turns it back into a thread.
   - Archive puts the Captain and its crew to sleep.
   - Delete returns the crew to the thread list.
   - Captains can read and message each other.

## 4. Isolation

- [D] Cloud threads run on an isolated Ubuntu VM (subagents may share their parent's VM or boot a fresh one), with the repos cloned and project snapshot restore.
- [D] Local threads run in a "New worktree" (the default), an existing worktree, or the current checkout. Worktrees live under `~/.capy/worktrees`.
- [O] `app.asar` has `worktrees.ensure/list/get` and `worktree_ensure` strings.
- [D] "Continue on" moves a thread between cloud and devices, carrying the branch, unpushed commits, and staged/unstaged/untracked changes, but not ignored files, untracked files over 2 MB, or running processes. Device-to-device moves relay through the cloud.
- [D] Offline local threads can be auto-moved to the cloud.

## 5. UI layout (static: strings, routes and docs; no screenshots taken)

- **Left sidebar** [O/D]
  - Nav is customisable (`sidebarNav`, `customize-sidebar-dialog`).
  - Captains are listed with an avatar (face) in a user-orderable group (`captainOrder`), with crew threads under each Captain. Crew threads can't go in folders.
  - Then pinned threads, folders, and threads with status (Working / Waiting / Needs you / Failed).
  - Top-level pages: Projects, Issues, Pulls, Automations, Drive, Memory, Explore, Settings.
- **Captain view** [O/D]: the Captain's chat, plus tabs **Overview / Digest / Crew / Issues / Drive (Files) / Profile**.
  - Overview shows the top 5 "Needs you" items and the crew list.
  - **Crew threads open beside the Captain chat** as a split pane (`thread-view` crew row component with `captainJamId`, `active` and `preview`).
  - The menu holds Share, Demote to thread, Archive, Delete, and Remove from crew.
  - The command palette exposes New Captain, Promote to Captain, and Add to Captain.
  - Profile holds name, face, alias, and "Crew runs on" (a machine picker with a connected dot).
- **Thread view** [O/D]:
  - chat transcript with todos;
  - a machine chip offering "Continue on";
  - pull requests, subagents, model picker, and terminal/desktop requests.
- **Issues** [O]: a list/board with toolbar filters (assignee incl. Captains, creator, Captain, priority, due, state, labels), issue pages at `/issues/$code` with Blocked-by/Blocking chips and sub-issue counts, and org-configurable issue states.
- **Pulls** [O]: a list filterable by author, repo, label and Captain, and a PR page at `/pull/$owner/$repo/$prNumber`.

## 6. Reusable takeaways [I]

- Model a Captain as an ordinary thread plus a flag, and crew as a `thread.captainThreadId` (one crew per thread). Promote/demote is then cheap.
- Persist the orchestrator's state as plain markdown files: rules, crew rules (injected into every child turn), plan, digest.
- Wake the orchestrator on child turn-end or a "needs input" event, and regenerate the digest rather than append to it. Push only "Needs you" items to the human.
- Make issues assignable to people *or* agent threads, with blockedBy/parent relations, PR links and custom states.

## Unverified / gaps

- I did not inspect main-process code (the asar wasn't extracted), so I don't know the device-tunnel protocol, how `kappu-macos` is used, or how local agent processes are spawned.
- I didn't find the Captain system prompt or the server-side wake/digest logic. Those run in the cloud.
- I didn't confirm whether Captains create Issues automatically. The API and filters exist, but I found no explicit Captain→issue-creation code.
- The UI description is not visually confirmed (the app wasn't run).
