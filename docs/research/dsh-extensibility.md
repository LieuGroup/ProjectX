# DSH extensibility: plugin vs fork for Captain / Issues / Threads

Source: the installed app.asar, extracted to `/tmp/dsh-src` (paths below are relative to `/tmp/dsh-src/dsh/node_modules/@deepseek-ai/`). The app was not modified.

## Q1: Loading local or private plugins
- The user profile is `~/.dsh/profiles/desktop/`. Its `package.json` lists `dsh.profile.bundles` as `[dsh-base, dsh-web-app, dsh-experimental-agent-team-profile]`. Per-entry enable/disable overrides live in `cordis.patch.yml`.
- `dsh-plugin-manager/README.md`:
  - You can install external bundles from the Web **Plugins** page, the `plugin_manager` tool, or `dsh plugin`.
  - Accepted specs: a package name/version, a Git URL, a tarball, an absolute local path, or `file:` / `link:` paths resolved against the profile directory (`~` is allowed).
  - With HMR on, changes apply live; otherwise they apply on restart.
  - Installed Host code runs in-process with no sandbox.
- **Answer: yes.** The official app can load private and local plugins.

## Q2: How much of the GUI a web client plugin controls
- A package's `/client` export is mounted by the Web Client loader. Example: `dsh-experimental-client-ui-agent-team/package.json` has exports `./client`.
- It uses a slot API: `ctx.slots.register({name,key},C)` and `ctx.slots.inject(...)`.
- Known slots:
  - `conversation.*` header seats (where the Agent Teams header action sits)
  - `sidebar.right.pane.tab`
  - `sidebar.workspaces.session.menu.item` / `.row.action`
  - `settings.plugins.tab`
  - `sidebar.brand.*`
- Layout (`dsh-client-ui-layout/README.md`):
  - The root-scoped `main` keyed slot holds global full-page panels (the Plugins page is one). `ctx.layout.selectPanel(id)` navigates to a panel.
  - It also offers `shell.bottom`, `shell.overlay` and `shell.leading`.
- The whole AppFrame is itself a plugin that occupies the `root` slot (renderer: `renderSlot('root')`).
- **Answer:**
  - Panels, header actions, full pages and sidebar tabs: yes.
  - Replacing the main layout or session list: possible in principle by disabling `client-ui-layout` / `client-ui-sidebar` in `cordis.patch.yml` and registering your own `root` / sidebar occupant. Not verified.

## Q3: Server-side APIs
These are inferred from the READMEs, not exercised:
- **Model tools:** `ctx.tools` registry with `ToolDefinition` (see `dsh-tool-*` packages).
- **Sessions and teammates:** `ctx.subagent` (spawn/fork), `ctx.sessions.create`, `ctx.agentTeams` (agent-team service, roster, task board).
- **Persistent state:** `ctx.storage` schema-validated domains (`dsh-storage-domain`). Records are addressed `<scope>/<id>`.
- **UI updates:** Host projection frames. agent-team publishes an `agentTeam` Session projection that the client reads from the Session store. It's a working pattern to copy for an `issues` projection.
- **Worktrees:**
  - `dsh-experimental-tool-worktree` is a model tool that creates a git worktree and switches the Session directory.
  - Agent Teams explicitly has no per-member worktree (`dsh-experimental-agent-team/README.md` Known Limitations), so the Captain plugin must combine the two itself.

## Q4: Examples and docs
- 244 READMEs with `kind: "package-reference"` frontmatter, each covering usage, implementation and a source map. There is no separate `docs/` folder.
- Best templates:
  - `dsh-experimental-agent-team-profile`: a bundle that wires a Host service, tools and a client UI.
  - `dsh-experimental-client-ui-agent-team`: a client header panel reading a projection.
  - `dsh-sdk-minimal`: a minimal profile and `cordis.patch.yml`.
- The packages ship `./src/*` exports, so source is likely included.

## Verdict
| Part | Verdict |
|---|---|
| Captain (orchestrator agent, spawns workers) | plugin (`ctx.agentTeams` / `ctx.subagent` + tools) |
| Issues (persistent tracker + tools) | plugin (`ctx.storage` + tools + projection) |
| Threads (per-issue sessions, worktree each) | plugin, but worktree-per-agent is DIY (unknown edge cases) |
| Board UI | plugin (`main` global panel / right pane tab / header action) |
| Main layout / session list replacement | unknown → probably plugin via disabling `client-ui-layout`/`sidebar` and filling `root`; fork if that fails |
