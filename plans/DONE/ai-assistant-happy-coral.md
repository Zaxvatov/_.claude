# Дорожная карта целей: time-anchored intent canvas (`/roadmap`)

Slug: `ai-assistant-happy-coral` (не переименовывать).
PM-MCP work items (status «К выполнению», исполнитель — Codex):
- **#805** (root) — Phase 0: ADR-0013 + docs.
- **#806** (pm-mcp-server) — `set_work_item_goals` + round-trip тест (контракт №1).
- **#807** (assistant-ui) — Phase 1: `GET /api/roadmap` + `app/roadmap.py`. ← dep #806.
- **#808** (assistant-ui) — Phase 2: React-остров scaffolding. ← dep #805.
- **#809** (assistant-ui) — Phase 3: read-only холст. ← dep #807, #808.
- **#810** (assistant-ui) — Phase 4a: панель + write-эндпоинты. ← dep #809, #806.
- **#811** (assistant-ui) — Phase 4b: draw.io-коннектор. ← dep #810.

Порядок (topo): #805 ∥ #806 → #807 ∥ #808 → #809 → #810 → #811.

## Context

Пользователь хочет дашборд, который показывает всю «пирамиду, положенную на бок»:
от самой глобальной цели справа (далёкий горизонт) через под-цели до атомарных
задач слева (сегодня), посаженных на календарную временную ось. Нижестоящие узлы
схлопываются по клику на вышестоящий. Это canvas с блоками, связями, тегами и
прогрессом. Позже на дни сядут события Google Calendar, а агент будет
анализировать риски и приоритеты.

**Ключевой вывод исследования:** модель данных под это уже есть и менять её не
нужно. ADR-0009 ввёл intent tree:

- `Goal` — единственный intent-узел; родитель в `metadata.parent_goal_id`,
  обоснование в `metadata.why`, высота в `metadata.horizon`
  (`life → multi_year → year → quarter → month`). Это и есть «пирамида»:
  `life` — вершина справа, атомарные задачи — слева сегодня.
- `WorkItem` — операционная задача, связана с ближайшей целью через `goal_ids`.
- И `Goal`, и `WorkItem` уже имеют `due_date`
  ([contracts.py](pm-mcp-server/app/schemas/contracts.py:27)).
- Read-tools готовы и проброшены в Assistant-UI: `get_goal_tree`,
  `get_goal_path`, `get_goal_decomposition`, `recommend_next_global` (отдаёт
  `why_path` + score). Страница [/tree](assistant-ui/app/templates/tree.html)
  (ADR-0009, сделана сегодня) рендерит тот же forest вложенным списком.

Значит, это **новая визуализация поверх существующих данных**, а не новая
модель. Основная новизна — сам холст-таймлайн и редактирование связей.

**Решения пользователя (Q&A):**

1. **Отдельная страница** — холст живёт как новый `/roadmap`; `/tree` остаётся
   лёгким списком и ретайрится позже, когда холст докажет себя (migration
   discipline: сначала строим новое).
2. **Ось — карта с масштабом** (день → неделя → месяц → год, «как на картах»),
   диапазон пока ограничить **2040 годом** (2026-01-01 → 2040-12-31).
3. **MVP включает редактирование**: создание задач, создание промежуточного
   узла с перепривязкой нижестоящих/вышестоящих блоков, **коннектор как в
   draw.io**.
4. **Современный стек** — пользователь допускает React: «без реакта тут никуда»,
   нужно «современное решение с большими возможностями», т.к. **позже интерфейс
   будут полностью перерисовывать**. React-остров строим как плацдарм будущего
   переписывания UI, но (см. ниже) — как строго ограниченное отклонение, не как
   новый default.

## Подход (recommended)

Новая страница `/roadmap` в Assistant-UI как **React-остров**: один смонтированный
React-апп для холста, собираемый Vite, отдаётся статическим бандлом из FastAPI,
ходит в существующие и новые `/api/*` JSON-эндпоинты. Остальное приложение
остаётся Jinja + vanilla JS — blast radius минимален.

Холст — **React Flow** (`@xyflow/react`): node/edge canvas с handles-коннекторами
«как в draw.io», pan/zoom, кастомные узлы. Поверх него своё:

