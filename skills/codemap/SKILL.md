---
name: codemap
description: Generate a hierarchical map of an unfamiliar repository so agents can navigate it without re-reading everything. Expensive — only for initial mapping of a codebase nobody has explored yet. If graphify is installed, use that instead.
---

> If the `graphify` skill is available in this session, use it instead — it
> builds a richer, persistent graph. This skill exists so the plugin works
> standalone.

# Codemap

Codemap builds and maintains hierarchical `codemap.md` files describing the
*why* and *how* of each directory in a repository, so agents can orient
themselves by reading a short map instead of the whole tree.

This plugin ships no code, so there is no helper script to run state
detection for you. Everything below — file discovery, change detection, and
state bookkeeping — is a workflow the calling agent runs with `Glob`, `Grep`,
`Read`, `Bash` (for `git`), and delegation to `explorer` and `fixer`, not a
script invocation.

## When to Use

- User asks to understand or map a repository.
- User wants codebase documentation for agents to navigate by.
- Starting work on an unfamiliar codebase before a larger task.

## Workflow

### Step 1: Check for Existing State

Read `.slim/codemap.json` directly — this is a single specific lookup, not a
reason to delegate. If it doesn't exist, check for legacy state at
`.slim/cartography.json` and move it to `.slim/codemap.json` if found.

- State exists → skip to Step 3 (Detect Changes).
- No state → continue to Step 2 (Initialize).

### Step 2: Initialize (only if no state exists)

1. **Decide include/exclude patterns** for core code/config files only:
   - **Include**: `src/**/*.ts`, `package.json`, and similar — whatever the
     repo's actual source layout is.
   - **Exclude (mandatory)**: tests (`**/*.test.*`, `**/*.spec.*`,
     `tests/**`, `__tests__/**`), docs (`docs/**`, `*.md` except the root
     `README.md` if useful), build/deps (`node_modules/**`, `dist/**`,
     `build/**`, `*.min.js`).
   - Respect `.gitignore`.

2. **Delegate discovery to `explorer`** (Agent tool, `subagent_type:
   "explorer"`). This is broad, uncertain-scope discovery — exactly what
   `explorer` is for. Ask it to:
   - `Glob` every file matching the include patterns, minus the excludes and
     `.gitignore` entries;
   - return the full sorted list of matched relative file paths;
   - return the set of folders that contain at least one matched file
     (every ancestor directory up to the root), with a per-folder file
     count;
   - `Read` root-level files (`package.json`, `README.md`, the plugin/entry
     file) far enough to summarize the project's overall purpose for the
     root atlas in Step 4.

3. **Record state.** Write `.slim/codemap.json` (a normal Write call — this
   is small, mechanical, and one file, so do it directly rather than
   delegating):

   ```json
   {
     "version": "1.0.0",
     "last_run": "2026-09-19T00:00:00.000Z",
     "root": ".",
     "include_patterns": ["src/**/*.ts", "package.json"],
     "exclude_patterns": ["**/*.test.ts", "dist/**", "node_modules/**"],
     "last_commit": "a1b2c3d",
     "folders": ["src", "src/agents", "src/config"]
   }
   ```

   `last_commit` is the current `git rev-parse HEAD` when the root is a git
   repository, `null` otherwise — it is what Step 3 diffs against, replacing
   the upstream script's per-file content hashes.

4. **Create empty `codemap.md` stubs** in every discovered folder that
   doesn't already have one:

   ```markdown
   # <folder>/

   <!-- Fill in this section with architectural understanding -->

   ## Responsibility

   <!-- What is this folder's job in the system? -->

   ## Design

   <!-- Key patterns, abstractions, architectural decisions -->

   ## Flow

   <!-- How does data/control flow through this module? -->

   ## Integration

   <!-- How does it connect to other parts of the system? -->
   ```

   For a handful of folders, write these directly. For many folders,
   delegate to one or more `fixer` lanes in parallel (one lane per group of
   folders) — this is bounded, mechanical work.

5. **Delegate codemap writing to `fixer`** — one lane per folder (or per
   folder group), each reading that folder's matched files and replacing the
   stub with real content per the "Codemap Content" section below.

### Step 3: Detect Changes (if state already exists)

Change detection here is git-based, not hash-based — the upstream script
hashed every matched file's bytes so it could work without git; this plugin
trades that generality for simplicity, since almost every real repo is a git
repo. See the caveat at the end of this step.

1. Read `.slim/codemap.json` for `last_commit`, `include_patterns`,
   `exclude_patterns`, and `folders`.
