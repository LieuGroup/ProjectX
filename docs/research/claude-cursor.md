# Claude desktop "Projects" and Cursor agent organisation: static RE notes

Date: 2026-11-04. Targets: `/Applications/Claude.app/Contents/Resources/app.asar` (52.6 MB, mtime Oct 22) and `/Applications/Cursor.app/Contents/Resources/app/out/vs/workbench/*.js` (Cursor 3.x; it has `workbench.glass.main.js`). Neither app was run or changed. Extracted copies are in `/tmp/cc-re/claude`. I extracted only the JS, JSON and HTML members by parsing the ASAR header with Python.

Labels: **[O]** means I saw it in the shipped code or strings. **[D]** means it comes from official docs. **[I]** means I inferred it.

**Limits of this analysis.** `rea analyze-javascript-application` (rea 6.4.0) ran on the Claude ASAR but returned `cleanup_incomplete` (`execution_failure`) instead of a usable result. The only Evidence ID it produced is `ev_bb7cc8400543d0cc0c4c57376b34dabf8d8d0d8a4230eaf87a79f6b8ff80463a`. I did not run REA on Cursor, which ships unpacked (`out/`) rather than as an ASAR. So the evidence below comes from literal string and grep searches over minified bundles. The file paths and code snippets are real, but none of this shows runtime behaviour, and the bundles are minified.

---

## 1. Claude desktop

### Data model
- **[O] A Project is a cloud object with these parts: uuid, name, description, `prompt_template` (the custom instructions), docs, files, project memory, and syncs.**
  - `[ProjectSync]` in `.vite/build/index.chunk-jPKPzV-P.js` calls `/api/organizations/{org}/projects/{id}`, `/projects/{id}/docs`, `/projects/{id}/files`, `/memory?project_uuid=` and `/projects/{id}/syncs`.
  - It then writes them to a local cache: a docs dir, a files dir (`{file_uuid}.{ext}`) and `metadata.json {uuid,name,description,prompt_template,synced_at}`.
