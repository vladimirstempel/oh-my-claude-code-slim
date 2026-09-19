# oh-my-claudecode-slim Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Плагин для Claude Code с 6 специализированными агентами и 7 скиллами, переносящий идею oh-my-opencode-slim без единой строки исполняемого кода.

**Architecture:** Плагин состоит только из markdown-файлов и двух JSON-манифестов. Агенты — `agents/*.md` с frontmatter, скиллы — `skills/*/SKILL.md`. Ядро — скилл `orchestrate`, который учит главный тред раскидывать работу по агентам. Рантайма, сборки и зависимостей нет: то, что в оригинале делал TypeScript (фоновые задачи, реестр сессий, отмена, переиспользование), в Claude Code уже есть нативно.

**Tech Stack:** Markdown, JSON. Никаких зависимостей. Проверка — `claude plugin validate` и ручной smoke-чеклист.

**Spec:** `docs/superpowers/specs/2026-09-19-oh-my-claudecode-slim-design.md`

## Замечание про TDD

В этом проекте нет исполняемого кода, поэтому классический цикл
«красный тест → зелёный тест» неприменим. Вместо него каждая задача
заканчивается **проверяемой командой**: `claude plugin validate`, `grep` по
обязательным полям frontmatter или наблюдаемое поведение в живой сессии
Claude Code. Шаги «Проверка» обязательны, их нельзя пропускать — это
единственный контроль качества в репозитории.

## Global Constraints

- Ноль исполняемого кода: никаких `scripts/`, `package.json`, сборки, npm-публикации, MCP-серверов.
- Ноль зависимостей.
- Имя плагина: `oh-my-claudecode-slim`. Владелец/автор — `stempelv@gmail.com`.
- Версия на первый релиз: `0.1.0`.
- Лицензия: MIT.
- Plugin-агенты **не поддерживают** поля `hooks`, `mcpServers`, `permissionMode` — эти поля запрещено добавлять в `agents/*.md`.
- Каждый скилл, пересекающийся с superpowers / ponytail / graphify, обязан содержать явную строку уступки в теле SKILL.md.
- Язык всех промптов и документации — английский (плагин переносимый). Исключение: файлы в `docs/superpowers/` остаются на русском.
- Апстрим-референс: `https://github.com/alvinunreal/oh-my-opencode-slim`, коммит `201562e02b9b9554e28c8967311b53a44513269f`.

---

### Task 1: Манифест плагина и скелет репозитория

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`
- Create: `.gitignore`
- Create: `LICENSE`
- Test: `claude plugin validate`

**Interfaces:**
- Consumes: ничего
- Produces: имя плагина `oh-my-claudecode-slim`, на которое ссылаются все последующие задачи; каталог `.reference/` с исходниками апстрима для портирования промптов.

- [ ] **Шаг 1: Склонировать апстрим-референс**

Промпты последующих задач портируются из оригинала. Клон нужен локально и не должен попасть в git.

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
git clone https://github.com/alvinunreal/oh-my-opencode-slim.git .reference
git -C .reference checkout 201562e02b9b9554e28c8967311b53a44513269f
```

- [ ] **Шаг 2: Создать `.gitignore`**

```gitignore
.reference/
.DS_Store
```

- [ ] **Шаг 3: Создать `.claude-plugin/plugin.json`**

```json
{
  "name": "oh-my-claudecode-slim",
  "version": "0.1.0",
  "description": "Slim multi-agent orchestration for Claude Code: six specialist agents and seven skills, zero runtime.",
  "author": {
    "name": "stempelv",
    "email": "stempelv@gmail.com"
  },
  "license": "MIT",
  "keywords": ["claude-code", "plugin", "agents", "orchestration", "slim"]
}
```

