# DSH: owning the `root` slot, and subagent provider backends

Source: installed app (`/tmp/dsh-src/dsh/node_modules/@deepseek-ai/`, compiled `lib/`, no `src/`) plus the upstream repo for `.ts` types. Paths below are relative to that folder. Nothing was modified.

## Q1 — Can a plugin own the whole page?

**Verdict: yes-with-caveats.** The engine is built for exactly this: `root` is a built-in slot, the AppFrame is just an ordinary occupant, and all data services live outside the layout plugin.

### (a) How AppFrame occupies `root`
- `SlotCore` constructor declares `root` as a **built-in `single` slot, scope `root`** (`dsh-web-frontend/dist/assets/index-BXV-oxvX.js`, `var SI=class{…constructor(){const e=this.record("root");e.spec={kind:"single",scope:"root"}…}`).
- `dsh-client-ui-layout` does one `ctx.slots.register({ name: "root", children: { sidebar, main (keyed), rightbar, shell.bottom, shell.overlay (list), shell.leading }, store }, AppFrame)` after `ctx.reflect.provide("layout", …)` — `dsh-client-ui-layout/lib/client.js:621-653`. Its only injects are `slots, theme, locale, shortcuts` (`:589-594`).
- The renderer does the single ctx-level render: `buildRenderApp` → `ctx.slots.renderSlot("root", {})` (`dsh-client-ui-renderer/lib/client.js:1285-1288`); `RootOutlet` renders `host.entriesOfSlot("root")[0]` inside an error boundary (`:1233-1259`). No root registration = hard throw (`:1534`).

### (b) Third-party `root` occupant; two registrants
- Any `dsh.client` plugin can call `ctx.slots.register({ name: "root", children: {...} }, MyApp)` — the slot is pre-declared, no parent needed.
- **Same priority twice → throws** `single slot "root" already has a registration at priority N … register at a different priority to shadow it (lowest renders)` (SlotCore `register`, `case "single"`). **Different priority → lowest number wins** (entries sorted by `priority ?? 0`; `entriesOfSlot` picks the first live one). So a plugin registering `root` with `priority: -1` **shadows AppFrame without disabling it**; if the winner throws it "abdicates" and the next entry renders (`renderer :1248`, `abdicate: true`).
- Caveat: the shadowed AppFrame's child slots (`main`, `sidebar`, …) are still *declared* by its entry, so `ui-conversation`, `ui-sidebar`, etc. still register into them harmlessly; they just never render. `renderSlot(key)` only works for keys **you** declared in your `children` (`renderer boundRenderSlot`, `SlotOwnershipError`), so your root cannot render their `main`/`sidebar` — you must declare your own slot names (e.g. `px.main`) if you want others to plug in.

### (c) Disabling `ui-layout` / `ui-sidebar` rows
Rows live in `dsh-web-app/cordis.patch.yml:266-280` (`ui-layout`, `ui-sidebar`). Patch `- id: ui-layout\n  disabled: true` in the profile works, **but** `ui-layout` also provides `ctx.layout`, which these inject (hard dependency → they stay inactive until it exists):