2. If the root is a git repository (`git rev-parse --is-inside-work-tree`):
   - `git status --porcelain` for uncommitted and untracked changes;
   - if `last_commit` is recorded and still resolves, `git diff --name-only
     <last_commit> HEAD` for changes committed since the last codemap run;
   - combine both into one list of changed paths.
3. Filter that list down to paths matching `include_patterns` and not
   `exclude_patterns`. For a large change set, hand the list and the
   patterns to `explorer` to filter with `Glob`/`Grep`; for a small one,
   filter it directly.
4. Classify each filtered path as **added** (not in the old `folders`'
   tracked files), **removed** (was tracked, no longer exists), or
   **modified** (still exists, appears in the changed-paths list). Report
   the three buckets, same as the upstream script's `changes` output.
5. Derive **affected folders**: every ancestor directory of every changed
   path, plus the root. Only these folders need their `codemap.md` updated.
6. Delegate updates to `fixer`, one lane per affected folder.
7. **Update state**: rewrite `.slim/codemap.json` with the new
   `last_commit` (current `HEAD`) and `last_run`. If the root isn't a git
   repo, just bump `last_run`.

**Caveat (real capability gap vs. upstream):** without git, there is no
cheap way to detect what changed since the last run. If the root is not a
git repository, or `last_commit` no longer resolves (e.g. history was
rewritten), say so plainly and fall back to a full Step 2 remap instead of
guessing.

### Step 4: Finalize Repository Atlas (Root Codemap)

Once affected/new directories are mapped, create or update the root
`codemap.md` — the entry point for any agent entering the repository:

1. **Map root assets**: document root-level files (`package.json`,
   `index.ts`, the plugin manifest, etc.) and the project's overall purpose.
2. **Aggregate sub-maps**: a "Repository Directory Map" section listing, for
   every folder with a `codemap.md`, its **Responsibility** summary and a
   link to the detailed map.
3. **Cross-reference**: paths to sub-maps so agents can jump directly to
   detail.

### Step 5: Register Codemap in CLAUDE.md

Claude Code auto-loads `CLAUDE.md` (project root and user home) into agent
context every session. To make agents discover the codemap automatically,
update or create the project's root `CLAUDE.md`:

1. If it already has a `## Repository Map` section, skip — already set up.
2. If it exists without that section, append the block below.
3. If it doesn't exist, create it with the block below.

```markdown
## Repository Map

A full codemap is available at `codemap.md` in the project root.

Before working on any task, read `codemap.md` to understand:
- Project architecture and entry points
- Directory responsibilities and design patterns
- Data flow and integration points between modules

For deep work on a specific folder, also read that folder's `codemap.md`.
```

Idempotent — repeated codemap runs detect the existing section and skip.

## Codemap Content

`fixer` lanes write `codemap.md` content. Use precise technical terminology:

- **Responsibility** — the directory's role using standard engineering terms
  (e.g. "Service Layer", "Data Access Object", "Middleware").
- **Design** — name specific patterns used (e.g. "Observer", "Factory",
  "Strategy"); detail abstractions and interfaces.
- **Flow** — trace how data enters and leaves the module; specific function
  call sequences and state transitions.
- **Integration** — dependencies and consumer modules; technical names for
  hooks, events, or API endpoints.

Example folder codemap:

```markdown
# src/auth/

## Responsibility
Authenticates incoming requests and issues session tokens.

## Design
Strategy pattern over pluggable auth providers:
- `providers/password.ts`, `providers/oauth.ts` implement a shared
  `AuthProvider` interface.
- `session.ts` is a thin façade over the token store.

## Flow
1. Request hits `middleware.ts` → resolves provider by request shape.
2. Provider validates credentials, returns a `Principal`.
3. `session.ts` mints a signed token, attaches it to the response.
4. Downstream handlers read `req.principal`, set by the middleware.

## Integration
- Consumed by: every route under `src/routes/` via `middleware.ts`.
- Depends on: `src/config` for token secrets, `src/db` for user lookup.
```

Example root atlas:

```markdown
# Repository Atlas: example-api

## Project Responsibility
A REST API service handling user accounts and billing.

## System Entry Points
- `src/index.ts`: process entry point, wires middleware and routes.
- `package.json`: dependency manifest and build scripts.
- `config/default.json`: runtime configuration defaults.

## Directory Map (Aggregated)
| Directory | Responsibility Summary | Detailed Map |
|-----------|------------------------|--------------|
| `src/auth/` | Authenticates requests, issues session tokens. | [View Map](src/auth/codemap.md) |
| `src/billing/` | Subscription and invoice logic, payment provider integration. | [View Map](src/billing/codemap.md) |
| `src/db/` | Data access layer and migrations. | [View Map](src/db/codemap.md) |
```
