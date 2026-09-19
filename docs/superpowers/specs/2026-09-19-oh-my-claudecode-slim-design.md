# oh-my-claudecode-slim — дизайн

Дата: 2026-09-19
Статус: утверждён, готов к написанию плана реализации

## Задача

Порт идеи [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)
на Claude Code. Существующее решение
[oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) отвергнуто как
избыточное: 19 агентов, 39 скиллов, 21 команда, ~6900 файлов, hooks.json на 220
строк, собственный MCP-сервер, HUD, team-server, npm-пакет.

## Результат исследования

oh-my-opencode-slim — 61 МБ TypeScript. Но ценность в нём не в коде, а в промптах:

| Слой | Объём | Переносимость в Claude Code |
|---|---|---|
| 7 агентов | orchestrator 365 строк, остальные 45–90 | да, как `agents/*.md` |
| 8 скиллов | codemap, deepwork, verification-planning, simplify, worktrees, clonedeps, reflect, self-config | да, как `skills/*/SKILL.md` |
| council | N моделей параллельно + синтез | нет смысла без мультипровайдера |
| TS-рантайм | background job board, task_status/revive, session reuse, мультиплексеры, companion (Rust), interview, presets, TUI, smartfetch, ast-grep | не нужен |

**Ключевой вывод:** весь TS-рантайм существует только потому, что OpenCode требует
регистрировать агентов программно. Claude Code читает агентов, скиллы и команды
как markdown-файлы. Порт = набор `.md`, ноль рантайма.

## Принятые решения

1. **Оркестратор — скилл, не хук.** Вызывается явно как `/orchestrate` и
   автоматически по `description`. Отвергнут SessionStart-хук (постоянные
   накладные расходы токенов в каждой сессии).
2. **Мультипровайдер не делаем.** Для этого есть opencode. Следствие: `council`
   выкинут — в Claude Code все councillors были бы одной моделью.
3. **6 агентов** — весь пантеон slim минус council.
4. **Модели настраиваются** через дефолты в плагине + скилл `omcs-config`.
5. **Самодостаточный набор**, но каждый пересекающийся скилл уступает дорогу
   superpowers / ponytail / graphify, если те установлены.
6. **7 скиллов.** `simplify` и `worktrees` выкинуты — покрыты нативно.

## Факты о Claude Code, на которых стоит дизайн

Проверено по докам (code.claude.com), не по памяти:

- Frontmatter агента: `model` (`haiku`/`sonnet`/`opus`/`inherit`/полный ID),
  `effort`, `maxTurns`, `tools`, `disallowedTools`.
- Plugin-агенты **игнорируют** поля `hooks`, `mcpServers`, `permissionMode`.
- Встроенный `Explore` с v2.1.198 наследует модель главной сессии, с потолком
  Opus на Claude API. Раньше всегда работал на Haiku. То есть на дорогой сессии
  поиск стал дорогим.
- Пользовательский или проектный субагент с именем `Explore` перебивает
  встроенный и сохраняет свой `model`. Про plugin-scope доки этой гарантии не
  дают — поэтому делаем свой `explorer`, а не переопределяем `Explore`.
- Про модель встроенного `Plan` отдельного правила в доках не нашёл. Не
  подтверждено, вероятно наследует.
- Приоритет scope: managed > project > user > plugin. Значит файл в
  `~/.claude/agents/fixer.md` перебивает плагинный.
- `CLAUDE_CODE_SUBAGENT_MODEL` — дефолт для всех сабагентов;
  `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` — жёстко, игнорирует `model` в
  определениях.

## Раскладка репозитория

```
oh-my-claudecode-slim/
  .claude-plugin/
    plugin.json          имя, версия, автор
    marketplace.json     установка через /plugin marketplace add
  agents/
    explorer.md
    oracle.md
    librarian.md
    designer.md
    fixer.md
    observer.md
  skills/
    orchestrate/SKILL.md
    omcs-config/SKILL.md
    deepwork/SKILL.md
    verification-planning/SKILL.md
    codemap/SKILL.md
    clonedeps/SKILL.md
    reflect/SKILL.md
  README.md
  LICENSE
```

Нет: `hooks/`, `scripts/`, `package.json`, сборки, MCP-сервера, npm-публикации,
автообновления.

`commands/` тоже нет: скиллы в Claude Code уже вызываются как `/<имя>`. Если
из-за namespace вызов окажется неудобным (`/oh-my-claudecode-slim:orchestrate`),
добавляем `commands/orchestrate.md` — одна строка, обратимо.

