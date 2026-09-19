---
name: omcs-config
description: Configure oh-my-claudecode-slim — change which model or effort level an agent runs on, disable an agent, or override its prompt. Use when the user wants a different model for explorer, oracle, librarian, designer, fixer, or observer, or asks to tune this plugin.
---

# omcs-config

This plugin has no config file and no code path for changing an agent's
model. The only supported mechanism is shadowing: write a user-scoped copy of
the agent file that Claude Code resolves ahead of the plugin's own copy.

## How an override works

Claude Code resolves subagents by scope, highest first: managed settings >
`--agents` CLI flag (current session) > project (`.claude/agents/`) > user
(`~/.claude/agents/`) > plugin (`agents/`). A file at
`~/.claude/agents/<name>.md` shadows the plugin's version of the same agent
completely — the plugin file is not merged, it is replaced.

All three changes below (model, prompt, disable) revert the same way: delete
`~/.claude/agents/<name>.md`, or drop the `permissions.deny` rule.

### Change the model or effort

1. Read the plugin's version at `${CLAUDE_PLUGIN_ROOT}/agents/<name>.md`.
2. Write the whole file to `~/.claude/agents/<name>.md`, changing only the
   `model:` (or `effort:`) line. Copy the prompt body verbatim — this
   procedure is a model/effort swap, not a rewrite, so don't paraphrase it.
3. Tell the user the override is now user-scoped: plugin updates will no
   longer reach that agent until they delete the file.

### Override the prompt

Same shadowing mechanism, different intent: write
`~/.claude/agents/<name>.md` with the user's own prompt body in place of the
plugin's. Start from the plugin's frontmatter and carry every field forward
unless the user is deliberately changing it — don't reduce the list to
whatever the new prompt happens to need. This matters most for `tools` and
`disallowedTools`: those are permission boundaries, not preferences, and
dropping them silently restores capability the plugin deliberately removed.
`fixer` ships `disallowedTools: Agent, Task` — the line that stops it
recursively delegating; lose it here and a rewritten `fixer` can spawn
subagents with no error to warn anyone. Unlike the model swap above, the
prompt body itself is a deliberate rewrite — ask the user what it should say
rather than inventing or paraphrasing one yourself.

### Disable an agent

This isn't file-shadowing — it's a permission rule. Add `"Agent(<name>)"`
(e.g. `"Agent(fixer)"`) to the `permissions.deny` array in a settings file:
project `.claude/settings.json`, project-local `.claude/settings.local.json`,
or user `~/.claude/settings.json`. Ask the user which scope they want; user
scope is usually right since it won't affect teammates on a shared project.
One-off session alternative: `claude --disallowedTools "Agent(<name>)"`.

## Session-wide alternatives

- `CLAUDE_CODE_SUBAGENT_MODEL` (in the `env` block of a settings file) only
  affects subagents that don't set their own `model:` frontmatter. It does
  **not** move this plugin's six agents by itself — every one of them sets
  `model:` explicitly, and Claude Code's resolution order puts a subagent's
  own frontmatter ahead of this variable (per-invocation parameter >
  frontmatter `model:` > `CLAUDE_CODE_SUBAGENT_MODEL` > main conversation's
  model).
- To force every subagent — including this plugin's — onto one model
  regardless of its `model:` field, also set `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`
  to `1`. Requires Claude Code v2.1.257+. Set without
  `CLAUDE_CODE_SUBAGENT_MODEL`, it falls back to the main conversation's
  model instead of forcing anything useful.
- Verify what a running subagent uses: run `/tasks`; the row names the
  model, and the effort level when one is set.

## Defaults shipped by this plugin

| Agent | model | effort | Writes? |
|---|---|---|---|
| explorer | haiku | — | no |
| oracle | opus | high | no |
| librarian | sonnet | — | no |
| designer | sonnet | — | yes |
| fixer | sonnet | — | yes |
| observer | sonnet | — | no |

`fixer` also carries `disallowedTools: Agent, Task` — it cannot spawn
subagents of its own. That's a delegation limit, not a model default, so it
doesn't get a table column; mention it if the user overrides `fixer` and
wants that capability back (removing the field, not the model change, is
what restores it).