- [ ] **Шаг 4: Создать `.claude-plugin/marketplace.json`**

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "oh-my-claudecode-slim",
  "owner": {
    "name": "stempelv",
    "email": "stempelv@gmail.com"
  },
  "plugins": [
    {
      "name": "oh-my-claudecode-slim",
      "description": "Slim multi-agent orchestration for Claude Code: six specialist agents and seven skills, zero runtime.",
      "version": "0.1.0",
      "source": "./",
      "category": "productivity",
      "homepage": "https://github.com/stempelv/oh-my-claudecode-slim",
      "tags": ["agents", "orchestration", "delegation", "slim"]
    }
  ]
}
```

- [ ] **Шаг 5: Создать `LICENSE`**

Стандартный текст MIT, `Copyright (c) 2026 stempelv`.

- [ ] **Шаг 6: Проверка**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
git status --short          # .reference/ не должен появиться в выводе
```

Ожидается: валидация проходит без ошибок; `.reference/` отсутствует в `git status`.

- [ ] **Шаг 7: Коммит**

```bash
git add .claude-plugin .gitignore LICENSE
git commit -m "feat: скелет плагина и манифесты"
```

---

### Task 2: Read-only агенты (explorer, oracle, librarian, observer)

**Files:**
- Create: `agents/explorer.md`
- Create: `agents/oracle.md`
- Create: `agents/librarian.md`
- Create: `agents/observer.md`
- Test: `claude plugin validate` + grep по frontmatter

**Interfaces:**
- Consumes: манифест из Task 1
- Produces: имена агентов `explorer`, `oracle`, `librarian`, `observer` — на них ссылается реестр в скилле `orchestrate` (Task 4).

**Источники промптов** (в `.reference/`):
- `src/agents/explorer.ts` → константа `EXPLORER_PROMPT`
- `src/agents/oracle.ts` → `ORACLE_PROMPT`
- `src/agents/librarian.ts` → `LIBRARIAN_PROMPT`
- `src/agents/observer.ts` → `OBSERVER_PROMPT`
- `src/config/constants.ts` → `READONLY_FILE_OPERATIONS_RULES` (подставляется в каждый из них на месте `${READONLY_FILE_OPERATIONS_RULES}`)

**Обязательные адаптации под Claude Code:**

| В оригинале | В порте |
|---|---|
| `grep` | `Grep` |
| `glob` | `Glob` |
| `read` | `Read` |
| `ast_grep_search` | убрать; заменить на `Grep` с многострочным режимом |
| `context7`, `gh_grep` (в librarian) | `WebSearch`, `WebFetch` и MCP context7, «если доступен» |
| блок `**Output Format**` с `<results>`/`<files>` у explorer | оставить — он полезен и в Claude Code |
| «Save context tokens — the Orchestrator never processes the raw file» | оставить |

- [ ] **Шаг 1: Создать `agents/explorer.md`**

Frontmatter строго такой:

```markdown
---
name: explorer
description: Fast codebase search and pattern matching. Use for finding files, locating code patterns, and answering "where is X?" questions. Prefer this over the built-in Explore agent — it runs on Haiku and stays cheap.
model: haiku
disallowedTools: Write, Edit, NotebookEdit
---
```

Тело — `EXPLORER_PROMPT` с подставленным `READONLY_FILE_OPERATIONS_RULES` и применёнными адаптациями из таблицы выше.

- [ ] **Шаг 2: Создать `agents/oracle.md`**

```markdown
---
name: oracle
description: Strategic technical advisor. Use for architecture decisions, complex debugging after two failed fix attempts, code review, simplification, and engineering guidance. Read-only — it advises, it does not implement.
model: opus
effort: high
disallowedTools: Write, Edit, NotebookEdit
---
```

Тело — `ORACLE_PROMPT` с подставленным `READONLY_FILE_OPERATIONS_RULES`.

- [ ] **Шаг 3: Создать `agents/librarian.md`**

```markdown
---
name: librarian
description: External documentation and library research. Use for official docs lookup, GitHub examples, version-specific API behavior, and understanding library internals. Not for questions about this codebase — use explorer for that.
model: sonnet
disallowedTools: Write, Edit, NotebookEdit
---
```

