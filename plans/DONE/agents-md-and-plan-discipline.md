# План: дисциплина AGENTS.md, файлы планов, подготовка к сквозной нумерации PM-MCP

## Контекст

Аудит правил агентов выявил три проблемы:

1. **Файлы планов** — ни один `AGENTS.md` не описывает, где их хранить. Claude Code в Plan Mode пишет в `~/.claude/plans/<random>.md`, Codex — в свою папку. Из-за этого агенты не видят чужие черновики, пользователь копи-пастит весь план между ними, каждая итерация съедает токены на повторное чтение 80% неизменённого текста.
2. **Дрейф правил** — большие блоки правил (`E`, `J`, `K`, и др.) уже скопированы в 10+ `AGENTS.md`, что неминуемо приведёт к расхождениям. Раздел `Q` (миграция/удаление легаси) есть в большинстве файлов, но не в `Design-system`, `ANKI` и минимальных Design-system внутри Best_photo_ai/Verua_automation.
3. **Нумерация задач PM-MCP** — каждая подсистема имеет свой счётчик, что даёт коллизии (`#057 ai-memory` ≠ `#057 pm-mcp-server`). Пользователь хочет единый глобальный сквозной счётчик.

Дополнительно: в `C:\Users\Zaxva\.claude\plans\` накопилось 26 устаревших файлов плана.

**Этот план покрывает только документационную часть (шаги 1–4 ниже).** Сквозная нумерация PM-MCP (Шаг 5) — самая рискованная часть с большим радиусом поражения — выносится **в отдельный ADR + план**: сначала инвентаризация схемы, dry-run mapping, бэкап, потом миграция. Подмешивать её к чистке планов и обновлению инструкций нельзя.

---

## Состояние на момент планирования (snapshot)

**PM-MCP база:**
- Всего задач: **339** (по проектам: ai-memory 105 / assistant-ui 125 / pm-mcp-server 102 / gateway 7)
- Не закрыто: **9 задач**, все актуальны (никаких неактуальных закрывать не нужно). Перед любым массовым `update_task`/`close_task` подтвердить допустимые значения `status`/`priority` через PM-MCP schema (правило раздела D глобального CLAUDE.md).

**AGENTS.md с разделом Q:** AI-Assistant (корневой), Best_photo_ai, Verua_automation, GmbH, Hauswirtschaftiche_Pflegeleistungen, Stalking_offline, _project_template.

**Без Q:** Design-system, ANKI, Best_photo_ai/Design-system, Verua_automation/Design-system.

**Подсистемы AI-Assistant** (`ai-memory`, `pm-mcp-server`, `assistant-ui`, `gateway`) имеют локальные AGENTS.md и явно описаны как «add to, not override» корневой AGENTS.md (см. корневой AI-Assistant/AGENTS.md, строки 8–12). Но самой строки `extends ../AGENTS.md` в локальных файлах нет — это надо явно закрепить.

**Технические возможности harness** (проверено через claude-code-guide):
- Claude Code: **нет** официального механизма переопределить путь plan-файлов (нет env `CLAUDE_PLANS_DIR`, нет настройки `planMode.directory`, нет hooks `PrePlanCreate`/`PostExitPlanMode`).
- Codex CLI: аналогичной опции в `~/.codex/config.toml` нет.
- Единственный путь — правило в AGENTS.md, по которому агент сам в начале plan mode переводит работу в проектный draft, а harness-файл становится stub.

---

## Решения

### Решение 1. Модель наследования AGENTS.md (против дрейфа)

**Принцип (two-tier source of truth):**
- Глобальный `~/.claude/CLAUDE.md` (= `~/.codex/AGENTS.md`) — **единственный источник** universal-правил (миграция Q, plan files, обращение со старыми PM-MCP ID).
- Проектный `AGENTS.md` — **единственный источник** project-specific дополнений (B/C/D архитектурные инварианты, F/G UI, O release, проектные нюансы K/L).
- Это уточнение, не ослабление исходного принципа «AGENTS.md — single source of truth»: каждый файл остаётся единственным источником для своего слоя.

**Реализация:**
- Локальные `AGENTS.md` (включая подсистемы AI-Assistant и самостоятельные проекты) обязаны содержать в шапке явную строку:
  ```
  This AGENTS.md extends the global rules in ~/.claude/CLAUDE.md (Section A — universal)
  and, for subsystems, the parent AGENTS.md. Parent rules remain mandatory; this file
  only adds project-specific extensions.
  ```
  Эта строка — **единственный** обязательный новый текст в проектных AGENTS.md.
- В `_project_template/AGENTS.md` раздел **Q сохраняется как короткая ссылочная секция-заглушка** (не удаляется полностью), чтобы поиск по `Q` в проектных файлах не показывал «секции нет»:
  ```
  ### Q. Migration and legacy cleanup

  Universal migration discipline is defined in `~/.claude/CLAUDE.md` Section A and
  is mandatory for this project. Project-specific migration notes (if any) go here.
  ```
  Аналогично можно завести ссылочную секцию **R. Plan files** (опционально, если хотим явное место для project-specific нюансов).
- **Для Design-system, ANKI и минимальных Design-system** достаточно добавить ту же ссылочную строку в шапку: она делает Section A глобального CLAUDE.md обязательной для агента, читающего проектный AGENTS.md.
- **Существующие проектные AGENTS.md с уже содержащимся Q** не трогаем в рамках этого плана (tech debt от копи-паста); сведение к ссылочной модели — отдельная задача после стабилизации.

### Решение 2. Размещение файлов планов

**Принцип:** оба агента работают с одним файлом в проектной папке, начиная с самой первой итерации draft.

- **Локации:**
  - `<project>/docs/plans/_drafts/<slug>.md` — пока план обсуждается (итерации Claude↔Codex).
  - `<project>/docs/plans/<id>-<slug>.md` — после ExitPlanMode и создания PM-MCP задачи.
  - Для подсистем AI-Assistant — внутри подсистемы (`D:\GitHub\AI-Assistant\<sub>\docs\plans\`).
  - Для кросс-подсистемных планов — `D:\GitHub\AI-Assistant\docs\plans\`.
  - Для глобальных / out-of-project планов — `D:\GitHub\plans\_drafts\<slug>.md`.
- **Git-discipline (две политики, выбрать одну для каждого проекта):**
  - Минимальная: `docs/plans/_drafts/` в `.gitignore`. Финальные `docs/plans/<id>-<slug>.md` — tracked, удаляются `git rm` в commit-сообщении при закрытии задачи. История репо хранит, что план был, и в каком коммите удалён. Рекомендую этот вариант для проектов с активной работой.
  - Полная изоляция (если репозиторий не должен видеть планы вообще): `docs/plans/*.md` в `.gitignore` с исключением `!docs/plans/README.md`. Используется в проектах, которые отдают артефакты внешним consumers.
- Для видимой структуры в git создать `<project>/docs/plans/README.md` с описанием папки и политикой. Подпапка `_drafts/` создаётся агентом при первом плане (пустые папки в git не нужны).
- **Критерий удаления плана** (все условия одновременно):
  1. Связанная задача в PM-MCP закрыта (`Готово` или `Не актуально`).
  2. Итог записан в AI-memory как `decision` или `change`.
  3. Архитектурные причины, если есть, вынесены в `docs/adrs/` или проектную документацию.
  4. Файл не содержит уникальных acceptance criteria, не попавших в задачу или документацию.
- **Жизненный цикл:**
  1. Пользователь даёт задачу → Claude в plan mode пишет минимальный stub в `~/.claude/plans/<random>.md`: одну строку `См. план: <project>/docs/plans/_drafts/<slug>.md`.
  2. Параллельно Claude создаёт `<project>/docs/plans/_drafts/<slug>.md` и работает там.
  3. Codex запускается в `<project>` и читает тот же файл. Итерации правят diff, токены не тратятся на повтор.
  4. ExitPlanMode → создаются задачи → переименование draft в `<id>-<slug>.md` (без подпапки `_drafts/`). При cross-project плане — splittous по проектам с собственными ID, либо один файл с composite-именем (см. «нерешённые вопросы»).
  5. По завершении задачи план удаляется при выполнении 4 условий выше.
- **Исключение к правилу «File creation: ask first»** (раздел A глобального): создание/обновление `<project>/docs/plans/_drafts/<slug>.md` разрешено без отдельного запроса, **если** пользователь явно запустил планирование или попросил составить план. Это исключение прописывается в глобальный CLAUDE.md.

### Решение 3. Раздел Q в Design-system / ANKI / минимальных Design-system

Через ссылочную строку (Решение 1). Дополнительного дублирования Q.1–Q.3 в эти файлы не требуется: Section A глобального CLAUDE.md уже содержит Migration discipline и Documentation discipline, а локальная ссылка делает их обязательными для агента.

### Решение 4. Существующие 26 файлов планов

Разделение через критерий из Решения 2 (4 условия):

**Удалить (план реализован, контекст в AI-memory/git):**
`61-sharded-snowflake.md`, `nested-popping-flute.md`, `start-vbs-bubbly-clarke.md`, `expressive-marinating-peacock.md`, `serene-zooming-snowflake.md`, `compressed-orbiting-whistle.md`, `enchanted-snuggling-puppy.md`, `silly-snacking-zebra.md`, `vectorized-soaring-reef.md`, `2-majestic-frost.md`, `2-melodic-tome.md`, `fancy-humming-cat.md`, `tidy-dancing-pebble.md`, `twinkling-percolating-honey.md`, `wise-foraging-quokka.md`, `floating-dazzling-teapot.md`, `inherited-sauteeing-hamster.md`, `inherited-inventing-tiger.md`, `zweck-snappy-umbrella.md`.

Перед удалением для каждого файла проверить: есть ли соответствующая запись в AI-memory. Если нет — создать минимальную запись подходящего kind (`change` для технических изменений, `decision` для решений, `note` для промежуточных выводов) с итогом, списком файлов и источником. Не делать `decision` автоматически на каждый план — kind зависит от природы выполненной работы.

Перед перемещением 7 незавершённых планов: если целевой файл уже существует, не перезаписывать. Сравнить содержимое; либо выбрать новое имя, либо объединить вручную.

**Перенести в `<project>/docs/plans/_drafts/`:**
- `cozy-splashing-waterfall.md` → `D:\GitHub\AI-Assistant\docs\plans\_drafts\autostart-ecosystem.md`
- `gateway-tasks-draft.md` → `D:\GitHub\AI-Assistant\gateway\docs\plans\_drafts\real-backends-tasks.md`
- `chatgpt-effervescent-moore.md`, `chatgpt-woolly-acorn.md` → `D:\GitHub\AI-Assistant\ai-memory\docs\plans\_drafts\chatgpt-read-public.md` и `chatgpt-write-gateway.md`
- `hidden-knitting-flurry.md` → `D:\GitHub\AI-Assistant\docs\plans\_drafts\daily-backup.md`
- `d-1-eintrittscheckliste-docx-immutable-crescent.md` → `D:\GitHub\Verua_automation\docs\plans\_drafts\eintrittscheckliste.md`
- `d-xlsx-linear-unicorn.md` → `D:\GitHub\Best_photo_ai\docs\plans\_drafts\anchor-aware-scoring.md`

### Решение 5. Сквозная нумерация PM-MCP — выносится в отдельный ADR + план

В рамках этого плана **только подготовка**:
- Завести новый файл `D:\GitHub\AI-Assistant\docs\adrs\0002-global-task-numbering.md` с фиксацией намерения и открытых вопросов (см. ниже).
- Создать draft плана миграции `D:\GitHub\AI-Assistant\pm-mcp-server\docs\plans\_drafts\global-numbering-migration.md` — пустой шаблон, который будет заполняться в отдельной плановой сессии.

Перед началом самой миграции (в отдельном плане) обязательно зафиксировать:

- **Архитектура:** глобальный sequence в отдельной таблице (`id_sequence` или SQLite AUTOINCREMENT с уникальным индексом). **Не использовать `SELECT MAX+1`** — гонки при параллельных `create_task`. Все insert-операции в транзакции `BEGIN IMMEDIATE`.
- **Формат ID:** `#1`, `#2`, ... без префикса (совпадает с текущим отображением); или `#001` с zero-padding — вопрос для ADR.
- **Бэкап SQLite:** при включённом WAL простое копирование `pm-mcp.sqlite3` даёт битый файл. Обязательно: либо остановить сервер, либо использовать SQLite online backup API (`.backup` команда). Копировать также `-wal` и `-shm`. Сохранить mapping artifact `old_project + old_id -> new_id` в отдельном файле.
- **Структура `metadata.legacy_id`:** строка `<project> #<n>` или объект `{project_path, project_slug, local_id, display}` — вопрос для ADR. Объект надёжнее для машинного поиска.
- **Регэксп-скрипт для docs:** строго ограниченный target (`git ls-files '*.md'` внутри каждого репозитория, exclude `.git`, `node_modules`, generated, archives), dry-run diff, ручная проверка перед применением.
- **AI-memory тексты:** не трогать (исторический контекст), `metadata.task` тоже. Lookup через `legacy_id` и подстроку в тексте.

---

## Открытые вопросы (для ADR-0002 / отдельного плана PM-MCP)

1. Является ли `D:\GitHub\AI-Assistant` (корень монорепо) самостоятельным `project_path` в PM-MCP, или только контейнером для подсистем? Сейчас registry знает только 4 подсистемы.
2. Формат ID: `#1` без padding или `#0001` с padding?
3. `metadata.legacy_id`: строка или структура с полями?
4. Должны ли draft-планы попадать в git, или строго `.gitignore`? (План предполагает gitignore — подтвердить.)
5. Для cross-project плана после создания нескольких задач: один общий файл, split по проектам, или composite-имя с несколькими ID?
6. Можно ли удалять старый план без AI-memory записи, если задача закрыта только в git/PM-MCP? (План говорит: нет, нужно создать summary-запись перед удалением.)
7. PM-MCP клиенты (Assistant-UI, прямые consumers AI-memory): где ещё используются старые ID, что сломается?

---

## Изменения в файлах (только этот план — шаги 1–4)

### 1. Глобальный `C:\Users\Zaxva\.claude\CLAUDE.md` (hard-linked с `~/.codex/AGENTS.md`)

В разделе A (Universal preferences) добавить:

**Plan files:**
- Файлы плана ведутся в `<project>/docs/plans/_drafts/<slug>.md` (kebab-case, ≤40 символов), не в `~/.claude/plans/` или `~/.codex/plans/`.
- При входе в plan mode агент первым действием создаёт draft в проектной папке. Harness-файл в `~/.claude/plans/<random>.md` содержит только строку `См. план: <project>/docs/plans/_drafts/<slug>.md`.
- После ExitPlanMode и создания задачи: `<project>/docs/plans/<id>-<slug>.md`.
- После завершения задачи план удаляется при выполнении 4 условий (закрытая задача, AI-memory запись, ADR при необходимости, нет orphan acceptance criteria).
- `docs/plans/_drafts/` в `.gitignore` каждого проекта.
- **Исключение к «File creation: ask first»:** создание/обновление файла в `<project>/docs/plans/_drafts/` разрешено без отдельного запроса, если пользователь явно запустил планирование.

**PM-MCP task IDs (намерение, реализация — отдельный план):**
- Целевая модель — единый глобальный счётчик `#<n>` для всех work items. Старые локальные ID сохраняются только в `metadata.legacy_id` для исторического lookup.
- Детали и порядок миграции — в ADR-0002 и отдельном плане `pm-mcp-server/docs/plans/_drafts/global-numbering-migration.md`.

**Уточнить наследование (раздел A):** добавить пункт «Локальный `AGENTS.md` проекта/подсистемы расширяет глобальные правила Section A. Универсальные правила обязательны и без дублирования.»

### 2. Шаблон `D:\GitHub\_project_template\AGENTS.md`

Добавить в шапку (между заголовком и `## Quick start`):

```
This AGENTS.md extends the global rules in ~/.claude/CLAUDE.md (Section A — universal).
Parent rules remain mandatory; this file only adds project-specific extensions
(B/C/D architecture invariants, F/G UI, O release, project-specific K/L details).
```

Раздел Q **удалить из шаблона** (он становится частью universal). В будущих новых проектах AGENTS.md будет короче.

`## Related docs` дополнить: «`docs/plans/` — active and pending plans (drafts gitignored).»

### 3. Существующие проектные AGENTS.md — точечная правка (8 файлов)

Только одно изменение в каждом: вставить ту же ссылочную строку в шапку. Большие блоки Q/R/K не дублируем.

- `D:\GitHub\AI-Assistant\AGENTS.md` (корневой)
- `D:\GitHub\Best_photo_ai\AGENTS.md`
- `D:\GitHub\Verua_automation\AGENTS.md`
- `D:\GitHub\GmbH\AGENTS.md`
- `D:\GitHub\Hauswirtschaftiche_Pflegeleistungen\AGENTS.md`
- `D:\GitHub\Stalking_offline\AGENTS.md`
- `D:\GitHub\Design-system\AGENTS.md`
- `D:\GitHub\ANKI\AGENTS.md`

### 4. Локальные AGENTS.md подсистем AI-Assistant и минимальных Design-system (6 файлов)

Та же ссылочная строка (для подсистем AI-Assistant — со ссылкой на корневой AI-Assistant/AGENTS.md как parent):

- `D:\GitHub\AI-Assistant\ai-memory\AGENTS.md`
- `D:\GitHub\AI-Assistant\pm-mcp-server\AGENTS.md`
- `D:\GitHub\AI-Assistant\assistant-ui\AGENTS.md`
- `D:\GitHub\AI-Assistant\gateway\AGENTS.md`
- `D:\GitHub\Best_photo_ai\Design-system\AGENTS.md`
- `D:\GitHub\Verua_automation\Design-system\AGENTS.md`

### 5. `.gitignore` и `docs/plans/README.md`

- В `.gitignore` каждого затронутого репозитория добавить `docs/plans/_drafts/` (минимальная политика — финальные планы tracked, удаляются `git rm` при закрытии задачи).
- Создать `<project>/docs/plans/README.md` с описанием папки и политикой удаления планов. Этот файл — единственная видимая «структурная» точка в git; подпапка `_drafts/` создаётся агентом при первом плане.
- Папка `D:\GitHub\plans\_drafts\` (out-of-project) создаётся агентом при первой необходимости (не в репозитории, gitignore не нужен).

### 6. Чистка `C:\Users\Zaxva\.claude\plans\`

- Перед удалением каждого реализованного плана проверить наличие AI-memory записи; если нет — создать `decision`/`change` summary одной строкой.
- Удалить 19 файлов реализованных планов.
- Переместить 7 файлов незавершённых планов в проектные `docs/plans/_drafts/` с переименованием по схеме.

### 7. Подготовка ADR-0002 (без раздела Decision)

- Создать `D:\GitHub\AI-Assistant\docs\adrs\0002-global-task-numbering.md` со статусом `Proposed` и разделами: **Context** (зачем), **Options** (рассмотренные альтернативы: глобальный sequence vs префикс по подсистеме vs status quo), **Constraints** (WAL/SQLite, гонки create_task, обратная совместимость с AI-memory текстами, отсутствие переписывания коммитов), **Open questions** (список из «Открытых вопросов» этого плана), **Status: Proposed**, **Decision: Not decided yet**. Раздел Decision **без placeholder-текста** — заполнится после следующей плановой сессии.
- Создать пустой draft `D:\GitHub\AI-Assistant\pm-mcp-server\docs\plans\_drafts\global-numbering-migration.md` со ссылкой на ADR-0002 — рабочая зона для следующей плановой сессии.

---

## Порядок исполнения (последовательность отдельных задач в PM-MCP)

1. **Глобальный CLAUDE.md**: Plan files + PM-MCP intent + наследование. Это база, на которой держится вся остальная документация.
2. **Шаблон `_project_template/AGENTS.md`**: ссылочная строка + удаление Q (он теперь universal) + `## Related docs`.
3. **Точечная правка 14 проектных/локальных AGENTS.md**: вставка ссылочной строки.
4. **Создание папок `_drafts/` и обновление `.gitignore`**.
5. **Чистка планов**: создать недостающие AI-memory записи → удалить 19 файлов → переместить 7 файлов.
6. **Создать ADR-0002 (Proposed) и пустой draft миграции PM-MCP**.

Все шаги безопасны и обратимы. После шага 6 — отдельная плановая сессия по миграции PM-MCP.

---

## Верификация

**После шага 1:**
- `~/.claude/CLAUDE.md` содержит новые блоки Plan files и PM-MCP intent в Section A.
- Проверить, что hard-link с `~/.codex/AGENTS.md` сохраняется (`fsutil hardlink list` или `Get-Item ... | Select FullName, Target`).

**После шага 2:**
- В `_project_template/AGENTS.md` есть ссылочная строка в шапке.
- Раздел Q удалён (он стал universal).

**После шагов 3–4:**
- В каждом из 14 затронутых AGENTS.md есть ссылочная строка.
- В каждом затронутом `.gitignore` есть строка `docs/plans/_drafts/`.
- В каждом проекте создан `docs/plans/README.md` с политикой.
- `git status` в каждом репозитории показывает только запланированные изменения. Подпапки `_drafts/` отсутствуют — это нормально, они создаются агентом при первом плане.

**После шага 5:**
- `C:\Users\Zaxva\.claude\plans\` содержит только текущий рабочий файл и/или новые drafts.
- Для каждого удалённого реализованного плана в AI-memory есть запись подходящего kind (поиск `mcp__AI-memory__search_memory` с query=slug → находит).
- Все 7 перенесённых drafts существуют в проектных папках (с конфликт-чеком: ни один не перезаписал существующий файл).

**После шага 6:**
- `D:\GitHub\AI-Assistant\docs\adrs\0002-global-task-numbering.md` существует со статусом `Proposed`, разделы Context/Options/Constraints/Open questions заполнены, Decision = `Not decided yet`.
- Существует пустой draft `D:\GitHub\AI-Assistant\pm-mcp-server\docs\plans\_drafts\global-numbering-migration.md` со ссылкой на ADR-0002.

**Dry verification workflow** (без создания лишнего harness-файла):
- В новом `~/.claude/CLAUDE.md` и в `_project_template/AGENTS.md` инструкции для plan mode прочитываются корректно: путь, slug-формат, stub-содержимое harness-файла, исключение для «File creation: ask first».
- В AI-memory записан summary новых правил подходящего kind (`decision` для модели наследования + локации планов, `change` для удалённых/перемещённых файлов).
- Первая реальная проверка workflow — при следующем естественном планировании, без искусственного запуска.