- **[D] On claude.ai, a Project is a workspace with its own chat history, knowledge base and custom instructions** ([Claude Academy](https://academy.claude.com/courses/claude-101/introduction-to-projects)).
- **[O] Chats link to a project by URL:** `${base}/project/${projectId}` (route builder), `uu.OpenProject` deep link → `/project/<uuid>`, and `signProjectThreadLink`.
- **[O] Local agent features ("Cowork" / "Code") add a second container called a Space.** IPC methods include `addFolderToSpace`, `addProjectToSpace`, `removeProjectFromSpace`, `addLinkToSpace`, `getAutoMemoryDir(spaceId)`, and migration loops over `o.folders`, `o.projects` and `o.links`.
  - Sessions carry `spaceId`, `cwd`, `userSelectedFolders`, `userSelectedFiles`, `userSelectedProjectUuids` and `permissionMode`.
  - A context builder for `case "project"` puts together `CLAUDE.md` plus the space's auto-memory as sources.
  - **[I]** Space = local folders + linked cloud Projects + links + memory. A cloud Project is the knowledge/instructions part, which gets pulled down locally for agent sessions.
- **[O] Session APIs:** `LocalAgentModeSessions` (start/list/…; 294 references), `isLinkedProjectThreadSession`, `readChatTabGroupConver…`, and scheduled tasks (`cronExpression`, `fireAt`, `notifySessionId`, `madeByDispatch`).
  - "Dispatch" is a coordinator that creates sessions or tasks. **[I]** It is a lightweight orchestrator.

### Isolation
- **[O] Local agent sessions can use git worktrees.** `[createWorktree]`, plus checks that "git worktree add did not land in the repository on the session's record". A git-argument guard rejects `worktree add` without `--no-checkout` in one path.
- **[O]** A tool-safety prompt treats "a git clone, a git worktree add or a mkdir" made by the session as that session's own workspace.
- **[O]** `setFocusedCwd`, `setFocusedSshTarget` and `setFocusedRemoteTarget` point to targets that can be local, SSH or remote.

### UI
- **[O] Top-level mode strings:** "Chat", "Cowork", "Code", "Projects", "Scheduled", "Dispatch", "New task", "New session".
- **[I/D] Layout:**
  - Left sidebar with mode switcher, New chat/task, Projects, Scheduled, and a recents list.
  - The Projects screen is a grid or list of project cards.
  - A project page shows its chats in the main column and a right panel for Instructions and Knowledge (files).
  - Cowork adds a folder/project context panel: [Cowork help](https://support.claude.com/en/articles/15520349-use-claude-cowork-on-web-desktop-and-mobile), [issue #40043](https://github.com/anthropics/claude-code/issues/40043).

### Local vs cloud
- **[O]** Projects, chats and memory live in the cloud and are cached locally. Cowork/Code agent sessions, worktrees and scheduled tasks run on the local machine (`LocalAgentMode`). The renderer also references remote session verification (`isVerifiedCoworkRemoteSession`).

---

## 2. Cursor

### Data model
- **[O] The unit of work is the composer** (chat/agent thread, keyed by `composerId`, 2849 refs). It has `composerData.gitWorktree.worktreePath`.
- **[O] Commands and settings:** `composer.createNew`, `cursor.composer.subagentModel`, queueing settings, and `bestOfNModelName` (parallel best-of-N runs per model).
- **[O] Background/Cloud agents:** `backgroundComposer` (935 refs), `workbench.view.backgroundAgent`, `addAsyncFollowupBackgroundComposer`, and strings such as "Cloud Agents", "Cloud subagent was aborted" and "Cloud agent submit failed".
  - **[D]** Cloud Agents used to be called Background Agents ([docs](https://cursor.com/docs/cloud-agent)).
- **[O/D] Projects → coordinator → agents.**
  - Strings in `workbench.glass.main.js`: "New Project", "Projects", "Add to Project", "Archived", "In Progress".
  - **[D]** ([Projects](https://cursor.com/docs/agent/projects)):
    - A Project is a long-lived chat with a coordinator agent. The coordinator does not write code itself. It plans the work and starts agents in parallel.
    - You pick a workspace (repo) and model.
    - Shared context files sync to every machine the agents use.
    - Subscriptions (Slack, schedule, PRs) show up as a **Listening** pill.
    - Child chats appear under the project row and in an **Agents** list above the chat input.
    - You can add or remove a chat with drag-and-drop or the "Add to Project" menu item.

### Isolation
- **[O] Worktree module (`worktree.js`):**
  - Ownership `cursor-managed`, `retentionPolicy: taskScoped`.
  - Functions and services: `worktreeManagerService`, `worktreeCleanupMainService` ("Worktree Cleanup"), `applyWorktreeToCurrentBranch` (and `…ViaMerge`), `openWorktreePR`, `isUndoingWorktree`, `reservedWorktree`.
  - `isGlassCursorWorktreeAgentWorkspace` and `glassWorkspaceLspWorktreesEnabled` exist; the first is `false` in the base env.
  - **[I]** Each local agent gets a temporary git worktree, and the user applies, merges or opens a PR when done.
- **[D] Cloud agents run on their own VM and branch.** `/in-cloud` hands work off to a cloud subagent, and the agent can move between local and cloud ([Agents Window](https://cursor.com/docs/agent/agents-window)).

### UI
- **[O] There is a separate "Agents Window" ("glass" mode):**
  - Keys: `cursor.openGlassModeWindow`, `cursor.switchToGlassWindow`, `glass.enable_open_agent_in_window`.
  - It has an **Agents Side Bar** that can sit on the left or right ("Toggle Agents Side Bar", `agentsSidebarOnLeft`, `unifiedSidebar`).
  - Default agent title is "New Agent". Labels: Local / Worktree / Cloud / Remote.
- **[D/I] Layout:**
  - Left sidebar: workspaces (multi-repo), then Projects (rows with nested agent chats), then an agent list grouped by status (In Progress / Archived).
  - Center: the agent chat thread.
  - Side or tab: a diffs/review pane (commit, PR), with file search via Cmd+P.
  - You can still open the classic IDE (Composer panel on the right side of the editor) with "Open IDE".

### Local vs cloud
- **[D]** Agents can run locally (in the workspace or a worktree), on remote SSH, or in the cloud (a VM, which can also be self-hosted). Projects are cloud-first, and the coordinator starts a local agent when something needs testing on your machine.

## Takeaways for our harness
- **Claude:** Project = instructions + knowledge files + memory + chats. Local agent work adds a Space (folders + projects) with worktrees.
- **Cursor:** Project = a coordinator chat that owns child agent threads. Each thread is isolated (worktree or VM) and is tracked in a status-grouped sidebar, with a diff/review pane.
