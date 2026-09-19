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
/plugin marketplace add https://github.com/stempelv/oh-my-claudecode-slim
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

There is no config file and no code path for changing an agent's model. Use
`/omcs-config` to write a user-scoped copy of an agent file that shadows the
plugin's own, or set `CLAUDE_CODE_SUBAGENT_MODEL` (and, to override every
agent's explicit `model:` field, `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1`,
requires Claude Code v2.1.257+) in your environment. See the `omcs-config`
skill for the full resolution order.

## Verify

1. `claude plugin validate .` passes.
2. After install, `@` typeahead lists all six agents.
3. `/orchestrate` on a two-lane task dispatches at least two background
   agents in one turn.
4. `/tasks` shows `explorer` on Haiku and `oracle` on Opus while they run.
5. `/omcs-config put fixer on haiku` writes `~/.claude/agents/fixer.md` with
   `model: haiku` and an unchanged prompt body.
6. `/clonedeps` clones into an ignored directory and leaves the project's
   own git state untouched.

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