Тело — `LIBRARIAN_PROMPT`. Секцию `**Tools to Use**` переписать:

```markdown
**Tools to Use**:
- WebSearch / WebFetch: official documentation and release notes
- context7 MCP, if the session has it: version-accurate library docs
- Grep over a local dependency checkout, if the clonedeps skill has made one
```

- [ ] **Шаг 4: Создать `agents/observer.md`**

```markdown
---
name: observer
description: Visual analysis. Use for interpreting images, screenshots, PDFs, and diagrams — returns structured observations without loading raw files into the caller's context.
model: sonnet
disallowedTools: Write, Edit, NotebookEdit
---
```

Тело — `OBSERVER_PROMPT` с подставленным `READONLY_FILE_OPERATIONS_RULES`.

- [ ] **Шаг 5: Проверка**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
# у каждого агента есть name, description, model:
for f in agents/*.md; do
  echo "--- $f"
  awk '/^---$/{n++; next} n==1' "$f" | grep -E '^(name|description|model):' | cut -c1-60
done
# запрещённых полей быть не должно:
grep -l -E '^(hooks|mcpServers|permissionMode):' agents/*.md && echo "ОШИБКА: запрещённое поле" || echo "ok: запрещённых полей нет"
```

Ожидается: валидация проходит; у всех четырёх файлов есть `name`, `description`, `model`; последняя команда печатает `ok`.

- [ ] **Шаг 6: Коммит**

```bash
git add agents
git commit -m "feat: read-only агенты explorer, oracle, librarian, observer"
```

---

### Task 3: Пишущие агенты (designer, fixer)

**Files:**
- Create: `agents/designer.md`
- Create: `agents/fixer.md`
- Test: `claude plugin validate` + grep

**Interfaces:**
- Consumes: манифест из Task 1
- Produces: имена агентов `designer`, `fixer` — на них ссылается реестр в скилле `orchestrate` (Task 4).

**Источники промптов:** `.reference/src/agents/designer.ts` → `DESIGNER_PROMPT`, `.reference/src/agents/fixer.ts` → `FIXER_PROMPT`, `.reference/src/config/constants.ts` → `WRITABLE_FILE_OPERATIONS_RULES`.

Отдельная задача от Task 2, потому что здесь другая модель прав: эти агенты
меняют файлы, и ревьюер должен отдельно проверить границы их полномочий.

- [ ] **Шаг 1: Создать `agents/designer.md`**

```markdown
---
name: designer
description: UI/UX design, implementation, and review. Use for styling, responsive layout, visual hierarchy, motion, component feel, and design polish. Owns visual quality — do not hand UI work to fixer.
model: sonnet
---
```

Тело — `DESIGNER_PROMPT` с подставленным `WRITABLE_FILE_OPERATIONS_RULES`. Адаптации:

- «Run only validation assigned by the Orchestrator» → «Run only the validation the caller assigned».
- Упоминания `edit/write/apply_patch` в правилах файловых операций → `Edit` / `Write`.

- [ ] **Шаг 2: Создать `agents/fixer.md`**

```markdown
---
name: fixer
description: Bounded implementation specialist. Give it a complete task spec and full context; it executes code changes and reports back. Not for research, architecture decisions, or UI/design work.
model: sonnet
disallowedTools: Agent
---
```

`disallowedTools: Agent` обязателен: в оригинале запрет на порождение
субагентов был только словами в промпте, здесь он становится реальным
ограничением инструментов.

Тело — `FIXER_PROMPT` с подставленным `WRITABLE_FILE_OPERATIONS_RULES`. Адаптации:

- «NO external research (no context7, gh_grep)» → «NO external research: no WebSearch, no WebFetch. If the task needs external docs, say so and stop.»
- «NO spawning subagents» оставить в тексте — дублирование с `disallowedTools` полезно, агент узнает причину отказа.

- [ ] **Шаг 3: Проверка**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
grep -c 'disallowedTools: Agent' agents/fixer.md    # ожидается 1
grep -E '^(hooks|mcpServers|permissionMode):' agents/*.md && echo "ОШИБКА" || echo "ok"
ls agents/ | wc -l                                   # ожидается 6
```

- [ ] **Шаг 4: Коммит**

```bash
git add agents
git commit -m "feat: пишущие агенты designer и fixer"
```

---

### Task 4: Скилл `orchestrate` — ядро

**Files:**
- Create: `skills/orchestrate/SKILL.md`
- Test: живая сессия Claude Code

**Interfaces:**
- Consumes: имена всех шести агентов из Task 2 и Task 3
- Produces: точку входа `/orchestrate`; поведение делегирования, на которое опираются `deepwork` (Task 6) и `reflect` (Task 7).

**Источник:** `.reference/src/agents/orchestrator.ts` — константа `AGENT_DESCRIPTIONS`
и функция `buildOrchestratorPrompt`. Ориентир по объёму: ~120 строк против 365 в оригинале.

- [ ] **Шаг 1: Написать frontmatter**

```markdown
---
name: orchestrate
description: Plan, delegate, and reconcile multi-part coding work across specialist agents instead of doing everything in the main thread. Use when a task has separable lanes — discovery, external research, implementation, UI, review — or when the user asks to orchestrate, delegate, or parallelize. Also triggered by /orchestrate.
---
```

- [ ] **Шаг 2: Написать секцию Role**

Портировать блок `<Role>` из `buildOrchestratorPrompt` дословно, заменив
«You are a workflow manager» на формулировку, которая работает как скилл, а не
как системный промпт: «While this skill is active, act as a workflow manager…».

- [ ] **Шаг 3: Написать реестр агентов**

Портировать `AGENT_DESCRIPTIONS` для шести агентов (`explorer`, `librarian`,
`oracle`, `designer`, `fixer`, `observer`), сохранив структуру каждого блока:
Lane / Role / Capabilities / **Delegate when** / **Don't delegate when** /
**Rule of thumb**. Блок `council` не переносить.

Обязательные правки при портировании:

- `@explorer` → «`explorer`, via the Agent tool with `subagent_type: "explorer"`».
- В блоке `explorer` добавить строку: «Prefer `explorer` over the built-in `Explore` agent: `Explore` inherits the main conversation's model (capped at Opus), `explorer` is pinned to Haiku.»
- Строки про `Permissions: read_files` → «Read-only: Write and Edit are denied».
- Из блока `observer` сохранить требование передавать полный путь к файлу.
- Из блока `designer` сохранить предупреждение про слабый копирайт и правило «не проси designer рассказать как надо, проси сделать».

- [ ] **Шаг 4: Написать секцию Workflow**

Портировать пункты 1–5 (`Understand`, `Path Selection`, `Delegation Check`,
`Plan and Parallelize`, `Verify`), включая:

- routing threshold целиком
- `Dispatch efficiency` (ссылаться на пути, а не вставлять файлы; кратко сообщать пользователю о делегировании)
- `Todo Continuity` целиком
- правило непересекающихся write-scope у параллельных писателей
- `Design Handoff Discipline` целиком

Заменить механику фоновых задач на нативную:

```markdown
### Background dispatch

- Launch independent lanes with the Agent tool and `run_in_background: true`, then end the turn with a one-line status. The harness notifies you when each finishes — do not poll.
- To check what is still running, call `ListAgents`.
- To add context to a running agent, use `SendMessage` with its name. A delivered message is not a guarantee the agent read or acted on it.
- To resume a finished agent with its context intact, use `SendMessage`. A fresh `Agent` call starts cold.
- To stop a lane that has become obsolete, use `TaskStop`. Inspect its partial writes before launching a replacement.
- When work must pause for something only the user can do, use `AskUserQuestion`. Never use it to wait for background agents.
- Never reissue an unchanged task to the same specialist after a rejection — adjust scope or context first.
```

- [ ] **Шаг 5: Написать секцию Deference**

```markdown
## Defer to installed skills

- `superpowers:brainstorming` installed and the task is creative or new-build work → run it before any delegation.
- `superpowers:writing-plans` installed and the work is multi-step → use it instead of the bundled `deepwork` skill.
- `ponytail` installed → it owns simplification and scope-cutting; do not duplicate its judgement.
- `graphify` installed → use it instead of the bundled `codemap` skill.
```

- [ ] **Шаг 6: Написать секцию Communication**

Портировать блок `<Communication>` целиком: Clarity Over Assumptions,
Concise Execution, No Flattery, Honest Pushback и пример Bad/Good. Убрать
упоминания `question` tool → `AskUserQuestion`, убрать `wait_for_user`.

- [ ] **Шаг 7: Проверка в живой сессии**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
wc -l skills/orchestrate/SKILL.md     # ориентир: 100–160 строк
```

Затем в новой сессии Claude Code с установленным плагином:

1. Ввести `/orchestrate` — скилл должен загрузиться.
2. Дать задачу с двумя независимыми линиями, например: «найди, где в этом
   репозитории задаются модели агентов, и параллельно выясни по докам, можно ли
   переопределить встроенный Plan-агент».
3. Ожидается: два вызова Agent tool с `run_in_background: true` в одном
   ответе — `explorer` и `librarian`; затем завершение хода с кратким статусом,
   без опроса статуса в цикле.

Записать фактический результат. Если делегирования не произошло — усилить
формулировки routing threshold и повторить.

- [ ] **Шаг 8: Коммит**

```bash
git add skills/orchestrate
git commit -m "feat: скилл orchestrate — ядро делегирования"
```

---

### Task 5: Скилл `omcs-config`

**Files:**
- Create: `skills/omcs-config/SKILL.md`
- Test: живая сессия Claude Code

**Interfaces:**
- Consumes: файлы агентов из Task 2 и Task 3, переменную `${CLAUDE_PLUGIN_ROOT}`
- Produces: способ переопределять модели агентов; на него ссылается README (Task 8).

Своего аналога в оригинале нет — там это делал TypeScript-конфиг. Здесь скилл
работает через приоритет scope: файл в `~/.claude/agents/` перебивает плагинный.

- [ ] **Шаг 1: Написать frontmatter**

```markdown
---
name: omcs-config
description: Configure oh-my-claudecode-slim — change which model or effort level an agent runs on, disable an agent, or override its prompt. Use when the user wants a different model for explorer, oracle, librarian, designer, fixer, or observer, or asks to tune this plugin.
---
```

- [ ] **Шаг 2: Описать механику переопределения**

Тело скилла должно содержать ровно этот алгоритм:

```markdown
## How an override works

Claude Code resolves subagents by scope: managed > project > user > plugin. A
file at `~/.claude/agents/<name>.md` shadows the plugin's version of the same
agent completely — the plugin file is not merged, it is replaced.

To change one agent's model:

1. Read the plugin's version at `${CLAUDE_PLUGIN_ROOT}/agents/<name>.md`.
2. Write the whole file to `~/.claude/agents/<name>.md`, changing only the
   `model:` (or `effort:`) line. Never paraphrase the prompt body.
3. Tell the user the override is now user-scoped: plugin updates will no longer
   reach that agent until they delete the file.

To revert: delete `~/.claude/agents/<name>.md`.
```

- [ ] **Шаг 3: Описать альтернативы через настройки**

```markdown
## Session-wide alternatives

- One model for every subagent: set `CLAUDE_CODE_SUBAGENT_MODEL` in the `env`
  block of a settings file.
- Force it, ignoring every agent's own `model:` field: also set
  `CLAUDE_CODE_SUBAGENT_MODEL_FORCE` to `1`. Requires Claude Code v2.1.257+.
- Verify what a running subagent uses: run `/tasks` while it is running; the
  row shows its model.
```

- [ ] **Шаг 4: Добавить таблицу дефолтов**

```markdown
## Defaults shipped by this plugin

| Agent | model | effort | Writes? |
|---|---|---|---|
| explorer | haiku | — | no |
| oracle | opus | high | no |
| librarian | sonnet | — | no |
| designer | sonnet | — | yes |
| fixer | sonnet | — | yes |
| observer | sonnet | — | no |
```

- [ ] **Шаг 5: Проверка в живой сессии**

В сессии с установленным плагином: «/omcs-config поставь fixer на haiku».

```bash
cat ~/.claude/agents/fixer.md | head -8
diff <(sed -n '/^---$/,/^---$/!p' ~/.claude/agents/fixer.md) \
     <(sed -n '/^---$/,/^---$/!p' /mnt/d/Projects/oh-my-claudecode-slim/agents/fixer.md)
```

Ожидается: файл создан, `model: haiku`, `diff` тела пустой — промпт не переписан.
Затем удалить файл: `rm ~/.claude/agents/fixer.md`.

- [ ] **Шаг 6: Коммит**

```bash
git add skills/omcs-config
git commit -m "feat: скилл omcs-config для настройки моделей агентов"
```

---

### Task 6: Скиллы `deepwork` и `verification-planning`

**Files:**
- Create: `skills/deepwork/SKILL.md`
- Create: `skills/verification-planning/SKILL.md`
- Test: `claude plugin validate` + grep

**Interfaces:**
- Consumes: поведение делегирования из Task 4
- Produces: два скилла, упомянутых в README (Task 8)

**Источники:** `.reference/src/skills/deepwork/SKILL.md` (158 строк),
`.reference/src/skills/verification-planning/SKILL.md` (111 строк).

- [ ] **Шаг 1: Портировать `deepwork`**

Frontmatter:

```markdown
---
name: deepwork
description: Structured workflow for large, high-risk, multi-phase coding efforts with dependencies and review gates. Do not activate for routine multi-file changes. If superpowers:writing-plans is installed, use that instead.
---
```

Тело — оригинальный SKILL.md с заменами: `@fixer`/`@oracle`/`@explorer` →
имена агентов через Agent tool; `task_status`/`task_result` → `ListAgents` и
уведомления харнесса; `wait_for_user` → `AskUserQuestion`.

Первой строкой тела добавить уступку:

```markdown
> If `superpowers:writing-plans` is available in this session, stop and use it
> instead. This skill exists so the plugin works without superpowers installed.
```

- [ ] **Шаг 2: Портировать `verification-planning`**

Frontmatter:

```markdown
---
name: verification-planning
description: Build a project-specific evidence path before implementing a feature, bug fix, refactor, or any non-trivial behavior change — decide up front what would prove or refute the change working. Use before the work, not after.
---
```

Тело — оригинальный SKILL.md практически дословно: он про рассуждение, а не про
инструменты, так что адаптаций почти не требуется. Проверить и заменить только
упоминания опенкодовых инструментов, если найдутся.

Уступки здесь **не нужно** — `superpowers:verification-before-completion`
проверяет результат постфактум, а этот скилл строит план доказательства до
начала. Это разные вещи.

- [ ] **Шаг 3: Проверка**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
grep -ri 'opencode\|@fixer\|@oracle\|@explorer\|task_status\|task_result\|wait_for_user' skills/ \
  && echo "ОШИБКА: остались следы оригинала" || echo "ok"
grep -c 'superpowers:writing-plans' skills/deepwork/SKILL.md    # ожидается >= 2
```

- [ ] **Шаг 4: Коммит**

```bash
git add skills/deepwork skills/verification-planning
git commit -m "feat: скиллы deepwork и verification-planning"
```

---

### Task 7: Скиллы `codemap`, `clonedeps`, `reflect`

**Files:**
- Create: `skills/codemap/SKILL.md`
- Create: `skills/clonedeps/SKILL.md`
- Create: `skills/reflect/SKILL.md`
- Test: `claude plugin validate` + grep + один живой прогон `clonedeps`

**Interfaces:**
- Consumes: агента `librarian` из Task 2 (читает клонированные исходники), поведение делегирования из Task 4
- Produces: три скилла, упомянутых в README (Task 8)

**Источники:** `.reference/src/skills/codemap/SKILL.md` (163 строки),
`.reference/src/skills/clonedeps/SKILL.md` (240 строк),
`.reference/src/skills/reflect/SKILL.md` (326 строк).

- [ ] **Шаг 1: Портировать `codemap`**

```markdown
---
name: codemap
description: Generate a hierarchical map of an unfamiliar repository so agents can navigate it without re-reading everything. Expensive — only for initial mapping of a codebase nobody has explored yet. If graphify is installed, use that instead.
---
```

Оригинал ссылается на скрипт `scripts/codemap.mjs`. **Скрипт не переносим** —
глобальное ограничение запрещает исполняемый код. Переписать соответствующий
раздел как обход через `Glob` + `Grep` + `Read` силами агента `explorer`.

Первой строкой тела:

```markdown
> If the `graphify` skill is available in this session, use it instead — it
> builds a richer, persistent graph. This skill exists so the plugin works
> standalone.
```

- [ ] **Шаг 2: Портировать `clonedeps`**

```markdown
---
name: clonedeps
description: Clone the source of a few important project dependencies into an ignored local directory so agents can read library internals — debugging SDK behavior, tracing framework code, checking what an API actually does. Not for ordinary docs questions; use librarian for those.
---
```

Оригинал явно требует не использовать вспомогательный скрипт и выполнять всё
как workflow — это уже совпадает с нашим ограничением, переносить как есть.
Заменить упоминания `@librarian` на агента `librarian` через Agent tool.

- [ ] **Шаг 3: Портировать `reflect`**

```markdown
---
name: reflect
description: Review recent work, spot repeated workflow friction, and propose the smallest reusable fix — a skill, an agent, a command, a config change, a prompt rule, or no change at all. Use when the user asks to learn from past sessions or improve a recurring workflow.
---
```

Оригинал — «orchestrator-only workflow». В Claude Code это просто скилл главного
треда. Заменить упоминания опенкодовых путей конфигурации
(`~/.config/opencode/...`) на `~/.claude/` и `.claude/`, а предложения «создать
агента» — на формат `agents/*.md` из Task 2.

- [ ] **Шаг 4: Проверка**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
claude plugin validate .
ls skills/ | wc -l                                  # ожидается 7
grep -ri 'opencode\|\.mjs\|scripts/' skills/ && echo "ОШИБКА" || echo "ok"
for d in skills/*/; do
  awk '/^---$/{n++; next} n==1' "$d/SKILL.md" | grep -qE '^name:' || echo "ОШИБКА: нет name в $d"
  awk '/^---$/{n++; next} n==1' "$d/SKILL.md" | grep -qE '^description:' || echo "ОШИБКА: нет description в $d"
