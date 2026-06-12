# Roadmap: реальная ось времени, даты у задач/целей (drag-reschedule) и подключение Google-календаря

**PM-MCP work item:** #843 (project `D:\GitHub\AI-Assistant`, в работе, goal `portfolio-control-plane`, заголовок синхронизирован с моделью). Статус: ЧЕРНОВИК v2 (учтён read-only review Codex), согласуем.

## Context

Страница `/roadmap` (React Flow island, ADR-0013) не выполняет обещание «временной холст»:

- **Нет движка оси времени.** `frontend/src/main.tsx::layoutGraph` жмёт весь отрезок 2026–2040 в 2200px; зум только меняет подпись LOD. `TimeRuler` рисует **только годы** как равномерные flex-элементы, не привязанные к X узлов, и **не двигается** вместе с холстом (опрос `getViewport()` раз в 300мс).
- **Нет кнопок День/Неделя/Месяц/Год** — только зум `+/−`.
- **У данных нет дат.** Реальные задачи PM-MCP отдаются (`list_work_items domain=project_tasks status=open`), но у всех `due_date=null`; цели в `goals.yaml` дат не имеют, только `horizon`. `roadmap.py::_node_date` подменяет отсутствующую дату на `today + HORIZON_OFFSETS[horizon]` → всё сваливается на «сегодня» в узкий столбец у левого края, реальные задачи теряются, а карточки целей (life-constitution) воспринимаются как «левые задачи».
- **Связи цель→задача обходят петлёй** (source=Right, target=Left; задача левее цели → длинная дуга).
- **Google-календарь не подключён, UI подключения нет.** `list_calendar_events` → `items:[]`, `sources:{}`. Подключение только через CLI `python -m app.calendar_auth` (нужен OAuth Desktop client Google Cloud); сервис pm-mcp в session 0 сам браузер открыть не может.

**Решения пользователя (2026-06-07):**
1. Календарь: подключение (OAuth) — в настройках, **разовый** вход командой в своей сессии. А выбор/видимость календарей — **Google-стиль боковая панель** на /roadmap: группы «Мои календари»/«Другие календари», цветной чекбокс у каждого → показывает/прячет его события (см. скрин Google Calendar).
2. **Модель дат у задач и целей — три datetime-поля:** `created` (дата+время создания), `planned` (плановая дата+время — драйвер раскладки на холсте), `actual` (фактическая дата+время). Заполняются при создании (агентом или вручную), **редактируются в окне задачи**. Сохраняется **история смены статуса** (как сейчас — таблица `task_status_history`). Без `planned` → рельс «Без даты».
3. **Drag-reschedule:** `planned` меняется **перетаскиванием карточки на другую дату/время**, как в Google Calendar.
4. Порядок: без предпочтений → делаем A → B.

**Итог:** честная ось времени (datetime) с переключателем масштаба; линейка совпадает с узлами и едет с холстом; задачи/цели стоят под своей **плановой** датой; даты ставятся/меняются вручную, перетаскиванием и агентом; видны created/planned/actual + история статусов; кратчайшие связи; Google-стиль панель календарей.

