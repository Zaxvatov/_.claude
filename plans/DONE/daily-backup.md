# План: ресурсные состояния локальных процессов в портфеле D:\GitHub

## Контекст

Цель — заложить во все проекты портфеля единый стандарт управления
ресурсным состоянием локальных процессов и сервисов: `Active` / `Idle` /
`Stopped`. Без такого стандарта фоновые сервисы (MCP-серверы, memory,
agents UI) держат normal priority и расходуют ресурсы машины пользователя
даже в простое; тяжёлые pipeline-процессы (фото, видео, browser
automation, build) не имеют согласованных правил повышения/понижения
приоритета.

Что должна дать эта серия задач:

- единый словарь состояний и правил их применения, который наследуется
  всеми будущими проектами через `_project_template`;
- ядро управления состояниями в `PM-MCP-server` (Process State Manager)
  и сам управляемый сервис `AI-memory` как первый пример;
- интеграция через `PM-Agent` (планирование) и `Assistant-UI`
  (отображение/действия), а не прямой импорт;
- правила в прикладных проектах: heavy compute не понижать в Idle во
  время выполнения, public API / MCP / pipeline артефакты не ломать.

Важные ограничения, согласованные с пользователем:

- TASKS.md больше не трогаем — он легаси, скоро будет удалён. Все задачи
  заводятся через **PM-MCP task tools** (Assistant-UI / PM-MCP CLI).
- Минимальное архитектурное правило добавляется **только в AGENTS.md**
  затронутых проектов; CLAUDE.md и код не трогаем.
- Никакой реализации в этой итерации — только задачи и правило.

Технический момент: в текущем окружении Claude Code PM-MCP-инструменты
не подключены, поэтому сами задачи в PM-MCP заведёт пользователь
по готовым формулировкам ниже (или через `python -m app.main` сценарий
PM-Agent). Файловые правки AGENTS.md я внесу сам после approve.

---

## Итог в одном экране

| Проект | Приоритет | Тип задачи в PM-MCP | Правка AGENTS.md |
|---|---|---|---|
| AI-memory | высокий | реализация состояний процесса | да |
| PM-MCP-server | высокий | архитектура Process State Manager | да |
| PM-Agent | средний | учёт состояний при планировании | да |
| Assistant-UI | средний | отображение и управление состояниями | да |
| Best_photo_ai | средний | ресурсные режимы в photo/video pipeline | да |
| Verua_automation | средний | ресурсные режимы для playwright/web-сервера | да |
| Stalking_offline | средний | ресурсные режимы для vault/web-viewer | да |
| Hauswirtschaftiche_Pflegeleistungen | средний | ресурсные режимы для PDF pipeline и viewer | да |
| ANKI | средний | проектировать будущие фоновые сервисы с состояниями | да |
| Design-system | средний | только правило в AGENTS.md (своих процессов нет) | да |
| _project_template | средний | стандарт ресурсных состояний в шаблоне | да |

---

## Часть 1. Задачи в PM-MCP (заводит пользователь)

Формат для каждой задачи:
- **Project** — целевой проект в PM-MCP.
- **Title** — название задачи.
- **Priority** — высокий / средний.
- **Status** — Бэклог.
- **Description** — содержание (можно вставить как есть).

### 1.1 AI-memory  [приоритет: высокий]

**Title:** Добавить поддержку ресурсных состояний процесса AI-memory

**Description:**
- Описать и затем реализовать режимы `Active` / `Idle` / `Stopped`
  для daemon-процесса AI-memory.
- Idle: безопасно понижать приоритет процесса (Windows: BELOW_NORMAL +
  по возможности EcoQoS / Efficiency mode).
- Active: автоматически возвращать normal priority при поступлении
  MCP-запроса; не ломать stdio bridge и `127.0.0.1:8765` listener.
- Не связывать AI-memory напрямую с PM-MCP-server — управление состоянием
  должно быть доступно извне через отдельный публичный интерфейс
  (MCP tool / CLI команда), который потом вызовет PM-MCP-server как
  обычный клиент.