done
echo "frontmatter проверен"
```

Затем живой прогон в сессии: «/clonedeps склонируй исходники одной зависимости
этого проекта» на любом чужом репозитории с зависимостями. Ожидается: скилл
спрашивает какие именно, клонирует в игнорируемый каталог, не трогает `git`
основного проекта.

- [ ] **Шаг 5: Коммит**

```bash
git add skills/codemap skills/clonedeps skills/reflect
git commit -m "feat: скиллы codemap, clonedeps, reflect"
```

---

### Task 8: README и финальная приёмка

**Files:**
- Create: `README.md`
- Test: полный smoke-чеклист из спеки

**Interfaces:**
- Consumes: всё созданное в Task 1–7
- Produces: документацию установки и раздел Verify

- [ ] **Шаг 1: Написать README**

Обязательные разделы:

1. **What this is** — один абзац: шесть агентов и семь скиллов, ноль рантайма; чем отличается от oh-my-claudecode (не 39 скиллов и 19 агентов) и от oh-my-opencode-slim (нет мультипровайдера — для этого есть opencode).
2. **Install**

```bash
/plugin marketplace add https://github.com/stempelv/oh-my-claudecode-slim
/plugin install oh-my-claudecode-slim
```

3. **Agents** — таблица из спеки: агент, модель, effort, права, роль.
4. **Skills** — таблица из спеки: скилл, назначение, кому уступает дорогу.
5. **Changing models** — короткая отсылка к `/omcs-config` и к `CLAUDE_CODE_SUBAGENT_MODEL`.
6. **Verify** — чеклист из шага 2 ниже.
7. **What this deliberately does not do** — список из раздела «Чего сознательно не делаем» в спеке.
8. **Credits** — oh-my-opencode-slim (alvinunreal), MIT.

- [ ] **Шаг 2: Записать в README раздел Verify**

```markdown
## Verify