- **Временная линейка** (фоновый слой): линейная шкала `date → X`, домен
  2026-01-01 → 2040-12-31, маркер «сегодня». LOD по уровню зума из
  `useViewport()` (год → квартал → месяц → неделя → день), как масштаб карты.
  Зум React Flow геометрический (как карта); семантику дают только подписи и
  плотность тиков.
- **Позиции — data-driven, layout считает фронт.** Бэкенд НЕ отдаёт пиксельные
  координаты: он отдаёт нормализованный граф (`date` / `horizon` / `sort` /
  `lane` / `meta`), а React считает x/y под текущий viewport, zoom и collapsed
  state. `X = timeScale(due_date)`; для узлов без даты — производная от `horizon`
  относительно сегодня (`month → +1мес`, `quarter → +3мес`, `year → +1год`,
  `multi_year → +3года`, `life → правый край / дорожка «Когда-нибудь»`).
  Инвариант пирамиды: `parent.X = max(horizon-X, max(children.X) + margin)` —
  родитель всегда правее детей. **Перенос даты перетаскиванием отложен** (Phase
  5), чтобы X не рассинхронился с данными.
- **Y — вертикальная упаковка дерева** на фронте (лист → слот, родитель →
  среднее детей; свёрнутое поддерево → один слот).
- **Свернуть/развернуть** — фильтрация nodes/edges по множеству collapsed.
- **Узлы** — кастомные React-компоненты в стиле MD3 (токены Design-system):
  `GoalNode` vs `TaskNode`, бейджи (horizon / статус / приоритет / цвет проекта),
  прогресс, кнопка сворачивания, handles для коннектора.

Источник истины — PM-MCP (никакой второй копии дерева в Assistant-UI,
инвариант [assistant-ui/AGENTS.md](assistant-ui/AGENTS.md)). Чтение через
`get_goal_tree` (+ work items), запись через существующие write-tools. Сервер уже
валидирует DAG при записи (cycle / dangling_parent / archived_parent /
invalid_horizon — [goal_tree.py](pm-mcp-server/app/goal_tree.py:74)), поэтому
перепривязка блоков безопасна на бэкенде.

## Канонические контракты (must-get-right)

Три места, где легко породить расхождение — фиксируем явно:

1. **Один канонический путь task→goal — `work_item.goal_ids`** (ADR-0009; именно
   его читают `_goal_matches_task` [server.py:513](pm-mcp-server/server.py:513) и
   `chain_alignment_score`/`recommend_next_global`
   [server.py:570](pm-mcp-server/server.py:570)). Сейчас `goal_ids` собирается из
   текстового маркера `[goal: id]` в `task.text` (парсинг в
   [server.py:445](pm-mcp-server/server.py:445) и отдельно в
   [registry.py:644](pm-mcp-server/app/adapters/registry.py:644)), а `WorkItem`
   строится из этого текста, **не** из `related_goals`, который при этом принимает
   `update_task` ([task_store.py:181](pm-mcp-server/app/task_store.py:181)).
   UI-канвас не должен редактировать скрытый `[goal: id]` в title/text. Поэтому
   **обязательно** (не fallback) добавляем тонкий структурный write-tool
   `set_work_item_goals(project_path, task_id, goal_ids)` и тестом доказываем, что
   `get_goal_tree` и `recommend_next_global` видят результат. Канвас пишет связь
   task→goal **только** через него и **не** использует `link_work_item_to_goal`
   (он пишет в отдельный `goal.metadata.work_item_ids` и не влияет на scoring —
   [server.py:3997](pm-mcp-server/server.py:3997)). Goal↔goal (декомпозиция) —
   единственный путь `update_goal(metadata.parent_goal_id)`.
2. **`create_task` требует `project_path`** (K.2/K.4); portfolio-wide canvas не
   угадывает subsystem. Правило: default = `goal.metadata.project_path` целевой
   цели; если его нет — UI **обязывает выбрать проект** перед созданием задачи.
