# oh-my-claudecode-slim

A slim multi-agent orchestration plugin for Claude Code. Six specialist
agents, seven skills, zero runtime code.

## What this is

`oh-my-claudecode-slim` ships six subagents (`explorer`, `oracle`,
`librarian`, `designer`, `fixer`, `observer`) and seven skills
(`orchestrate`, `deepwork`, `verification-planning`, `omcs-config`,
`codemap`, `clonedeps`, `reflect`) as plain markdown and JSON: agent prompts,
skill instructions, and two plugin manifests. There is no script, no build
step, and no dependency to install. This is deliberately a slim subset, not
a port of the full toolset: where `oh-my-claudecode` ships around 39 skills
and 19 agents, this plugin ships 7 and 6, trading breadth for a set small
enough to read in one sitting and reason about in full. And where
`oh-my-opencode-slim` (the project this one is ported from) can mix models
from different providers in the same session, this plugin cannot: Claude
Code runs on Claude models only. If you need multi-provider model mixing,
use opencode instead.

## Install

```bash
/plugin marketplace add https://github.com/vladimirstempel/oh-my-claude-code-slim
/plugin install oh-my-claudecode-slim
```

## Agents

| Agent | Model | Effort | Rights | Role |
|---|---|---|---|---|
| `explorer` | haiku | none | read-only (Write/Edit/NotebookEdit denied) | fast codebase search and pattern matching |
| `oracle` | opus | high | read-only (Write/Edit/NotebookEdit denied) | architecture decisions, hard debugging, code review |
| `librarian` | sonnet | none | read-only (Write/Edit/NotebookEdit denied) | external docs and library research |
| `designer` | sonnet | none | writes files | UI/UX design, implementation, and review |
| `fixer` | sonnet | none | writes files | bounded implementation from a complete task spec |
| `observer` | sonnet | none | read-only (Write/Edit/NotebookEdit denied) | visual analysis of images, screenshots, and PDFs |

`fixer` also carries `disallowedTools: Agent, Task`: it cannot spawn its own
subagents. Hand it a complete task and all the context it needs up front.

Claude Code registers plugin agents under a plugin-scoped identifier, so the
name to pass as `subagent_type` (and the name you will see in `/tasks` and the
`@` typeahead) is `oh-my-claudecode-slim:<name>`, e.g.
`oh-my-claudecode-slim:explorer`. The bare name does not resolve.

## Skills

| Skill | Purpose | Defers to |
|---|---|---|
| `orchestrate` | plan, delegate, and reconcile work across the six agents | none |
| `deepwork` | scheduler workflow for large, multi-phase, high-risk work with review gates | `superpowers:writing-plans`, if installed |
| `verification-planning` | build a project-specific evidence path before a non-trivial change | none |
| `omcs-config` | change an agent's model, effort, or prompt, or disable it | none |
| `codemap` | build a hierarchical map of an unfamiliar repository | `graphify`, if installed |
| `clonedeps` | clone a few important dependency sources into an ignored local directory | none |
| `reflect` | review recent work and propose the smallest reusable fix | none |

## Changing models

There is no config file and no code path for changing an agent's model. Three
things work, cheapest first:

1. **One delegation:** pass the Agent tool's `model` argument (`haiku`,
   `sonnet`, `opus`, `fable`). Nothing to write, nothing to revert.
2. **Every subagent:** set `CLAUDE_CODE_SUBAGENT_MODEL`, plus
   `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` to beat each agent's explicit `model:`
   field (requires Claude Code v2.1.257+).
3. **One agent, permanently:** fork it into `~/.claude/agents/<name>.md` and
   deny the plugin's twin with `"Agent(oh-my-claudecode-slim:<name>)"`. A
   user-scoped file alone does *not* shadow a plugin agent — the names never
   collide, so you get two agents and the plugin's copy still wins.

Run `/omcs-config` and it will do the right one. See that skill for the full
resolution order.

## Verify

1. `claude plugin validate . && claude plugin validate ./agents && claude
   plugin validate ./skills` all pass. The bare `claude plugin validate .` only
   checks the marketplace manifest — the agent and skill directories need their
   own paths.
2. After install, `@` typeahead lists all six agents. They appear scoped, as
   `oh-my-claudecode-slim:explorer` and so on — the prefix is correct, not a
   failure.
3. `/orchestrate` on a two-lane task dispatches at least two background
   agents in one turn.
4. `/tasks` shows `explorer` on Haiku and `oracle` on Opus while they run
   (the model column requires Claude Code v2.1.242+).
5. `/omcs-config put fixer on haiku` writes `~/.claude/agents/fixer.md` with
   `model: haiku` and an unchanged prompt body, and adds
   `"Agent(oh-my-claudecode-slim:fixer)"` to `permissions.deny` so only the
   fork remains.
6. `/clonedeps` clones into an ignored directory and leaves the project's
   own git state untouched.

## What this writes into your repository

"Zero runtime" means no code runs, not that nothing is written. Three skills
create or edit files in your project, always with your approval:

| Skill | Writes |
|---|---|
| `deepwork` | `.slim/deepwork/<task>.md` progress files; adds `.slim/deepwork/` to `.gitignore` and an allow-entry to `.ignore` |
| `codemap` | `codemap.md` in the repo root and in every mapped folder; `.slim/codemap.json`; appends a `## Repository Map` section to the project's `CLAUDE.md` |
| `clonedeps` | cloned repos under `.slim/clonedeps/repos/`; `.slim/clonedeps.json`; a marker block in `.gitignore` and `.ignore`; appends a `## Cloned Dependency Source` section to the project's `CLAUDE.md` |

Everything under `.slim/` except the two small JSON manifests is git-ignored.
The `.ignore` entries exist so Claude Code's search tools can still read what
`.gitignore` hides. `/clonedeps` cleanup removes the clones and the marker
blocks, and asks before touching the manifest or the `CLAUDE.md` section.

## What this deliberately does not do

- Multi-provider model mixing or a `council` command. Claude Code runs
  Claude models only. Use opencode if you need to mix providers.
- A companion window, or tmux/zellij/kitty panel integration.
- Preset switching or an interview flow.
- `smartfetch` or `ast-grep` integration.
- Its own MCP servers.
- A `simplify` skill or a `worktrees` skill: Claude Code already has native
  `/simplify` and `EnterWorktree`/`ExitWorktree`.
- Auto-update.
- An npm package. Install it as a Claude Code plugin, nothing else.

## Credits

Ported from [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)
by alvinunreal, MIT licensed. The agent and skill prompts in this plugin are
adapted from that project for Claude Code.

## License

MIT. See `LICENSE`.