1. `claude plugin validate .` passes.
2. After install, `@` typeahead lists all six agents.
3. `/orchestrate` on a two-lane task dispatches at least two background agents in one turn.
4. `/tasks` shows `explorer` on Haiku and `oracle` on Opus while they run.
5. `/omcs-config put fixer on haiku` writes `~/.claude/agents/fixer.md` with `model: haiku` and an unchanged prompt body.
6. `/clonedeps` clones into an ignored directory and leaves the project's own git state untouched.
```

- [ ] **Шаг 3: Прогнать весь чеклист**

Выполнить пункты 1–6 по-настоящему и записать фактический результат каждого.
Не отмечать шаг выполненным по предположению — только по наблюдённому выводу.

- [ ] **Шаг 4: Финальная проверка репозитория**

```bash
cd /mnt/d/Projects/oh-my-claudecode-slim
find . -path ./.git -prune -o -path ./.reference -prune -o -type f -print | sort
# ожидается ~17 файлов, все .md или .json, ни одного .ts/.js/.mjs/.sh
find . -path ./.git -prune -o -path ./.reference -prune -o \
  \( -name '*.ts' -o -name '*.js' -o -name '*.mjs' -o -name '*.sh' -o -name 'package.json' \) -print \
  | grep . && echo "ОШИБКА: появился код" || echo "ok: кода нет"