## Скилл `orchestrate` — ядро

Адаптация промпта оркестратора из slim (365 строк → ориентир ~120).

**Остаётся:**

- роль workflow-менеджера, а не исполнителя по умолчанию
- реестр агентов с блоками «делегируй когда / не делегируй когда»
- routing threshold: сам делаю только одно изолированное низкорисковое действие,
  когда накладные расходы на делегирование выше самой работы
- параллельный dispatch независимых линий перед зависимой работой
- непересекающиеся write-scope у параллельных писателей
- design handoff discipline: не «упрощать» и не рефакторить работу designer после
  него; копирайт правит оркестратор, не ломая визуальные решения
- правила общения: без лести, без пересказа собственных действий, честный
  pushback, краткие уведомления о делегировании

**Выкидывается — покрыто нативно:**

| В slim | В Claude Code |
|---|---|
| Background Job Board, `task_status` | уведомления харнесса + `ListAgents` |
| `task_revive`, `task_message` | `SendMessage` в живого агента |
| `wait_for_user` | `AskUserQuestion` |
| `task_cancel` | `TaskStop` |
| session reuse через `task_id` | `SendMessage` по имени агента |
| `background: true` | `run_in_background` в Agent tool |
| смена модели у сабагента | `model` в Agent tool |

**Добавляется** блок deference одним абзацем: если установлены superpowers — для
творческой работы сначала `brainstorming`, для многошаговых планов
`writing-plans`; если установлен ponytail — он главный по упрощению; если
установлен graphify — он вместо `codemap`.

## Агенты и модели

| Агент | model | effort | Права | Роль |
|---|---|---|---|---|
| `explorer` | haiku | — | read-only | Поиск по коду. Дешевле встроенного Explore. Оркестратор зовёт его вместо Explore |
| `oracle` | opus | high | read-only | Архитектура, сложный дебаг, ревью, YAGNI-надзор |
| `librarian` | sonnet | — | read-only + WebSearch/WebFetch | Внешние доки, GitHub-примеры, библиотеки |
| `designer` | sonnet | — | write | UI/UX: делает и ревьюит |
| `fixer` | sonnet | — | write, `disallowedTools: Agent` | Реализация по готовой спеке, без своего делегирования |
| `observer` | sonnet | — | read-only | Картинки, PDF, диаграммы изолированно от главного контекста |

Промпты берутся из `src/agents/*.ts` оригинала и переписываются под инструменты
Claude Code (Grep/Glob/Read вместо grep/glob/ast_grep_search).

Переопределение моделей: `/omcs-config` копирует нужного агента в
`~/.claude/agents/` с другой моделью — user scope перебивает plugin scope.

## Состав скиллов

| Скилл | Назначение | Уступает дорогу |
|---|---|---|
| `orchestrate` | ядро, авто-триггер + `/orchestrate` | — |
| `omcs-config` | модели, агенты, тюнинг самого плагина | — |
| `deepwork` | многофазная рискованная работа с гейтами ревью | superpowers:writing-plans |
| `verification-planning` | evidence path до начала нетривиальной работы | — |
| `codemap` | иерархическая карта незнакомой репы | graphify |
| `clonedeps` | исходники зависимостей локально для чтения | — |
| `reflect` | повторяющееся трение → новый скилл/агент/правило | — |

## Чего сознательно не делаем

Мультипровайдер и `council` · companion-окно · tmux/zellij/kitty панели · preset
switching · interview · smartfetch · ast-grep · собственные MCP · скиллы
`simplify` и `worktrees` (нативные `/simplify` и `EnterWorktree`/`ExitWorktree`) ·
автообновление · npm-пакет.

## Проверка

Кода нет, поэтому проверка — smoke-чеклист в README, не тесты:

1. `claude plugin validate` — манифест валиден
2. локальная установка через `/plugin marketplace add <путь или репа>`
3. `@`-typeahead показывает все 6 агентов
4. `/orchestrate` на игрушечной задаче раскидывает работу минимум в 2
   параллельных фоновых агента
5. `/omcs-config поставь fixer на haiku` создаёт `~/.claude/agents/fixer.md`
   с `model: haiku` и неизменённым остальным содержимым
6. `explorer` в `/tasks` показывает haiku, `oracle` — opus

Автоматизация через `claude plugin eval` — отдельный заход, не в этой итерации.

## Итог

~15 файлов, ноль зависимостей, ноль сборки, ноль рантайма.