## Подтверждённые факты кода (на них опирается план)
- Схема `tasks` (SCHEMA_VERSION=4, `tasks_db.py:16`): есть `due_date`(date-only)/`created_at`/`updated_at`/`closed_at`; **нет** `planned_at`/`actual_at`. `migrate()` (`tasks_db.py:464`) — НЕ простой ALTER: `CREATE TABLE IF NOT EXISTS` (новой колонке существующую таблицу не добавит), legacy-ветки по `_table_columns`, запись `schema_migrations`, финальный `PRAGMA foreign_key_check`. → для 4→5 нужен явный `_table_columns(...,'tasks')`-чек + `ALTER TABLE ... ADD COLUMN` только если колонок нет, идемпотентный повторный прогон.
- **`task_status_history` уже есть** (`tasks_db.py:48`) → история статусов как сейчас, не трогаем.
- `update_task` пропускает поля через whitelist `_normalize_update_fields` (task_store.py:173) — добавить `planned_at`/`actual_at`. `create_task`/`create_task_record` — добавить параметры. **У task-update НЕТ optimistic-guard** (у целей есть `expected_mtime_ns`) → добавляем **в PM-MCP** (conditional `UPDATE … WHERE updated_at=?`, rowcount 0 → `task_conflict`), не read-compare-write в Assistant-UI (TOCTOU). См. A1/A4.
- **Сквозной контракт даты** должен пройти через ВСЕ публичные DTO: `app/schemas/contracts.py` (WorkItem/Goal, `due_date` :27/:51), `_task_to_dict`, `list_work_items`, goals adapter (`goals.py:34` +isoformat :243, `roadmap.py::goal_fields_payload`/`_walk_goal`), Assistant-UI `RoadmapTaskCreateRequest` (main.py:312) + `task_create_payload` (roadmap.py:123) + roadmap payload.
- `due_date` (дедлайн, date-only) остаётся — завязан на scoring (`_due_date_score` server.py:491); **НЕ** драйвер раскладки. Драйвер = `planned_at`.
- **Календарь уже умеет цвет/выбор:** `calendar_sources` имеет `selected` и `color` (calendar_store.py:26-27), sync берёт `backgroundColor`/`accessRole`/`selected` (`_source_from_calendar_list` calendar_sync.py:327). → не «добавить цвет», а: read-tool `list_calendar_sources` + API + UI. **`selected` (sync inclusion) ≠ видимость на холсте** → новое поле `visible_on_roadmap` (миграция calendar schema). Static fallback tool-list `assistant-ui/app/mcp_client.py:75` — добавить новый tool.
- Эталон настроек — Obsidian (`settings.html`+`static/settings.js`+`/api/settings/obsidian`). Без новых npm-зависимостей (тики/подписи `Intl.DateTimeFormat`).

**Модель «Даты» (контракт):** `created_at` (есть, read-only) · `planned_at` (новое) · `actual_at` (новое). `planned_at`/`actual_at` = **ISO 8601 в UTC** (`…+00:00`); UI конвертирует local↔UTC (`datetime-local` без tz), PM-MCP **валидирует** (reject invalid/oversized; `null`/`""` = очистка). `due_date` = дедлайн, остаётся date-only. Для задач — колонки SQLite; **для целей — top-level поля goal** (как `due_date`; goals.py `GOAL_FIELDS`), не metadata. Изменение публичной модели WorkItem/Goal → **обязателен mini-ADR / amendment к ADR-0009/0013** (см. Post-approval).

---

## Phase A — холст: ось времени + даты + drag (не требует Google)

### A0. Синхронизация трекинга + ADR-решение
- Заголовок/описание #843 синхронизированы с planned/actual-моделью (сделано).
- **Mini-ADR (обязателен)** или amendment к ADR-0009 (intent-tree) / ADR-0013 (canvas): `planned_at`/`actual_at` как публичные поля WorkItem/Goal; datetime-канон UTC; `due_date` остаётся дедлайном.

### A1. PM-MCP: schema 4→5 + сквозной контракт
- `app/tasks_db.py`: bump `SCHEMA_VERSION=5`; колонки `planned_at`/`actual_at TEXT` в `TASK_TABLES_SCHEMA` (новые БД) **и** явная ветка в `migrate()`: если в `_table_columns('tasks')` их нет → `ALTER TABLE tasks ADD COLUMN ...`; идемпотентно; `foreign_key_check` уже есть.
- `app/task_store.py`: `TaskRecord` += поля; `upsert_task_record` пишет; `_normalize_update_fields` += whitelist; `create_task_record` += kwargs; `update_task_record` += optional `expected_updated_at` → conditional `UPDATE … WHERE updated_at=?`, rowcount 0 → `task_conflict`.
- `server.py`: `create_task` += параметры; `update_task` пробрасывает `expected_updated_at`; `_task_to_dict` отдаёт оба; **datetime-валидация** (UTC ISO; reject invalid/oversized; `null`/`""`=clear).
- `app/schemas/contracts.py`: WorkItem/Goal DTO += `planned_at`/`actual_at`; `list_work_items` отдаёт.
- Цели: `app/adapters/goals.py` `GOAL_FIELDS` (top-level, как `due_date`) += `planned_at`/`actual_at` (+isoformat); goals.yaml — top-level ключи; create/update_goal проводят.

