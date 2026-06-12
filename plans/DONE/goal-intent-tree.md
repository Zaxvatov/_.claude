# План: модель целей/задач как дерево смысла (intent tree)

**Статус:** УТВЕРЖДЁН (3 раунда ревью Codex, все правки внесены). MVP — 4 волны (W0–W3) + preflight goal-API bugfix; интейк/посев — в `capture-inbox`.
**PM-MCP:** см. блок «Пул задач» в конце файла.
**Дата:** 2026-06-04
**Автор черновика:** claude
**Затронутые подсистемы:** `pm-mcp-server` (ядро), `assistant-ui` (визуализация)
**Связанные ADR:** ADR-0001 (D-4 kinds), ADR-0004 (idea contract), ADR-0005 (lifecycle). Предлагается новый **ADR-0009** «Intent tree (recursive goals)».
**Связанный фикс (отдельно):** коллизия `metadata.lifecycle` в idea-flow (ideas.py:148 пишет `incubating`, validation.py:430 принимает только `durable/project_state/ephemeral`). Это **прецедент** ошибки «перегрузка поля двумя смыслами» — в этом плане её не повторяем.

---

## 1. Контекст и проблема

Триггер-вопрос пользователя: «Создать AI-memory — это цель или задача?». Вывод: **цель vs задача — не свойство вещи, а её позиция в цепочке смысла.** Один узел является целью относительно того, что под ним, и задачей относительно того, что над ним.

Пользователь хочет, чтобы ассистент **визуально отображал всю цепочку** от самой глобальной цели (ещё не сформулирована) до атомарной задачи, и отвечал на два вопроса по любому узлу:
- **Зачем это?** (обход вверх по родителям → к корню-смыслу)
- **Что делать сейчас?** (обход вниз → к листовой задаче дня)

Дополнительно: верхняя цель **нестабильна и будет уточняться перебором** («тестирую жизнь, отметаю ненужное»). Значит дерево обязано **расти вверх** — позволять вставлять новый корень над существующими узлами позже.

### Текущая модель (по факту в коде)

| Слой | Где | Поддерживает | Не поддерживает |
|---|---|---|---|
| Goals | `pm-mcp-server` `config/goals.yaml`, адаптер `app/adapters/goals.py` | `id, title, status, summary, priority, due_date, strategic_weight, metrics, progress_history, progress, metadata`; `horizon` лежит в `metadata` | **`parent_goal_id` — НЕТ** (цели плоские) |
| Tasks | `pm-mcp-server` SQLite (`WorkItem`) | `goal_ids: list` (M:N), `dependencies` (DAG), `initiative`/`epic` (строки-ярлыки) | строгий `parent_task_id` (есть только через dependencies) |
| Scoring | `recommend_next_global` | компонент `strategic_alignment` по связанным активным целям | обход по цепочке родителей целей |

**Ключевой факт для реализации:** `_parse_goal` (goals.py:228-231) складывает любое нестандартное поле в `metadata`. Значит `parent_goal_id`, `why`, управляемый `horizon` можно ввести **без миграции схемы** — через существующий свободный `metadata`.

### Что отклоняем

Модель ChatGPT из 8 фиксированных уровней (`Life Vision → Goal → SubGoal → Initiative → Project → Epic → Task → Subtask`). Причины:
1. Релятивность уровня (тот же триггер-вопрос) → фиксированные типы дают паралич «это SubGoal или Initiative?».
2. Вершина не сформулирована → фиксированные уровни не растут вверх без болезненной пере-типизации.
3. Коллизия терминов: «project» в системе уже занят (`project_path` = репозиторий). Нельзя отождествлять с intent-узлом.

---

## 2. Цель плана

Минимальная доработка, дающая **видимое дерево смысла с двумя обходами**, на существующем стеке, **без новых контейнеров и без слома схемы целей**. Восемь «уровней» схлопываются в: **2 рекурсивных слоя + 1 атрибут высоты (`horizon`)**.

---

## 3. Целевая модель