git status --short   # чисто, .reference не виден
```

- [ ] **Шаг 5: Коммит и тег**

```bash
git add README.md
git commit -m "docs: README с установкой, таблицами агентов и чеклистом приёмки"
git tag v0.1.0
```

---

## Self-review плана

**Покрытие спеки:**

| Раздел спеки | Задача |
|---|---|
| Раскладка репозитория | Task 1 |
| 6 агентов и их модели | Task 2, Task 3 |
| Скилл orchestrate, что остаётся и что выкидывается | Task 4 |
| Настройка моделей через omcs-config | Task 5 |
| Состав из 7 скиллов | Task 4, 5, 6, 7 |
| Уступки superpowers / ponytail / graphify | Task 4 (шаг 5), Task 6 (шаг 1), Task 7 (шаг 1) |
| Чего сознательно не делаем | Task 8 (шаг 1, раздел 7); проверяется в Task 8 шаг 4 |
| Проверка: smoke-чеклист | Task 8 |

Пробелов нет.

**Заглушек нет:** каждый шаг содержит либо готовый к вставке текст, либо точную
ссылку на исходный файл в `.reference/` с перечнем требуемых правок.

**Согласованность имён:** `explorer`, `oracle`, `librarian`, `designer`,
`fixer`, `observer` — одинаково во всех задачах. Имя плагина
`oh-my-claudecode-slim` одинаково в `plugin.json`, `marketplace.json`, README и
командах установки. Скилл настройки везде `omcs-config`.