3. **Optimistic concurrency на дереве целей.** Все goal write-tools принимают
   `expected_mtime_ns` [server.py:3958](pm-mcp-server/server.py:3958), адаптер
   отдаёт `file_state{mtime_ns}` ([goals.py](pm-mcp-server/app/adapters/goals.py)).
   `GET /api/roadmap` включает `file_state`; write-эндпоинты передают
   `expected_mtime_ns`; `goals_file_conflict` → HTTP 409 → canvas перезагружает
   граф (две правки дерева не перетирают друг друга). Задачи — отдельное SQLite,
   там свой контроль.

## Архитектурное решение (новый ADR-0013)

Записанные ADR — 0001–0011 ([docs/adrs/README.md](docs/adrs/README.md:58)); номер
**0012 уже зарезервирован** budget-задачей #802 (external transaction reads),
поэтому roadmap-канвас берёт **0013**. Файл
`docs/adrs/0013-interactive-canvas-frontend.md`.

- **Решение:** React/Vite-острова разрешены **только для canvas-heavy
  поверхностей** (как `/roadmap`), НЕ как новый default для Assistant-UI.
  Текущий brick остаётся `FastAPI + Jinja2 + Material Web` для обычных страниц
  ([assistant-ui/AGENTS.md](assistant-ui/AGENTS.md)); React-остров — точечное,
  огороженное отклонение, собирается Vite, отдаётся статикой, общается через
  `/api` JSON. Дополняет ADR-0001, не отменяет его.
- **Design-system внутри острова (граница ADR-0008):** обычные кнопки, диалоги,
  формы — через `md-*`; иконки — Material Symbols; цвета — только MD3-токены.
  Внутренние примитивы React Flow (handles, edges, мини-карта, фон-канвас)
  допустимы как canvas-примитивы вне `md-*`.
- После approval — **отдельно**, с подтверждением пользователя (central-plan
  workflow), предложить запись brick в
  `D:\GitHub\_engineering_rules\tech-stack-choices.md`: «React/Vite island for
  canvas-heavy internal UI».

## Build ownership

- Source: `assistant-ui/frontend/` (Vite + React + TS). `package-lock.json`
  **коммитится**; установка — `npm ci` из `assistant-ui/frontend/`.
- Build output: `assistant-ui/app/static/roadmap/` — **в `.gitignore`**, как
  generated artifact (J.3: не коммитим сгенерированное). Собирается
  `npm run build`, по аналогии с тем, как Python-окружение строится `uv sync`.
  Сборка — **на install/update** (в
  [register_services.ps1](tools/register_services.ps1) и README/AGENTS.md),
  **не** на каждый старт сервиса; `verify_frontend.py` гарантированно собирает
  бандл перед smoke. (Альтернатива «коммитить бандл» отклонена по J.3.)
- **Отсутствующий бандл не роняет `/roadmap` в 500:** роут отдаёт понятную
  диагностическую заглушку («canvas не собран — выполните `npm run build`»),
  при этом бэкенд-`/api/*` продолжают работать.
- Node v24 / npm 11 на машине уже есть; прецедента node-toolchain в репо нет —
  это явная новая инфраструктура, фиксируется в ADR-0013.

## Фазы и work items

**Phase 0 — ADR + tasks + toolchain.** Завести PM-MCP work items по subsystem
(K.1/K.2), связать зависимости (K.7). Написать ADR-0013 (см. выше), обновить
список в [docs/adrs/README.md](docs/adrs/README.md).

**Phase 1 — Backend roadmap API (read).** `GET /api/roadmap` собирает из PM-MCP
**один нормализованный граф**: goal tree + связанные work items +
`recommend_next_global` (для будущего overlay) + `file_state`. Чтение связей
task→goal — union `work_item.goal_ids` и `goal.metadata.work_item_ids`, как уже
делает `get_goal_tree`; но `goal.metadata.work_item_ids` — **read-only
legacy/compat вход**: новые writes его не создают (контракт №1), иначе контракт
снова расползётся. Бэкенд отдаёт `date` / `horizon` / `sort` / `lane` / `meta` на
узлах, **не** пиксели. Проверить, что work items реально попадают в payload
текущей обвязки `get_goal_tree` в `server.py`; иначе подмешать `list_work_items`.