### A2. Roadmap read-model `assistant-ui/app/roadmap.py`
- `_node_date`: убрать `today + HORIZON_OFFSETS`. Дата = `planned_at`(задачи/цели)/`start`(события); нет → `undated:true`. `horizon` — только визуальный признак.
- Узлы несут UTC-datetime `planned_at` + в `meta`: `created_at`/`actual_at`/`due_date`/`project_path`.
- Life-цели (`horizon: life`) → `band:"life"`.

### A3. Пассивная ось времени (frontend, read-only) `frontend/src/main.tsx`
- Координата `x(ts)=(ts−start)/msPerPx(view)`; `view∈{day,week,month,year}`; центрирование на «сейчас» (не `fitView`).
- Кнопки **День/Неделя/Месяц/Год** (MD-сегменты).
- `TimeRuler` на `useViewport()`: тики из той же `x(ts)`, едут с холстом; **генерация тиков только по видимому диапазону + буфер** (не рендерить весь 2026–2040 на day-view); подписи `Intl.DateTimeFormat('ru-RU')`, дата+время на мелких масштабах.
- Раскладка: `y` по дорожке, `x` по `planned_at`; `undated`→рельс «Без даты»; life-полоса. LOD (year/month компактно). Floating-рёбра (кратчайший путь).
- a11y/perf: панель/sidebar **вне canvas**; клавиатурный доступ к масштабу/инспектору; компактные узлы на крупных view.

### A4. Редактор + write-safety PATCH `assistant-ui/app/main.py` + `roadmap.py` + `main.tsx`
- Панель «как в Google Calendar»: `planned`/`actual` (`datetime-local`, local↔UTC) + `created` (read-only) + **история статусов** (`get_work_item_history`); Сохранить → PATCH.
- Новый `PATCH /api/roadmap/tasks/{task_id:path}`: **field-allowlist только `planned_at`/`actual_at`**; CSRF-header; `project_path`+canonical id через `get_work_item`; **optimistic guard в PM-MCP** — PATCH шлёт `expected_updated_at` в `update_task` (conditional UPDATE, rowcount 0 → 409 `task_conflict`+reload), НЕ read-compare-write в UI (TOCTOU). Цели: поля в `RoadmapGoalRequest` (guard `expected_mtime_ns` уже есть).
- Создание: `RoadmapTaskCreateRequest`/`task_create_payload` += `planned_at`; оптимистично с откатом.

### A5. Drag-reschedule `main.tsx`
- Узлы задач/целей таскаются по горизонтали (события/life — нет); `onNodeDragStop`: `x→planned_at` со снапом к единице view (День — к времени), live-подсказка + подсветка тика; из «Без даты» на ось = задать, обратно = очистить.
- Тот же PATCH из A4 (allowlist + guard + **rollback при ошибке**).

---

## Phase B — Google-календарь: источники + панель календарей

Зависит от внешнего шага пользователя (OAuth Desktop client + разовый consent). Холст Phase A рисует события, как только появятся. **Цвет/`selected` уже синкаются** — добавляем доступ и UI.

### B1. PM-MCP: read-tool источников + поле видимости
- `list_calendar_sources` → `id`, `summary`, `color`, `access_role`, `trust_tier`, `selected`, `visible_on_roadmap`. Обновить static fallback tool-list `assistant-ui/app/mcp_client.py:75` (degraded discovery). Тесты discovery.
- **Развести два смысла:** `selected` = включён в sync (calendar.db); **новое `visible_on_roadmap`** (миграция calendar schema 1→2) = показ на холсте, по нему фильтруют `list_calendar_events`/roadmap. Google-чекбокс переключает `visible_on_roadmap` мгновенно, без resync. `selected` синком не затирается (ставится `True` только для впервые обнаруженных eligible).