| injects `layout` | what breaks if absent |
|---|---|
| `dsh-client-ui-sidebar`, `dsh-client-ui-sidebar-right` | sidebars (you're replacing them anyway) |
| `dsh-client-ui-workspace` → provides `uiWorkspace` | **ui-conversation, ui-chat, ui-subagent inject `uiWorkspace`** → chat UI never activates |
| `dsh-client-ui-plugin-manager` | Plugins page |
| `dsh-client-ui-open-in-app` | "Open in…" button |

Not dependent on layout (keep working): `sessions` (`dsh-api-session-controller/lib/client.js`, injects `connection, fileUpload, typert, remote…`), `remote` (`dsh-api-gateway`), `workspaces` (`dsh-api-workspace-controller`), `uiSession`, `configForms/settingsSchema` (`dsh-client-ui-settings`, injects only `remote, remote.settings`), `theme`, `locale`, `shortcuts`, `modules`, HMR. Session store / streaming / settings RPC are independent of the chrome. **Recommendation: do not disable `ui-layout`; shadow it with a lower priority** (keeps `ctx.layout`, `uiWorkspace`, and the conversation stack alive for later reuse). Optionally disable `ui-sidebar`, `ui-sidebar-right` only.

### (d) Services a custom root can use
| Service | Key facts |
|---|---|
| `ctx.sessions` | `retain(target,{source})`/`using()`, `binding(id)`, `createSession`, `fork`, `prompt(content, mode, signal, requestId)`, `cancel`, `rename`, `search`, `SessionEventStream` with `loadOlder/loadThrough`, projection stores (`dsh-api-session-controller/README.md:30-82`) |
| `ctx.remote` / `remote.*` | typed RPC + streams to host (`dsh-api-gateway`); `remote.session`, `remote.subagents`, `remote.settings`, `remote.workspace`, `remote.pluginManager` |
| `ctx.workspaces` | workspace projection + navigation |
| `ctx.slots` | `register`, `inject`, `registerFactory`, `provideRoot` (global hooks), `snapshot()` for introspection |
| `ctx.layout` | `selectPanel`, `beginNavigation`, `toggleSidebar`, `openRightbar/closeRightbar`, `panelInfo` — only if ui-layout is alive (meaningless when shadowed, but harmless) |
| `ctx.configForms` / `remote.settings` | read/write settings sections, schema-driven forms |
| `uiSession`, `uiWorkspace`, `uiConversation`, `conversation` | optional higher-level helpers if their rows stay enabled |

**Minimal recipe:** client plugin (`dsh.client`, `platform: web`, exports `./client`) with `inject = ["slots","sessions","remote","locale","theme"]`; in `apply`: `ctx.slots.register({ name:"root", priority:-1, children:{ "px.main":{kind:"keyed",scope:"root"}, "px.overlay":{kind:"list",scope:"root"} } }, PxApp)`. Install via `file:`/`link:` in the profile (per `dsh-extensibility.md`). Nothing to disable. Risk: the `priority` shadowing rule is in minified shell code, not README — re-verify on each DSH upgrade.

## Q2 — External agents as workers (subagent providers)

### What's installed vs upstream
| Package | Installed? | Role |
|---|---|---|
| `dsh-subagent` | yes (`dsh-base` row `subagent`) | seam: `ctx.subagents` provider registry + manager |
| `dsh-subagent-spawn-in-process` / `-fork-in-process` | yes (`spawn`, `fork`) | local continuable children |
| `dsh-tool-subagent` (`subagent`, `subagent_fork`), `dsh-tool-subagent-control` | yes | model tools; `send_message/interrupt_agent/list_agents` |
| `dsh-acp`, `dsh-acp-app` | yes | ACP **server** (DSH as the agent), not a client |
| `dsh-subagent-acp`, `dsh-subagent-claude-code`, `dsh-subagent-codex`, `dsh-subagent-dsh-sdk` | **no** — upstream only | ACP client / Claude Code SDK / Codex / DSH-SDK one-shot providers (`dsh-agent-preset/skills/cordis-composition-reference/references/packages.md:446-449`; [subagent-acp README](https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/packages/subagent/subagent-acp/README.md)) |

`dsh-subagent-acp` config: `providerName, command, args, permission: allow|reject, env, disposeEofGraceMs, disposeGraceMs`; spawns a fresh process per run, does ACP `initialize`+`newSession`, collects `agent_message_chunk` text, maps `end_turn→completed, cancelled→aborted, …`. It is **one-shot**: no follow-up messages (README "Known Limitations"). Being an upstream package it can be installed into the profile like any other plugin, or copied as the template for ours.

### Provider contract (`SubagentProvider`, upstream `packages/subagent/subagent/src/types.ts`; enforced in `dsh-subagent/lib/index.js:3256-3328`)
| Member | Meaning |
|---|---|
| `name: string` | registry key (`spawn`, `fork`, `acp`, → `grokbot`); duplicate → `DUPLICATE_PROVIDER` |
| `capabilities: { agentOptions, agentPreset, outputSchema, depthLimit }` | manager rejects a request needing an unsupported flag (`UNSUPPORTED_CAPABILITY`); externals use `NO_START_CAPABILITIES` (all false, `:2789`) |
| `inheritsParentContext: boolean` | descriptive only (tool wording) |
| `agentRouteDefaults?` | optional static provider/model |
| `start?(req: ResolvedSubagentStartRequest): Promise<SubagentRun>` | **external path**. `req = { cwd (absolute), prompt: ContentBlock[], parent: Agent, label, signal, … }`. Return `{ id: SessionId (fresh uuid), result: Promise<SubagentResult>, dispose(): Promise<void> }`. `result` never rejects for child failure — resolve `{ output: ContentBlock[], stopReason: 'completed'|'aborted'|'error'|'max-tokens'|'refusal', diagnostic?, structured? }`. Helpers exported: `subprocessRunHandle`, `settleRunResult` |
| `prepareContinuable?(req): Promise<{ seed? }>` | **local path**; presence selects it. Manager then creates the Agent itself (`ctx.agents.create`, `:1815`). Not usable for a non-DSH worker |

Either `start` or `prepareContinuable` is required. Registration: `ctx.subagents.registerProvider(new P(...))` inside a plugin with `inject = ["subagents"]` (`dsh-subagent-spawn-in-process/lib/index.js:11,36`). Then one `dsh-tool-subagent` row per provider: `config: { provider: grokbot, toolName: grokbot }` (`dsh-base/cordis.patch.yml:380-395`).

**A "grokbot provider" / "ACP provider"** = a Host plugin implementing `start()`: spawn CLI (or ACP stdio), feed `req.prompt` text, stream stdout into `output`, wire `req.signal` → kill, return `subprocessRunHandle`. Caveats: externals are one-shot (no `send_message`), have no local Session (catalog `mode:'external'`), and `startActivation` consumes a slot of `maxActiveSubagents` (default 8).

### Wake mechanism (how the parent learns the child finished)
1. External: manager awaits `run.result` (`announce`, `index.js:1883-1886`) → `close()` → `notifySettlement()` (`:2040,2048-2062`).
2. `createSettlementMessage` builds a user-role notice (`"<child> finished…"`, closing text, `Structured result:`, diagnostic; `source.kind: "subagent-settled"`).
3. Delivery `sendWaking(parent, msg, parent.status === "idle" ? "queue" : "steer")` (`:2058,1631-1643`): idle parent → `parent.followup()` **starts a new turn**; busy parent → `parent.steer()` injects at the next step. `delivery:'caller'` (workflows) sends no notice; `result` promise only.
4. Observe-only events `subagent/start` / `subagent/end` (`SubagentRunEndInfo` with `stopReason`, `lastAssistantMessage`) for UI/Inbox plugins. Local children additionally wake on `agent/inbox/*` events (`:1861-1863`).