**Phase 2 — React-остров (scaffolding).** `assistant-ui/frontend/` (Vite + React
+ TS), сборка в `app/static/roadmap/` (gitignored). Jinja-роут `/roadmap` отдаёт
тонкий шаблон-маунт (`ds/base.html` + контейнер + bundle). Пункт в
`_sidebar.html`. Расширить `scripts/verify_frontend.py` (`--page roadmap`):
`npm ci && npm run build` + Playwright-smoke.

**Phase 3 — Read-only холст.** React Flow из `/api/roadmap`: временная линейка +
LOD, фронтовый layout (X из date/horizon, Y-упаковка), узлы Goal/Task с тегами и
прогрессом, коннекторы parent→child, свернуть/развернуть, pan/zoom, маркер
«сегодня», домен до 2040. Free-drag по X запрещён (`nodesDraggable` ограничен).

**Phase 4a — Редактирование через панель (write, ниже риск).** Боковая панель /
диалог: создать задачу под целью (с правилом `project_path` из контракта №2),
создать промежуточный goal-узел, править поля/`horizon`/`why`/дату. Новые
POST/PATCH `/api/roadmap/*` (CSRF + сессия, как у прочих write-эндпоинтов) с
`expected_mtime_ns`; ошибки валидации PM-MCP (cycle/dangling/archived/
invalid_horizon/conflict) → 409/400 инлайн. Канонический task→goal путь —
контракт №1.

**Phase 4b — Коннектор (write, самый рискованный UX).** draw.io-style
перетягивание handle→handle для перепривязки: goal→goal через
`update_goal(parent_goal_id)`; присоединение/отсоединение задачи через
`set_work_item_goals` (канонический `goal_ids`-путь, контракт №1). Создание
промежуточного узла «между» с
перепривязкой нижестоящих и вышестоящих блоков. Отдельная фаза из-за error
handling и edge cases (циклы ловит сервер → инлайн-откат).

**Phase 5 — Позже (вне MVP).** События Google Calendar на дневных ячейках (MCP
календаря уже подключён); overlay рисков/приоритетов (`recommend_next_global`
score + `why_path`, `get_drift_report`, `get_blockers`); free-drag-to-reschedule;
ретайр `/tree` по migration discipline, когда холст заменит список.

## Критичные файлы

- **assistant-ui (backend):**
  [app/main.py](assistant-ui/app/main.py) — роут `/roadmap`, `GET /api/roadmap`,
  write-эндпоинты (паттерн как у [/tree и /api/goal-tree](assistant-ui/app/main.py:505));
  новый `app/roadmap.py` — view-model (join дерева и work items, нормализация
  `date/horizon/sort/lane`, прокидывание `file_state`), по аналогии с
  [app/dashboard.py](assistant-ui/app/dashboard.py);
  [app/mcp_client.py](assistant-ui/app/mcp_client.py) — общий
  `clients.call_pm(tool, args)` (новых методов не нужно);
  `app/security.py` — CSRF/сессия для write;
  [app/live_updates.py](assistant-ui/app/live_updates.py) — ChangeBus для refetch.
- **assistant-ui (frontend, новое):** `assistant-ui/frontend/` (Vite/React/TS):
  api-client, layout (X из date/horizon, Y-упаковка), `TimeRuler`, `GoalNode`,
  `TaskNode`, connector-логика; `package-lock.json`; сборка в
  `app/static/roadmap/`.
- **assistant-ui (шаблоны):** `app/templates/roadmap.html` (маунт),
  [app/templates/_sidebar.html](assistant-ui/app/templates/_sidebar.html).
- **assistant-ui (verify/install):**
  [scripts/verify_frontend.py](assistant-ui/scripts/verify_frontend.py),
  [tools/register_services.ps1](tools/register_services.ps1) (build-шаг), README.
- **pm-mcp-server:** новый write-tool `set_work_item_goals(project_path,
  task_id, goal_ids)` + round-trip тест (контракт №1) —
  [server.py](pm-mcp-server/server.py),
  [task_store.py](pm-mcp-server/app/task_store.py),
  [registry.py](pm-mcp-server/app/adapters/registry.py),
  [tests](pm-mcp-server/tests). Плюс правка, если `get_goal_tree` не отдаёт work
  items в текущей обвязке.