```
   INTENT-СПИНА (зачем / смысл)                 TASK-СЛОЙ (что делать)
   = расширенные goals, рекурсия                = WorkItem, рекурсия/DAG
   через metadata.parent_goal_id                через dependencies (как есть)

   ┌──────────────────────────┐
   │ корень (гипотеза)        │  horizon: life        ← переставляемый,
   │ «Понять, чего я хочу»    │                          возможно несколько корней (forest)
   └───────────┬──────────────┘
   ┌───────────▼──────────────┐
   │ «Быть эффективнее»       │  horizon: multi_year
   └───────────┬──────────────┘
   ┌───────────▼──────────────┐
   │ «Создать AI-Assistant»   │  horizon: year   ←── link (metadata.project_path):
   └───────────┬──────────────┘                       D:\GitHub\AI-Assistant
   ┌───────────▼──────────────┐
   │ «Создать AI-memory»      │  horizon: quarter
   └───────────┬──────────────┘
               │ goal_ids (M:N, как есть)      ┌───────────────────────────┐
               └──────────────────────────────►│ Task: реализовать          │
                                               │ memory_search   [status]   │
                                               │   └─ dependency → subtask   │
                                               └───────────────────────────┘
```

**Тест принадлежности узла:** узел — это **Task**, если есть конкретное «сделано/не сделано» и его кто-то делает; **Intent**, если он достигается только через детей и существует, чтобы объяснять «зачем».

Принципы:
- **Высота = атрибут `horizon`, не тип.** Глубина = цепочка `parent_goal_id`, не ограничена.
- **Intent-узел и «цель» — один тип** (расширяем `goals`, не вводим новую сущность).
- **`why` ≠ `summary`.** `summary` — описание, `why` — обоснование (вверх по цепочке это и есть ответ «зачем»).
- **`project_path` ≠ intent-узел.** Связь через `metadata.project_path` на цели, без отождествления.
- **Источник правды дерева — `pm-mcp-server`** (операционное состояние, по ADR-0001; не ai-memory).
- **Раскрутка двунаправленная.** Модель направления-агностична: снизу вверх (от задачи к смыслу, induction) и сверху вниз (от смысла к задачам, deduction) — это одни и те же операции над деревом (`parent_goal_id` вверх / создание детей вниз). Различается только workflow агента, не data-model. Подробно — §10.
- **Это «дерево смысла + операционный слой работ», не «дерево задач»** (формулировка Codex). Intent tree отвечает на «зачем» (`Goal → parent_goal_id → root`); work items — на «что делать» (статусы, зависимости, блокеры, readiness).
- **`dependencies` — это DAG блокировок, не смысловая декомпозиция.** Декомпозиция смысла идёт ТОЛЬКО через intent-дерево; task-dependency не трактуется как parent/child задачи.
- **Задача крепится к ближайшему intent-узлу через `goal_ids`;** смысл выше наследуется по цепочке родителей.

---

## 4. Переиспользование стека (сверка с tech-stack-choices.md)

Новых кирпичиков **не требуется**. Применяем:

- **Brick #3 (SQLite/FTS5/WAL + JSON1 partial index)** — релевантен, если goals мигрируют в SQLite (см. Q1). Пока N целей мал → дерево строится in-memory из плоского YAML-списка, индекс не нужен. Путь миграции (если понадобится индексируемый query по `parent_goal_id`): `CREATE INDEX ... ON goals(json_extract(metadata,'$.parent_goal_id')) WHERE archived_at IS NULL`.
- **Brick #5 (FastMCP + runtime_contract)** — `EXPECTED_TOOLS` авто-выводится из FastMCP registry (`pm-mcp-server/app/runtime_contract.py:13`), ручной правки НЕ требует: новый `@mcp.tool()` появляется там сам. Обязательно обновить **статический** fallback `PM_MCP_TOOL_NAMES` в `assistant-ui/app/mcp_client.py:24` (fail-open список). *(Поправка по ревью Codex п.6.)*
- **Brick #6 (FastAPI+Jinja2+Material Web, no SPA)** — tree view только server-rendered MD3, `<md-*>`, токены `--md-sys-*`. Без React/графовых JS-движков.
- **YAML через `safe_dump`** *(не brick #13 — тот про Markdown frontmatter; Codex р2)*: `goals.yaml` пишется `yaml.safe_dump(allow_unicode=True, sort_keys=False)` — та же safe-YAML дисциплина. Сохраняем.
- **Чистый сервис + MCP-обёртка** *(не brick #14 hybrid — UI ходит в pm-mcp только через MCP loopback, не direct import; Codex р2)*: tree-логика в `app/goal_tree.py`, `server.py` оборачивает в MCP tools, UI зовёт через MCP-клиент.
- **Brick #16 (frontend org)** — tree-рендер JS в `assistant-ui/static/` (app-local feature), inline ≤ ~50 строк.

**Кандидат в новый brick (предложить только после согласования):** «Recursive tree over flat store: in-memory build + cycle/orphan guard» — если паттерн окажется переиспользуемым.

---

## 5. Этапы (волны) — сужено до 4 (по ревью Codex)

Зависимости: **W0 → W1 → W2 → W3**. По правилу K.4 — отдельный work item на каждый `project_path`. Посев дерева, Todoist ingest, Universal Capture, proposals для целей и Gateway `propose_goal` вынесены в отдельный план **`capture-inbox`** (см. §9–§10).

**W0 — ADR-0009 + точный контракт payload (до кода).**
- Зафиксировать data-contract в ADR-0009.
- Явные read-tools вместо размытых «хелперов» (Codex п.1): `get_goal_tree(root_goal_id=None, project_path_filter=None, max_depth=None, include_archived=False)`, `get_goal_path(goal_id)` (вверх к корню), `get_goal_decomposition(goal_id, include_work_items=true)` (вниз + связанные задачи). Решить: 3 явных tool'а vs один с режимами (рекоменд.: 3 явных). *(Codex р2: `project` → `project_path_filter` — слово `project` перегружено.)*
- **Форма payload узла (Codex р2/р3 — фиксируем в ADR, не оставляем свободу реализации):** `node` (сама цель), `children`, `path` (цепочка к корню), `linked_work_items` (источник: union задач с `goal_id ∈ work_item.goal_ids` + обратных ссылок `goal.metadata.work_item_ids`; НЕ полагаться на несуществующий `list_work_items(goal=...)` — Codex р3), `diagnostics` (`dangling_parent`, `cycle`, `duplicate_goal_id`, `archived_parent`; `multiple_roots` — **info**, не ошибка — Codex р3), `normalized_horizon` + `raw_horizon`.
- Пин `horizon`-enum + правило нормализации (Codex п.2): drift `goals.yaml.example` `quarterly` ↔ план `quarter`.
- Формула chain-aware scoring (capped/decayed) фиксируется здесь, до кода (Codex п.7; детали — W2).

**W1 — `pm-mcp-server`: ядро дерева в чистом модуле (Codex п.3).**
- Новый модуль `pm-mcp-server/app/goal_tree.py`: build forest, validate parent, detect cycle, find path, collect linked tasks. `server.py` только оборачивает в MCP tools.
- Конвенция `metadata.parent_goal_id` + `metadata.why` + `metadata.horizon` (нормализация/валидация на write-path).
- **Двойная валидация (Codex п.4 + р3):** write-path жёсткий — `create_goal`/`update_goal` запрещают цикл, отсутствующий parent (`dangling_parent`) и **ссылку active-цели на archived parent**; read-path терпимый — `get_goal_tree` не падает на битом YAML, а возвращает `diagnostics`: `dangling_parent`, `cycle`, `duplicate_goal_id`, `archived_parent`; `multiple_roots` — info-диагностика (forest допустим), не ошибка.
- Обновить **статический** `PM_MCP_TOOL_NAMES` fallback в `assistant-ui/app/mcp_client.py:24` (EXPECTED_TOOLS авто-выводится — Codex п.6). Docs: `docs/MCP_API.md`, `ARCHITECTURE.md`, `AGENTS.md`.
- Тесты: cycle, orphan/dangling parent, forest (несколько корней), depth, archived-ветка, diagnostics.

**W2 — `pm-mcp-server`: chain-aware scoring по формуле (Codex п.7).**
- `strategic_alignment` учитывает цепочку родителей по **capped/decayed** модели: прямая цель — полный вес, родители убывают по depth, итог capped текущим лимитом компонента (root НЕ доминирует всё).
- **Затронуть ОБЕ scoring-функции (Codex р3):** `_project_policy_score` (`server.py:3130`) и `_global_work_item_score` (`server.py:3420`) — иначе `recommend_next_global` останется без `why_path`.
- В ответ рекомендации добавить `why_path` для предложенной задачи.

**W3 — `assistant-ui`: read-only визуализация + PREFLIGHT-фиксы (Codex п.5).**
- **Preflight (до дерева) — починить подтверждённые UI↔PM-MCP goal-несовпадения:**
  - `create_goal`: UI шлёт плоский payload (`main.py:502`, поля `weight`/`horizon`), tool ждёт `create_goal(goal={...})` со схемой `Goal` (`strategic_weight`, `horizon` в metadata) — `server.py:3855`.
  - `update_goal`/`archive_goal`: UI шлёт `id` (`main.py:507/513`), tool ждёт `goal_id`/`fields` — `server.py:3865/3879`.
  - `/api/goal-progress` зовёт `goal_progress_report` (`main.py:448`), которого в PM-MCP **нет** (живёт только в fake-тесте) — заменить на `list_goals` / существующий snapshot.
  - **`list_goals` реальная форма (Codex р2):** `{goals: [{domain, source, goal: {...}}], goal_count, file_state}` — каждый элемент обёрнут под `.goal` (`registry.py:98-107` `_goal_entry`, `server.py:3759`); UI и **fake-тест-клиенты** должны читать реальный контракт, а не упрощённую плоскую форму.
- Затем дерево-вью: страница (`/goals` расширить или `/tree`), вверх = «зачем» (`get_goal_path`), вниз = «что сейчас» (`get_goal_decomposition` + `recommend_next_global`).
- Server-rendered MD3; fail-open fallback; JS app-local; проверка `frontend-verification` (desktop+mobile).

---

## 6. Открытые вопросы для обсуждения с Codex (главное)

- **Q1. Хранилище целей: YAML или миграция в SQLite?** Рекомендация: YAML сейчас (малый N, 0 миграций, brick #3 как путь роста). Trade-off: YAML — нет индексов и concurrent-write, ручная целостность дерева.
- **Q2. `parent_goal_id`/`why`/`horizon` — `metadata.*` или first-class поля схемы `Goal`?** Рекомендация: `metadata` сначала (0 миграций), промоут в поля при доказанной нагрузке. Trade-off: first-class даёт валидацию и явный контракт; metadata гибче, но без схемной проверки.
- **Q3. Словарь `horizon` → РЕШЕНО в W0:** зафиксировать enum + нормализация; устранить drift `quarterly` (в `goals.yaml.example`) ↔ `quarter`. Предложение: `life, multi_year, year, quarter, month`.
- **Q4. Intent-узел = тот же тип, что `goal`?** Рекомендация: да, расширяем `goals`, не вводим новую сущность.
- **Q5. `initiative`/`epic` (строки-ярлыки):** оставить как группировку задач или промоутить в intent-узлы? Рекомендация: оставить как есть сейчас, пересмотреть позже.
- **Q6. Визуализация дерева в no-SPA MD3:** вложенные `<md-list>` / `<details>` / лёгкий SVG? Нужен выбор подхода без графового JS-фреймворка.
- **Q7. Валидация cycle/orphan → РЕШЕНО (W1, подтв. Codex):** двойная — write жёсткий, read терпимый с `diagnostics`.
- **Q8. «Задача дня» → УТОЧНЕНО (W2):** chain-aware через capped/decayed `strategic_alignment`; subtree-scoped вариант — опционально позже.
- **Q9–Q11 (Todoist ретайр, Gateway `propose_goal`, дом `personal`, размещение intake-кусков) → ПЕРЕНЕСЕНЫ** в план `capture-inbox` (Codex п.8/п.9: интейк вынесен из intent-tree MVP).
- **Q12. Контракт tree API (Codex п.1):** 3 явных read-tool'а (`get_goal_tree`/`get_goal_path`/`get_goal_decomposition`) vs один с режимами. Рекоменд.: 3 явных. Решить в W0.

---

## 7. Риски и анти-цели

- **Переусложнение в 8 уровней / графовую БД.** Отклонено: 2 слоя + `horizon`. Граф — только read-model позже, если «связанное с X» станет реальной болью (иначе 4-й source of truth против ADR-0001).
- **Гонки YAML vs целостность дерева.** Митигировать: write-валидация (cycle/orphan) + существующий `expected_mtime_ns` conflict guard в goals.py.
- **Повтор ошибки `lifecycle`.** Не перегружать существующие поля вторым смыслом; `horizon` — отдельное управляемое поле, не переиспользование `priority`/`strategic_weight`.
- **Conflation `project_path` ↔ intent-узел.** Только связь через metadata.
- **Сложность визуализации в no-SPA.** Держать просто (вложенные списки), не строить редактор графа.
- **Scope creep.** Universal Capture (из прошлого обсуждения) — **отдельный план**; здесь только модель/дерево/визуализация. Seam: capture позже «роняет» новые узлы на это дерево и проставляет `parent_goal_id`/`goal_ids`.
- **Todoist** — external seam (детали в `capture-inbox`): ничто в MVP от него не зависит. *(Codex р2: убран из core-рисков.)*

---

## 8. Критерии приёмки

- [ ] Модель согласована с Codex; ADR-0009 зафиксировал data-contract.
- [ ] `parent_goal_id` работает без миграции схемы (или миграция задокументирована, если промоут в поля по Q2).
- [ ] `get_goal_tree` корректно отдаёт forest, обходы вверх/вниз; устойчив к циклам и orphan; покрыт тестами.
- [ ] `recommend_next_global` отдаёт `why_path` для рекомендованной задачи (chain-aware capped/decayed scoring).
- [ ] Preflight: UI↔PM-MCP goal-контракт починен (`create_goal(goal=...)`, `update_goal(goal_id, fields)`, `archive_goal(goal_id)`, `/api/goal-progress`).
- [ ] UI рисует спину; клик по узлу даёт «зачем» (вверх) и «что сейчас» (вниз + задача дня); проверено `frontend-verification` (desktop + mobile).
- [ ] `uv run ruff check .` + `uv run pytest` зелёные в каждой затронутой подсистеме.
- [ ] Обновлены `ARCHITECTURE.md` / `docs/MCP_API.md` / `AGENTS.md` затронутых подсистем.

---

## 9. Вне scope этого плана

- Universal Capture / Capture decision, intake-куча/Inbox, agent-skill двунаправленной раскрутки, **посев дерева (бывш. W4)**, `todoist→SQLite ingest`, дом `personal`, proposals для целей, Gateway `propose_goal` — всё в отдельном плане **`capture-inbox`** (по ревью Codex п.8). Здесь — только интерфейсная совместимость (§10).
- Новые домены `contact` / `learning-item`.
- Relationship Graph как первичное хранилище.
- Фикс idea-`lifecycle`-коллизии (отдельная задача; пререкизит только если capture начнёт использовать idea-путь).

---

## 10. Интерфейсная совместимость с будущим интейком (детали — в плане `capture-inbox`)

По ревью Codex (п.8) интейк вынесен из intent-tree MVP. Здесь — только **контракт стыковки**, чтобы MVP не пришлось переделывать:

- **Дерево direction-agnostic.** Любой будущий источник (chat-агент, Todoist seed, ручной capture) создаёт goals/tasks через те же `create_goal`/`create_task` с `parent_goal_id`/`goal_ids`. Спец-API для интейка в этом плане нет.
- **Todoist seed допустим, но НЕ волна MVP.** Read-only адаптер уже есть (`read_todoist_work_items`, токен `PM_MCP_TODOIST_TOKEN`) и включается независимо как zero-regret эксперимент. Канон — PM-MCP; ничто не зависит от Todoist (pluggable-адаптер, ретайр тривиален).
- **Всё остальное → план `capture-inbox`:** 5-шаговый посев (bottom-up induction ↔ top-down deduction), двунаправленный skill раскрутки, модерация целей/задач, `todoist→SQLite ingest`, дом `personal` (real `project_path` vs домен vs adapter — решить там, Codex п.9), Gateway `propose_goal`.

---

## Пул задач (PM-MCP) — создан, assignee `codex`

Граф: **#790 → #791 → #792 → #793**; **#793** также зависит от **#789**; **#789** независим.

| ID | Подсистема | Волна | Статус | Зависит от |
|---|---|---|---|---|
| **#789** | assistant-ui | Preflight goal-API bugfix (UI-only) | К выполнению | — |
| **#790** | pm-mcp-server | W0 — ADR-0009 + контракт | К выполнению | — |
| **#791** | pm-mcp-server | W1 — `app/goal_tree.py` core | Бэклог | #790 |
| **#792** | pm-mcp-server | W2 — chain-aware scoring | Бэклог | #791 |
| **#793** | assistant-ui | W3 — read-only визуализация | Бэклог | #792, #789 |

Старт без блокеров (можно параллельно): **#789** (preflight) и **#790** (ADR/контракт).

## Процедурные заметки (по central-plan-workflow)

- Tech-stack bricks/hooks: Codex подтвердил — новых не нужно. Кандидат «recursive tree over flat store» предлагать только после реализации и повторного применения (не сейчас).
- ✅ Создан пул задач #789–#793 (assignee `codex`, зависимости проставлены) — см. «Пул задач (PM-MCP)».
- ✅ Итог согласования в AI-memory `#1561`→`#1562` (portfolio, durable; обновлён после создания пула).