- Сохранить single-instance поведение daemon (см. готовую задачу #092).
- Логировать переходы состояний; покрыть тестами.

### 1.2 PM-MCP-server  [приоритет: высокий]

**Title:** Добавить архитектуру Process State Manager для локальных процессов

**Description:**
- Спроектировать новый слой `process_state` внутри PM-MCP-server.
- Поддерживаемые состояния: `Active` / `Idle` / `Stopped`.
- Реестр управляемых процессов: проект → набор процессов с
  идентификацией (имя, способ запуска, способ обнаружения PID),
  декларативно описанный в config.
- Allowlist проектов и процессов — только то, что в нём, может управляться;
  всё остальное (системные процессы Windows, сторонние приложения)
  никогда не трогается.
- Управление приоритетом процесса (нормальный / пониженный +
  EcoQoS), не завершение, если нет явного правила Stopped.
- Аудит-лог всех переходов (через AI-memory `kind=change` или отдельный
  лог-файл, по решению ARCHITECTURE.md).
- Публичные MCP tools, минимальный набор: `list_managed_processes`,
  `get_process_state`, `set_process_state(project, process, state)`,
  `register_process(...)`.
- В будущем интеграция с AI-memory — только через MCP-клиент
  (как любой другой потребитель), без прямого импорта.
- Завести ADR в `PM-Agent/DOCS/System/ARCHITECTURE_DECISIONS.md`
  до реализации.

### 1.3 PM-Agent  [приоритет: средний]

**Title:** Учитывать ресурсные состояния процессов при планировании задач

**Description:**
- При построении плана сценариев читать текущее состояние процессов
  из PM-MCP-server (`list_managed_processes`).
- Перед выполнением задачи переводить нужные сервисы в `Active`
  через `set_process_state(...)`; после завершения или при простое —
  возвращать в `Idle`.
- Не запускать тяжёлые процессы без необходимости; учитывать
  ограничения железа пользователя (декларируется в config PM-Agent).
- Перед изменением состояния процессов выводить пользователю
  рекомендацию/предложение (approval gate в существующем стиле PM-Agent).
- Не завершать процессы в `Stopped` без явного правила в allowlist.
- Зависимость: PM-MCP-server задача 1.2.

### 1.4 Assistant-UI  [приоритет: средний]

**Title:** Добавить отображение и управление ресурсными состояниями процессов

**Description:**
- Новая панель / блок dashboard со списком локальных процессов и их
  текущим состоянием (`Active` / `Idle` / `Stopped`).
- Показывать, какие проекты/сервисы сейчас активны и потребляют ресурсы.
- Безопасные действия в UI: «Перевести в Idle», «Вернуть в Active»,
  «Остановить» (последнее — только если есть явное правило в allowlist).
- Подтверждение пользователем для любого действия, меняющего состояние.
- Стили — только из `../Design-system`; не создавать локальные стили.
- Источник данных и операций — PM-MCP-server tools из задачи 1.2.
- Зависимость: PM-MCP-server задача 1.2.

### 1.5 Best_photo_ai  [приоритет: средний]

**Title:** Учитывать ресурсные режимы при запуске photo/video pipeline

**Description:**
- Перед тяжёлыми этапами pipeline (`01..12` фото, `13..20` видео,
  `28..31` обучение) переводить процесс в `Active` (нормальный приоритет).
- Не переводить ML / torch / image-processing этапы в Efficiency/Idle
  во время обработки.
- После завершения pipeline или во время ожидания пользовательского
  review (`24_run_review_web.py`) можно понижать приоритет до `Idle`.
- Сохранить текущую структуру pipeline и `Scripts/config_paths.py`
  как single source of truth; не переименовывать существующие файлы,
  константы и артефакты.
- Управление состоянием — через PM-MCP Process State Manager,
  без прямого импорта в Scripts.

### 1.6 Verua_automation  [приоритет: средний]

**Title:** Учитывать ресурсные режимы для playwright и web-сервера

**Description:**
- Web-сервер `verua_automation.web_main` (порт 8010) и фоновые
  Playwright-браузеры — перевести в `Idle` при отсутствии активных
  задач, поднимать в `Active` перед запуском сценариев.
- Не понижать приоритет во время активного browser automation
  сценария (chromium процессы могут таймаутить).
- Не трогать системные процессы Windows.
- Сохранить точку входа `.\!start.vbs` → `.\start.ps1` и контракт
  `.\verify.ps1`.
- Управление — через PM-MCP Process State Manager (allowlist).

### 1.7 Stalking_offline  [приоритет: средний]

**Title:** Учитывать ресурсные режимы для vault и web-viewer

**Description:**
- Локальный web-viewer (SafeOffline) и фоновые задачи переводить в
  `Idle` при простое; возвращать в `Active` при действиях пользователя.
- Не трогать crypto-операции и операции с зашифрованным SQLite во
  время их выполнения (security rules I.1–I.6 в AGENTS.md
  не нарушать).
- Не менять `release-versions.json` и portable build процесс.
- Управление — через PM-MCP Process State Manager.

### 1.8 Hauswirtschaftiche_Pflegeleistungen  [приоритет: средний]

**Title:** Учитывать ресурсные режимы для PDF pipeline и viewer

**Description:**
- Перед генерацией PDF / Swiss QR Bill переводить процесс в `Active`.
- Локальный viewer и idle-режимы между прогонами — `Idle`.
- Не понижать приоритет во время генерации PDF, чтобы не раскачивать
  тайминги верстки.
- Сохранить `setup.ps1` / `build_portable.ps1` контракт.
- Управление — через PM-MCP Process State Manager.

### 1.9 ANKI  [приоритет: средний]

**Title:** Проектировать новые фоновые сервисы с учётом Active/Idle/Stopped

**Description:**
- На текущий момент в проекте нет долгоживущих процессов, требующих
  управления. Задача — фиксирующая: при добавлении любого фонового
  сервиса/демона он должен сразу проектироваться с поддержкой
  `Active` / `Idle` / `Stopped` и регистрироваться в allowlist
  PM-MCP Process State Manager.
- Heavy compute (импорт колод, генерация карточек) — не понижать
  в Idle во время выполнения.

### 1.10 Design-system  [приоритет: средний]

**Title:** Зафиксировать в правилах учёт ресурсных состояний для будущих сервисов

**Description:**
- В Design-system нет своих локальных процессов. Задача —
  организационная: убедиться, что AGENTS.md содержит правило про
  Active/Idle/Stopped (см. часть 2 ниже), чтобы любой будущий
  preview-server или auxiliary tool учитывал стандарт.

### 1.11 _project_template  [приоритет: средний]

**Title:** Добавить стандарт архитектуры ресурсных состояний процессов

**Description:**
- В шаблон будущих проектов добавить правило проектирования
  процессов с состояниями `Active` / `Idle` / `Stopped`.
- Зафиксировать:
  - когда процесс можно переводить в `Idle`;
  - какие процессы нельзя трогать (системные, чужие);
  - требование allowlist для управляемых процессов;
  - требование логирования всех изменений состояния;
  - правило: heavy compute задачи не переводить в Efficiency/Idle
    во время выполнения;
  - правило: изменение состояния процесса не должно ломать публичные
    API, MCP-интерфейсы и pipeline артефакты.
- Сама правка ARCHITECTURE.md / docs шаблона — отдельной задачей
  при реализации; в этой задаче только постановка.

---

## Часть 2. Правка AGENTS.md (вношу я)

В каждый AGENTS.md из списка ниже добавляется один и тот же короткий
раздел на английском (соответствует требованию «AGENTS.md is written
entirely in English»). Размещение — после секции `M: Testing` либо
в существующей секции `C: Architecture invariants` / `D: Data flow
invariants`, по факту структуры конкретного файла. Никакие другие
строки не трогаются.

Текст для вставки (единый, минимальный):

```markdown
## Process resource states

- Long-running processes and background services in this project must be
  designed with three explicit resource states: `Active`, `Idle`,
  `Stopped`.
  - `Active` — normal or elevated priority for actual work, pipelines,
    compute, request handling.
  - `Idle` — lowered priority (Windows `BELOW_NORMAL` and, when
    possible, EcoQoS / Efficiency mode) for background or waiting
    state.
  - `Stopped` — process fully stopped; only allowed when an explicit
    rule exists.
- Heavy compute steps (ML, torch, image/video processing, PDF
  generation, browser automation runs) must not be moved into Idle /
  Efficiency mode while they are executing.
- Process state must be controlled from outside via the PM-MCP Process
  State Manager (allowlist-based). Do not import PM-MCP-server or
  AI-memory directly to manage process state.
- State transitions must not break public APIs, MCP interfaces,
  pipeline artifacts, or single-instance daemon contracts.
- All state transitions are logged (audit trail).
```

Файлы для правки (11 шт.):

- `D:\GitHub\AI-memory\AGENTS.md`
- `D:\GitHub\PM-MCP-server\AGENTS.md`
- `D:\GitHub\PM-Agent\AGENTS.md`
- `D:\GitHub\Assistant-UI\AGENTS.md`
- `D:\GitHub\Best_photo_ai\AGENTS.md`
- `D:\GitHub\Verua_automation\AGENTS.md`
- `D:\GitHub\Stalking_offline\AGENTS.md`
- `D:\GitHub\Hauswirtschaftiche_Pflegeleistungen\AGENTS.md`
- `D:\GitHub\ANKI\AGENTS.md`
- `D:\GitHub\Design-system\AGENTS.md`
- `D:\GitHub\_project_template\AGENTS.md`

Правила правки:

- Только append нового раздела `## Process resource states` (или вставка
  в логическое место в зависимости от структуры конкретного файла —
  без переписывания соседних разделов).
- Не менять нумерацию существующих разделов (A–P).
- Не трогать сам код, ARCHITECTURE.md, README.md, CLAUDE.md, TASKS.md,
  pyproject.toml, скрипты, Design-system токены.

---

## Часть 3. Чего я НЕ делаю в этой итерации

- Не редактирую TASKS.md ни в одном проекте (легаси).
- Не редактирую CLAUDE.md (по согласованию).
- Не редактирую ARCHITECTURE.md / docs / код / тесты.
- Не реализую Process State Manager и состояния процесса AI-memory.
- Не пишу ADR в `PM-Agent/DOCS/System/ARCHITECTURE_DECISIONS.md`
  (это часть задачи 1.2 при реализации).
- Не завожу задачи в PM-MCP сам — у меня нет соответствующего
  MCP-инструмента в окружении; пользователь либо заведёт их через
  Assistant-UI / PM-MCP CLI по готовым формулировкам выше, либо я
  перенесу формулировки в Assistant-UI вручную в отдельной
  follow-up сессии.

## Часть 4. Что требует отдельного согласования перед реализацией

- ADR по Process State Manager в `ARCHITECTURE_DECISIONS.md`:
  место хранения allowlist, формат audit-лога, способ регистрации
  процессов (config-файл vs runtime-API).
- Способ применения EcoQoS на Windows (PowerShell `Set-Process` /
  WinAPI `SetProcessInformation`); потребуется elevated права?
- Сценарий «всё PM-MCP / Assistant-UI / AI-memory упало» — кто
  возвращает процессы из Idle в Active без управляющего слоя
  (страховочный механизм в каждом сервисе).
- Перечень EXE/процессов в allowlist для каждого проекта — отдельный
  config до реализации задачи 1.2.

---

## Часть 5. Verification (как проверить итог этой итерации)

Эта итерация ограничена правкой AGENTS.md и подготовкой формулировок;
поэтому верификация лёгкая:

1. Для каждого из 11 AGENTS.md убедиться, что:
   - добавлен ровно один раздел `## Process resource states`;
   - текст идентичен шаблону из части 2;
   - все остальные строки совпадают с предыдущей версией (diff
     показывает только добавление).
   ```bash
   git -C D:/GitHub/<project> diff AGENTS.md
   ```

2. Убедиться, что **никакие другие файлы** не изменены ни в одном из
   11 проектов:
   ```bash
   git -C D:/GitHub/<project> status
   ```

3. После заведения задач в PM-MCP — проверить, что они появились
   в Бэклоге соответствующих проектов через Assistant-UI
   (`/project/<name>` страница).

4. Реализация (Process State Manager и т.д.) выполняется отдельными
   подходами по уже заведённым задачам — вне рамок этой итерации.

---

## Критические файлы (только AGENTS.md, всё остальное — read-only)

- D:\GitHub\AI-memory\AGENTS.md
- D:\GitHub\PM-MCP-server\AGENTS.md
- D:\GitHub\PM-Agent\AGENTS.md
- D:\GitHub\Assistant-UI\AGENTS.md
- D:\GitHub\Best_photo_ai\AGENTS.md
- D:\GitHub\Verua_automation\AGENTS.md
- D:\GitHub\Stalking_offline\AGENTS.md
- D:\GitHub\Hauswirtschaftiche_Pflegeleistungen\AGENTS.md
- D:\GitHub\ANKI\AGENTS.md
- D:\GitHub\Design-system\AGENTS.md
- D:\GitHub\_project_template\AGENTS.md
