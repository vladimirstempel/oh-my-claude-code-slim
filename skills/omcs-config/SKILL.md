---
name: omcs-config
description: Configure oh-my-claudecode-slim — change which model or effort level an agent runs on, disable an agent, or override its prompt. Use when the user wants a different model for explorer, oracle, librarian, designer, fixer, or observer, or asks to tune this plugin.
---

# omcs-config

This plugin has no config file and no code path for changing an agent's model.
Three mechanisms actually work, and they trade off scope against effort. Pick
the narrowest one that answers the request.

## Which mechanism

| Want | Use | Persists? |
|---|---|---|
| A different model for one delegation | the Agent tool's `model` argument | no |
| Every subagent on one model | `CLAUDE_CODE_SUBAGENT_MODEL` | yes, per settings file |
| One agent permanently changed | fork the agent file + deny the plugin twin | yes |

### Important: plugin agents are scoped, and bare names do not shadow them

Claude Code registers this plugin's agents as `oh-my-claudecode-slim:explorer`,
`oh-my-claudecode-slim:oracle`, and so on. That prefix is part of the
identifier, not decoration.

A file at `~/.claude/agents/explorer.md` therefore does **not** shadow the
plugin's `explorer`. The two names never collide, so the general scope
precedence (managed settings > `--agents` CLI flag > project `.claude/agents/`
> user `~/.claude/agents/` > plugin) never gets a chance to apply. You end up
with two separate agents — `explorer` and `oh-my-claudecode-slim:explorer` —
both live at once. And because `orchestrate` dispatches the scoped identifier,
the plugin's copy is the one that gets the work. Copying a file is not enough
on its own; see "Permanently, one agent" below.

## 1. One dispatch only: the `model` argument

The Agent tool accepts a `model` argument on each call: `haiku`, `sonnet`,
`opus`, or `fable`. It overrides the agent's own `model:` frontmatter for that
one delegation and touches no file.

This is the cheapest answer and almost always the right one for "run the oracle
on sonnet this once" or "do this search on opus". Prefer it. Nothing to revert.

## 2. Every subagent at once: environment variables

- `CLAUDE_CODE_SUBAGENT_MODEL` (in the `env` block of a settings file) only
  affects subagents that don't set their own `model:` frontmatter. It does
  **not** move this plugin's six agents by itself — every one of them sets
  `model:` explicitly, and Claude Code's resolution order puts a subagent's
  own frontmatter ahead of this variable (per-invocation parameter >
  frontmatter `model:` > `CLAUDE_CODE_SUBAGENT_MODEL` > main conversation's
  model). That order applies from Claude Code v2.1.251 on; before it, the
  variable came first and overrode both.
- To force every subagent — including this plugin's — onto one model
  regardless of its `model:` field, also set `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`
  to `1`. Requires Claude Code v2.1.257+. Set without
  `CLAUDE_CODE_SUBAGENT_MODEL`, it falls back to the main conversation's
  model instead of forcing anything useful.
- Verify what a running subagent uses: run `/tasks`; the row names the
  model, and the effort level when one is set.

Revert by removing the variables from the settings file's `env` block.

## 3. Permanently, one agent: fork it, then deny the twin

Use this only when the change must stick across sessions for a single agent.
It is two steps, and skipping the second leaves the user with two agents and
the plugin's copy still winning every `orchestrate` dispatch.

1. Read the plugin's version at `${CLAUDE_PLUGIN_ROOT}/agents/<name>.md`.
2. Write the whole file to `~/.claude/agents/<name>.md`, changing only the
   `model:` (or `effort:`) line. Copy the prompt body verbatim — this
   procedure is a model/effort swap, not a rewrite, so don't paraphrase it.
3. Disable the plugin's twin with a permission rule, so exactly one of the two
   remains. Add the **scoped** identifier to the `permissions.deny` array in a
   settings file:

   ```json
   { "permissions": { "deny": ["Agent(oh-my-claudecode-slim:<name>)"] } }
   ```

   The scoped form is what works — `"Agent(<name>)"` would deny the user's own
   fork instead. Write it plain, without regex anchors: `^` and `$` inside
   `Agent(...)` were verified not to match here, and the agent stays enabled.
4. Tell the user two things: the override is now user-scoped, so plugin updates
   will no longer reach that agent; and `orchestrate` will now dispatch the
   bare `<name>`, because the scoped one is gone.

Which settings file: project `.claude/settings.json`, project-local
`.claude/settings.local.json`, or user `~/.claude/settings.json`. Ask the user
which scope they want; user scope is usually right since it won't affect
teammates on a shared project.

Revert by deleting `~/.claude/agents/<name>.md` and dropping the
`permissions.deny` rule.

## Disable an agent outright

Same permission rule, without the fork: add `"Agent(oh-my-claudecode-slim:<name>)"`
(e.g. `"Agent(oh-my-claudecode-slim:fixer)"`) to `permissions.deny` in one of
the settings files above. The agent disappears from the available list
entirely. One-off session alternative:
`claude --disallowedTools "Agent(oh-my-claudecode-slim:<name>)"`.

## Override the prompt

Same fork mechanism as section 3, different intent: write
`~/.claude/agents/<name>.md` with the user's own prompt body in place of the
plugin's, and deny the plugin twin so the fork is the one that runs. Start from
the plugin's frontmatter and carry every field forward unless the user is
deliberately changing it — don't reduce the list to whatever the new prompt
happens to need. This matters most for `tools` and `disallowedTools`: those are
permission boundaries, not preferences, and dropping them silently restores
capability the plugin deliberately removed. `fixer` ships
`disallowedTools: Agent, Task` — the line that stops it recursively delegating;
lose it here and a rewritten `fixer` can spawn subagents with no error to warn
anyone. Unlike the model swap above, the prompt body itself is a deliberate
rewrite — ask the user what it should say rather than inventing or paraphrasing
one yourself.

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