### B2. Подключение (Settings → «Календарь»), fail-closed на запись
- `GET /api/settings/calendar` — degraded/fail-open: только `configured`/scopes/**fingerprint** (НЕ refresh token/secret).
- `POST` (sync trigger, запись выбора) — **fail-closed**, CSRF.
- Таб «Календарь» (зеркало Obsidian): статус + команда `cd pm-mcp-server; uv run python -m app.calendar_auth --client-secrets <path>` (subsystem env, не bare `python`) + путь к токену.

### B3. Панель календарей на /roadmap (Google-стиль, см. скрин)
- Левая панель **вне canvas**: «Мои календари» (owner/writer) / «Другие календари» (reader/freeBusy/public); цветной чекбокс (touch ≥44px, доступен с клавиатуры) → `visible_on_roadmap` → фильтр событий на холсте. Цвет события = цвет календаря.
- Видимость персистится в `visible_on_roadmap` (B1); `selected` (sync inclusion) не трогаем — переключение мгновенно, без resync.

### B4. Docs
- README how-to: OAuth Desktop client → `PM_MCP_GOOGLE_CALENDAR_CLIENT_SECRETS_PATH` → consent → отметить календари. Ссылка на ADR-0016.

---

## Файлы (ключевые)
- PM-MCP: `app/tasks_db.py` (миграция 4→5), `app/task_store.py`, `app/schemas/contracts.py`, `app/adapters/goals.py`, `server.py`; Phase B — `server.py` (`list_calendar_sources`), `app/calendar_store.py`/`calendar_sync.py` (+ тесты).
- Backend холста/настроек: `assistant-ui/app/roadmap.py`, `assistant-ui/app/main.py`, `assistant-ui/app/mcp_client.py` (fallback tool-list).
- Фронт: `assistant-ui/frontend/src/main.tsx`, `assistant-ui/frontend/src/styles.css`.
- Настройки/ADR: `assistant-ui/app/templates/settings.html`, `assistant-ui/app/static/settings.js`, `docs/adrs/` (mini-ADR/amendment).

## Verification
- PM-MCP unit: `cd pm-mcp-server; uv run pytest -k "task_store or migrat or create_task or goals or calendar"`. Кейсы: миграция старой БД (tasks 4→5, calendar 1→2) **идемпотентна** (колонки добавлены, данные целы, `foreign_key_check` чист); create/update planned/actual (UTC) читаются; **invalid datetime reject**, `null`=clear; **concurrent `update_task` с устаревшим `expected_updated_at` → `task_conflict`** (rowcount 0); **`due_date` scoring не изменился**; `task_status_history` пишется; `list_calendar_sources` отдаёт color/selected/`visible_on_roadmap`; **sync не затирает user `selected`**; `visible_on_roadmap=false` прячет события; private события без `description/location/html_link`.
- Assistant-UI: `cd assistant-ui; uv run pytest`. Кейсы: узлы по `planned_at`, `planned_at=null`→«Без даты»; PATCH — **CSRF 403 без header**, **unknown fields rejected** (только planned/actual), `expected_updated_at` mismatch→409; calendar `GET` fail-open / `POST` fail-closed; токен/secret не в ответе.
- Фронт: `cd assistant-ui/frontend; npm ci`(если deps)`&& npm run build`; `cd assistant-ui; uv run python scripts/verify_frontend.py --page roadmap` (НЕ 8000). Скрины desktop/mobile: линейка день/неделя/месяц/год совпадает с узлами и едет с холстом; кнопки масштаба; узлы под planned; связи без петель; drag меняет дату с rollback при ошибке; панель created/planned/actual + история статусов; sidebar календарей вне canvas, чекбоксы прячут события. **Perf:** day-view не рендерит весь 2026–2040 (тики по viewport+буфер). **a11y:** клавиатура до масштаба/календарей/инспектора.
- Календарь (после подключения): таб `configured`; панель «Мои/Другие» с цветами; чекбоксы прячут/показывают события; после синка события на холсте.
- **Lint (gate по AGENTS):** `cd pm-mcp-server; uv run ruff check .` и `cd assistant-ui; uv run ruff check .` — чисто (для затронутых подсистем).

## Post-approval (вне plan mode)
- Разбить #843 на под-задачи A0–A5 / B1–B4 с зависимостями; id вписать сюда.
- **ADR обязателен** (не «проверить»): mini-ADR или amendment к ADR-0009/0013 — `planned_at`/`actual_at` меняют публичную модель WorkItem/Goal + datetime-канон UTC.
- Видимость календарей = `selected` в `calendar_sources` (единый source-of-truth), не отдельный файл.
- Tech-stack — только существующие bricks, без новых: SQLite+WAL (#3), React island ADR-0013, MD3/Design-system (#6/#10), Google Calendar brick (google-api-python-client, read-only), keyring/restricted-file secrets. Свериться с `_engineering_rules/tech-stack-choices.md` при старте.
