# oh-my-zcode-slim

An [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)-equivalent agent orchestration suite for **ZCode**, packaged as a ZCode plugin. It turns the primary ZCode session agent into an orchestrator that delegates to specialist subagents — balancing quality, speed, and cost — with a configurable multi-model council for high-stakes decisions and OMOS's workflow skills.

## Credits

This suite is a port of **[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)** by **[@alvinunreal](https://github.com/alvinunreal)** — [ohmyopencodeslim.com](https://ohmyopencodeslim.com). The specialist roster and agent prompts (explorer, librarian, oracle, designer, fixer→coder, observer, council/councillors), the orchestration doctrine and its routing table, and the skills (`simplify`, `verification-planning`, `deepwork`, `clonedeps`, `worktrees`, `reflect`) are adapted from their work under MIT. Go star the original.

The ZCode-specific parts — plugin packaging, the seat/MCP configuration scripts, the ZCode doctrine mechanics, and the porting notes in [docs/LESSONS.md](docs/LESSONS.md) — are this repo's. Licensed under MIT (see [LICENSE](LICENSE)), which carries the upstream notice.

Everything machine-specific (model providers, MCP servers, council seats) is **yours to configure** — the plugin ships only what works everywhere.

## What's in the suite

| Layer | Contents |
|---|---|
| **Plugin** (7 agents) | `coder`, `explorer`, `librarian`, `oracle`, `designer`, `observer`, `council` |
| **Plugin skills** (6) | `simplify` (auto-mounted on oracle), `verification-planning`, `deepwork`, `clonedeps`, `worktrees`, `reflect` |
| **Plugin commands** (2) | `/deepwork`, `/reflect` |
| **Doctrine** (`doctrine/AGENTS.md` → `~/.zcode/AGENTS.md`) | Orchestrator role, routing table, delegation mechanics, seat-agnostic council protocol, skill routing |
| **Council seats** (yours) | `councillor-*` agents you generate with `scripts/add-councillor.sh` — any models you have |

Default model pins use builtin refs (`custom:builtin%3Azai-coding-plan:GLM-5.3` / `GLM-5.3-Flash`), available to every z.ai-authenticated ZCode session. Shipped agents declare **no MCP servers** — see [MCP configuration](#mcp-configuration) before adding any.

## Install

```bash
git clone https://github.com/MartijnDekkers/oh-my-zcode-slim.git
cd oh-my-zcode-slim
```

1. **Plugin**: ZCode → Settings → Plugin Management → Discover → **+** → add this repository as a marketplace (in remote/SSH setups, add it by **git URL** — see [Remote (SSH) setups](#remote-ssh-setups)) → install and enable `oh-my-zcode-slim`.
2. **Doctrine**: `./install-doctrine.sh` (backs up any existing `~/.zcode/AGENTS.md` to `.bak`). Plugins cannot contribute AGENTS.md files, so this one file installs separately.
3. **Council seats** (optional but recommended): `./scripts/list-models.sh` to see model refs proven to work on your machine, then one command per seat:

   ```bash
   ./scripts/add-councillor.sh glm53 'custom:builtin%3Azai-coding-plan:GLM-5.3'
   ./scripts/add-councillor.sh mymodel 'custom:<provider>:<model>'
   ```

   Seats land at user scope (`~/.zcode/agents/`) or, with `--workspace <path>`, at project scope for a per-repo council. Two or more seats make the council protocol live; the doctrine refuses to fake a consensus below that.
4. Restart your ZCode session (agents, skills, commands, and AGENTS.md load at session start).

## Configuring your council

The council is whatever `councillor-*` agents exist in your session — the orchestrator dispatches every one of them in parallel, then the `council` agent synthesizes a structured consensus report (consensus level, agreed/disputed points, recommendation). Seats are plain markdown files; the script is sugar:

- Add: `./scripts/add-councillor.sh <seat> <model-ref>` (overwrite with `--force`)
- List proven model refs: `./scripts/list-models.sh`
- Remove: `./scripts/remove-councillor.sh <seat>`
- Diversity is the point: seats on different providers/models give the council its value. Same-model seats just cost tokens.

### Model pin rules (hard-won — see docs/LESSONS.md)

- Pin format: `custom:<provider-id>:<model-id>`; provider ids are **case-sensitive**.
- In remote-attached sessions, custom providers materialize under **UUID provider ids**, not display names — `list-models.sh` prints the ids that actually resolve on your machine.
- A bad pin fails loudly at spawn ("Model provider is not configured"), which is the intended behavior — ping your seats after adding them.

## Overriding shipped agents

Same-named agents at user scope (`~/.zcode/agents/`) or workspace scope (`<repo>/.zcode/agents/`) take precedence over plugin agents. Copy `oh-my-zcode-slim/agents/<name>.md` out, edit the `model` pin or prompt, and your copy wins. That is also how you version a per-project variant of any specialist.

## MCP configuration

Agents may declare `mcpServers` in frontmatter — but note: **every listed server must be connected at spawn time, or the agent refuses to start** (and fails again whenever that server is down). That is why shipped agents declare none. To give an agent MCP-backed lookup tools:

```bash
# copies the plugin agent out to user scope (your copy overrides the plugin's)
# and wires the servers in; existing user files are edited in place
./scripts/enable-mcp.sh explorer Terraform codegraph "MS Learn"
```

Or by hand: copy `oh-my-zcode-slim/agents/<name>.md` to `~/.zcode/agents/` (or `<repo>/.zcode/agents/`), add the frontmatter line `mcpServers: [<ServerName>, ...]` using the exact server keys from your `~/.zcode/cli/config.json` (or the workspace equivalent).

Either way, accept the availability coupling: an agent is only as available as its listed servers — or leave MCP off and let the orchestrator, which has every server, run lookups itself and paste findings into delegation prompts.

## Remote (SSH) setups

If your ZCode desktop attaches to a remote workspace over SSH — e.g., a laptop UI attached to a home server — the **server and everything it reads live on the remote host**, and that changes several defaults:

- **Run the scripts on the remote host.** `install-doctrine.sh`, `add-councillor.sh`, and friends write to the `~/.zcode` of the machine running the ZCode server. Inside an SSH session that's the remote host — which is what you want. Keep the repo clone on the remote host too.
- **Add the marketplace by git URL, not local directory.** The desktop's directory picker browses your laptop's filesystem, but the chosen path is resolved on the remote host — a local-directory marketplace fails with `Marketplace source path does not exist`. A GitHub/git marketplace source installs correctly in remote mode.
- **Plugin updates may need a nudge.** The app's refresh does not reliably pull marketplace clones; restart the app, or remove and re-add the marketplace, to pick up a new version.
- **Custom model providers materialize under UUID ids.** Providers you define in the desktop app reach the remote runtime under generated UUID provider ids, not their display names — a pin like `custom:OpenCode:qwen3.8-max` will *never* resolve in a remote session, even though the model runs fine when you pick it in the UI. Run `./scripts/list-models.sh` **on the remote host** and pin from what it prints; UUIDs are per-machine, so never copy someone else's.
- **Don't trust a model's self-identification.** Ask a switched session "what model are you?" and it may recite its session-start context — a provider switch mid-session is invisible from the inside. The remote usage DB (`~/.zcode/cli/db/db.sqlite`, table `model_usage`) is ground truth.

The full debugging narratives behind these rules (with log and database evidence) are in [docs/LESSONS.md](docs/LESSONS.md).

## Smoke tests

Run in a fresh session:

1. **Agent discovery** — the subagent list shows the 7 plugin agents (`oh-my-zcode-slim:*` with bare aliases) plus your `councillor-*` seats.
2. **Councillor ping** — dispatch each seat with a trivial prompt; failures name the provider/model that didn't resolve.
3. **Role behavior** — ask coder for a styling change (it refuses and points to designer); ask explorer "where is X" (file:line list); run a council question (required report sections); confirm oracle loads `simplify`.
4. **Suite preference** — recon questions dispatch `@explorer`, never the built-in `Explore`.
5. **Skills** — `/deepwork` and `/reflect` appear in the `/` menu.

## OMOS → ZCode mapping

The orchestrator is not an agent file: in ZCode the **primary session agent is always the orchestrator** (it owns the Agent/delegation tool; a subagent cannot delegate further). The doctrine installed at `~/.zcode/AGENTS.md` plays the role of OMOS's orchestrator system prompt.

| OMOS concept | ZCode equivalent |
|---|---|
| `orchestrator` primary agent | Main session agent + doctrine in `~/.zcode/AGENTS.md` (subagent guard included) |
| `fixer` | `coder` |
| Background tasks + `task_status`/`task_result`/`task_cancel`/`task_revive`/`task_message` | `run_in_background: true`, `TaskOutput`, `TaskStop`, messaging/resuming an agent via `SendMessage` + its agent id |
| `wait_for_user` | `AskUserQuestion` / asking the user in-turn |
| `ast-grep` | codegraph MCP if you configure it (`codegraph_*` tools) |
| `observer` vision routing | `observer` agent using ZCode's image-capable Read tool |
| Council seats with diverse models | `councillor-*` agents generated per user/project, `council` synthesizer |
| Presets (`/preset`, model config JSON) | Static `model` pins + user-scope overrides, versioned in git |
| Multiplexer panes / Companion status app | ZCode desktop UI (task list, completion notifications) |
| Hook-driven wake on background completion | Native background-task notifications |

## Not ported from OMOS, and why

- **codemap** — codegraph MCP (live index) supersedes it for repository navigation; the markdown-atlas artifact can be ported later if wanted.
- **Meta-config skill & `/preset`** — this suite is static markdown + git; the README replaces the config skill, commits replace presets.
- **`/interview`** — needs OpenCode's localhost session API; ZCode's `AskUserQuestion` covers structured Q&A.
- **`/loop`** — the OMOS runtime is plugin TypeScript; the orchestrator can run execute-verify loops natively. A markdown-only `/loop` can be added later if missed.
- **`gh_grep` MCP** — no hosted endpoint to point ZCode at; add your own code-search MCP and list it per agent if wanted.
- **Hooks** (orchestrator-wake, phase-reminder, …) — covered by native background-task notifications and skill text.
- **`smartfetch`, `acp_run`** — WebFetch/WebSearch cover fetching; ZCode has no ACP subsystem.

See [docs/LESSONS.md](docs/LESSONS.md) for the debugging notes behind these rules (remote provider UUIDs, MCP hard-requirements, marketplace update behavior, and why you should never trust a model's self-identification).

## Updating

Edit → bump `version` in `oh-my-zcode-slim/.zcode-plugin/plugin.json` → commit → push. Plugin updates go through the marketplace in the app (note: the app's refresh may not pull marketplace clones — restart the app or remove/re-add the marketplace if an update doesn't surface). Doctrine changes: re-run `./install-doctrine.sh`. Your `councillor-*` files live outside the plugin and survive every update untouched.