- **docs:** `docs/adrs/0013-interactive-canvas-frontend.md`,
  [docs/adrs/README.md](docs/adrs/README.md),
  [assistant-ui/AGENTS.md](assistant-ui/AGENTS.md) (frontend invariants + Node/Vite
  + island boundary), [assistant-ui/ARCHITECTURE.md](assistant-ui/ARCHITECTURE.md)
  (`/roadmap` + build pipeline), README.

## Переиспользуемое (не изобретать заново)

- **PM-MCP read:** `get_goal_tree`, `get_goal_path`, `get_goal_decomposition`,
  `recommend_next_global` ([goal_tree.py](pm-mcp-server/app/goal_tree.py)).
- **PM-MCP write:** `create_goal`, `update_goal` (валидирует DAG, принимает
  `expected_mtime_ns`), `create_task`, `update_task` (НЕ для goal-связи — она
  через новый `set_work_item_goals`)
  ([adapters/goals.py](pm-mcp-server/app/adapters/goals.py)).
  `link_work_item_to_goal` из канваса **не** используем (контракт №1).
- **Assistant-UI:** `clients.call_pm`, существующие `/api/goal-*`, security
  (CSRF/сессия), ChangeBus + `/api/dashboard/events`, цвета проектов из
  `app.dashboard`, MD3-токены Design-system для узлов.
- **Модель:** `horizon`-лестница + `due_date` уже в схеме → нормализация позиции.

## Риски и решения

- **React/Vite ≠ default.** Огорожено ADR-0013: только canvas-heavy поверхности;
  island-паттерн (малый радиус) + плацдарм будущего переписывания UI.
- **Новый Node/Vite toolchain без прецедента** (Node v24 / npm 11 есть) →
  build-шаг в `verify_frontend.py` и install-flow, output gitignored (J.3).
- **Две связи task→goal** → один канонический путь `goal_ids` (контракт №1).
- **`project_path` для create_task** → из `goal.metadata.project_path`, иначе
  выбор проекта в UI (контракт №2).
- **Гонки правок дерева** → `expected_mtime_ns` + 409-reload (контракт №3).
- **Геометрический зум vs семантический LOD** → LOD только в линейке по
  `useViewport()`; узлы масштабируются геометрически, «как карта».
- **Раскладка больших деревьев** → фронтовый вертикальный аллокатор; при росте —
  опционально `dagre`/`elkjs`.
- **Узлы без даты** → производная от `horizon`; `life` → правый край.
- **Конфликт «free-canvas vs время»** → позиции data-driven, коннектор меняет
  связи, поля/даты — через панель; free-drag-to-reschedule отложен.

## Проверка (end-to-end)

- **pm-mcp-server:** `uv run ruff check .`; `uv run pytest` — round-trip тест
  `set_work_item_goals`: после записи `get_goal_tree` и `recommend_next_global`
  видят новые `goal_ids` (контракт №1); плюс правка `get_goal_tree`, если нужна.
- **assistant-ui backend:** `uv run ruff check .`; `uv run pytest` — тесты на
  агрегацию `/api/roadmap` (включая `file_state`), нормализацию узлов,
  write-эндпоинты (мок `call_pm`: канонический `goal_ids`-путь, default
  `project_path`, проброс `expected_mtime_ns` и 409).
- **frontend:** `npm ci && npm run build` (Vite) + `tsc` typecheck; затем
  `uv run python scripts/verify_frontend.py --page roadmap` — эфемерный порт
  (никогда не `8000`), Playwright поверх существующего desktop/mobile-контракта
  [verify_frontend.py](assistant-ui/scripts/verify_frontend.py): холст
  монтируется, есть nodes/edges, **zoom меняет transform**, **сворачивание меняет
  число видимых узлов**, **коннектор создаёт/перепривязывает связь**, mobile **не
  перекрывает toolbar**, нет наложения текста.
- **Ручной e2e** на запущенном сервисе (через эфемерный порт): открыть
  `/roadmap`, создать задачу под целью (проверить правило `project_path`),
  создать промежуточный узел и перепривязать коннектором, убедиться в `/tree` и
  `get_goal_tree`, что `parent_goal_id` / `goal_ids` изменились; увидеть прогресс.
