# PM-MCP bilingual enums — RU+EN contract (canonical v6)

## Changelog

- **v1** (2026-05-27): первая версия как `pm-mcp-bilingual-enums.md` в central plans.
- **v2** (2026-05-31, Codex review #1): план устарел после PM-MCP #704; 6 правок (extra="forbid", VALID_TASK_STATUSES не расширять, filter normalize, complete event, goals out, allowed_status_inputs).
- **v3** (2026-05-31, Codex review #2): переформулирован как self-contained canonical (не «apply Edits to other file»); Phase 1+2 объединены; добавлены acceptance для `move_task`, case-sensitivity priority, monkeypatch publish_event. Старый `pm-mcp-bilingual-enums.md` помечен как устаревший source material — удаляется в housekeeping.
- **v4** (2026-05-31, Codex review #3): 6 уточнений — (a) `bulk_update_tasks` не через `_change_task_status` (shared normalize helper только), (b) explicit `Literal` вместо dynamic tuple unpack, (c) `list_work_items` multi-domain status filter conditional (project_tasks only), (d) `pm-mcp-server/AGENTS.md` на английском (правило репо), (e) housekeeping без противоречия «delete+copy» (replace + harness one-line), (f) «Verification этой сессии» переименован в «Acceptance после housekeeping».
- **v5** (2026-05-31, Codex review #4): 5 финальных правок — (a) заголовок v3 → v4, (b) Risks mitigation для bulk согласована с §Phase 1 (helper напрямую, не `_change_task_status`), (c) registry filter — conditional нормализация только для `project_tasks`, (d) housekeeping шаг 1 — rename старого + новый файл (НЕ Write replace, согласовано с правилом «edit, не Write» для существующих Markdown планов), (e) раздел «After approval» → «Plan promotion / housekeeping before implementation» (задачи в `На согласовании`).
- **v6** (2026-05-31, Codex review #5, **plan approved**): 4 финальных микро-правки — (a) заголовок v4 → v5 (и теперь v6), (b) удалён дубль risk про `list_work_items filter EN` (оставлена правильная conditional multi-domain версия), (c) удалён дубль risk про `bulk_update_tasks` (оставлена более точная про атомарность), (d) housekeeping шаг 6 — переименование плана после создания PM-MCP задачи (когда ID появляется), не после перевода в `К выполнению` (это independent approval шаг).

## Context

Агенты (Claude, Codex, ChatGPT) при работе с PM-MCP регулярно гадают значения `status`/`priority` потому что:
- БД хранит EN (`backlog`/`proposed`/`ready`/`in_progress`/`done`/`obsolete`).
- API наружу возвращает только RU; `_validate_status("done")` отвергает EN.
- Skill `pm-mcp-task-flow` запрещает гадать, но нет таблицы маппинга.

**Что уже есть в коде (verified read-only 2026-05-31):**

| Артефакт | Расположение | Статус |
|---|---|---|
| `RU_TO_DB_STATUS` / `status_to_db` / `status_to_ru` | `app/task_store.py:15-70` | есть |
| `RU_TO_DB_PRIORITY` / `priority_to_db` / `priority_to_ru` | `app/task_store.py:24-82` | есть |
| `TaskStatus = Literal[...]` RU (6), `TaskPriority` RU (4) | `server.py:80-103` | **#704 done (commit 6cf981e)** |
| Tools используют `TaskStatus`/`TaskPriority` | `create_task` (3494), `move_task` (3533), `update_task_status` (4389), `list_work_items` (3591) | **#704 done** |
| `_validate_status` / `_validate_priority` | `server.py:510, 526` | принимают только RU |
| `_task_to_dict` (sets status/priority) | `server.py:594` | RU only |
| `_project_task_to_work_item` (sets status/priority) | `registry.py:436` | RU only |
| `_matches_work_item_filters` filter | `registry.py:486` | точное равенство, нет EN normalize |
| `update_task_status complete` | `server.py:4389` | `complete = new_status.strip() == DONE_STATUS` до нормализации |
| `workflow.run_completed` publish | `server.py:2853` | публикуется только при `complete=True` |
| `WorkItem` pydantic config | `app/schemas/contracts.py:7` | `SchemaModel(model_config=ConfigDict(extra="forbid"))` |
| `Goal.status` (отдельный enum) | `contracts.py:46` | `active`/`archived` — другая taxonomy, **out of scope** |

## Goal

1. Агенты могут передавать оба варианта (`status="done"` или `status="Готово"`) — оба валидны, нормализуются в RU canonical для API/storage.
2. WorkItem в выдаче содержит `status` (RU canonical) + `status_en` (EN alias) параллельно; симметрично `priority` + `priority_en`. Grep по обоим работает.
3. MCP `tools/list` schema показывает все 12 валидных input значений в enum (machine-readable, не docstring).
4. Existing RU-only consumers (assistant-ui, gateway, тесты) не ломаются.

## Constraints / non-goals

**Constraints:**
- БД остаётся EN canonical (не менять storage layer).
- Output `status`/`priority` остаются RU canonical (backwards-compat).
- `WorkItem.SchemaModel extra="forbid"` — новые fields явно объявлять.
- `VALID_TASK_STATUSES` (RU 6 значений) **не расширять** — используется в summary/counts/transitions; расширение даст лишние нули в отчётах.
- Tech stack по `tech-stack-choices.md` #5: `Literal` в type-hints для FastMCP schema-export.
- Priority leniency (`.lower()`) **сохраняется** — текущий `_validate_priority` принимает "HIGH" и "high"; не ужесточаем до strict lowercase.

**Non-goals:**
- Goals (`list_goals`/`goal_review`, `contracts.py:46`) — другой EN-enum (`active`/`archived`), отдельный schema-hardening item.
- Process state manager (PSM) enums.
- Memory kinds (`fact`/`decision`/...) — AI-memory, уже EN.
- Domain enum (`project_tasks`/...) — уже EN, bilingual не нужен.
- Type enum для tasks — отдельный аудит, не этот план.
- Перенос canonical в БД на RU.
- Schema migration БД.
- Изменение output `status` RU → EN (ломает consumers).

## Phase 1+2 — `pm-mcp-server` contract + docs (one PM-MCP task, one commit)

Контракт и документация одним коммитом — public API change без docs ломает consumers.

### Code changes

**`app/task_store.py`**:
- `STATUS_ALIASES: dict[str, str]` — computed: RU→RU identity + EN→RU (через `RU_TO_DB_STATUS`); 12 ключей.
- `PRIORITY_ALIASES` симметрично; 8 ключей.
- `normalize_status_input(value: str) -> str` — возвращает RU canonical; `KeyError` если не найдено.
- `normalize_priority_input(value: str) -> str` — симметрично, **сохраняет** `.lower()` leniency для совместимости.

**`server.py`**:
- `VALID_TASK_STATUS_INPUTS = tuple(STATUS_ALIASES.keys())` — 12 значений для schema/error response.
- `VALID_TASK_PRIORITY_INPUTS = tuple(PRIORITY_ALIASES.keys())` — 8.
- `TaskStatusInput` type alias для tool input signatures (наряду с существующим `TaskStatus` RU-only для display). **Использовать explicit `Literal["Бэклог", "На согласовании", "К выполнению", "В работе", "Готово", "Не актуально", "backlog", "proposed", "ready", "in_progress", "done", "obsolete"]`** — не dynamic `Literal[*VALID_TASK_STATUS_INPUTS]`, потому что Pydantic/FastMCP может не экспортировать enum из dynamic tuple в JSON-schema. Если dynamic подхватывается (verified через `tools/list` snapshot в acceptance) — допустимо ужать до computed.
- `TaskPriorityInput` симметрично: explicit `Literal["низкий", "средний", "высокий", "критический", "low", "medium", "high", "critical"]`.
- `_validate_status` (line 510): первой строкой `status = normalize_status_input(status.strip())` (KeyError → `ApiError("invalid_status", ..., allowed_statuses=<RU 6>, allowed_status_inputs=<12>)`). Сохранить `allowed_statuses` RU-only — клиенты могут показывать пользователю.
- `_validate_priority` (line 526): `priority = normalize_priority_input(priority)` (включая `.lower()`).
- Tool signatures обновить на `TaskStatusInput`/`TaskPriorityInput`: `create_task`, `move_task`, `update_task_status`. **`list_work_items` — особый случай**: tool multi-domain (`project_tasks`/`todoist`/`calendar`/`obsidian`/`budget`/`gmail`), у каждого домена своя status taxonomy. Оставить `status: str | None`; нормализация alias применяется условно — только если `domain=project_tasks` (или resolved через project_path → adapter project_tasks). Для других доменов status передаётся as-is (там собственные enum-системы).
- `bulk_update_tasks` (атомарный DB path): использовать общий helper нормализации (`normalize_status_input` / `normalize_priority_input` напрямую перед DB write), **НЕ маршрутизировать через `_change_task_status`** — он не bulk-oriented (одна задача → side effects → events).
- `update_task` (fields dict): отдельно решить контракт — поддерживается ли `fields["status"]` вообще; если да — применить тот же `normalize_status_input` helper при apply, не через `_change_task_status`.
- `update_task_status` (line 4389): нормализовать `new_status` **до** вычисления `complete = ... == DONE_STATUS`. Иначе `new_status="done"` теряет `workflow.run_completed` event (`server.py:2853`).
- `_task_to_dict` (line 594): добавить `status_en=status_to_db(task.status)`, `priority_en=priority_to_db(task.priority)`.

**`app/adapters/registry.py`**:
- `_project_task_to_work_item` (line 436): добавить `status_en`/`priority_en` симметрично `_task_to_dict`.
- `_matches_work_item_filters` (line 486) ИЛИ wrapper в `list_work_items`: нормализовать `status` param через `normalize_status_input` **только если domain=`project_tasks`** (согласовано с multi-domain caveat в §Phase 1 Tool signatures). Для других доменов (todoist/calendar/obsidian/budget/gmail) status pass-through без alias-нормализации.

**`app/schemas/contracts.py`**:
- `WorkItem` (наследует `SchemaModel` с `extra="forbid"`): явно объявить `status_en: str | None = None` и `priority_en: str | None = None`. Без объявления — runtime `ValidationError`.

### Tests

- `tests/test_tasks_db.py`: параметрический тест — `normalize_status_input(<each ru>) == normalize_status_input(<each en>)` для всех 6 пар; то же для priority (4 пары + case variants).
- `tests/test_task_flow.py`:
  - `move_task(project, task_id, status="done")` → задача в статусе "Готово" + `workflow.run_completed` опубликован (assert через `monkeypatch publish_event`).
  - `update_task_status(project, task_id, new_status="done")` — отдельный тест от `move_task` (разные parameter names: `status` vs `new_status`); тот же assert.
  - `create_task(project, "Title", status="ready", priority="high")` → создаётся с RU canonical.
  - `list_work_items(project, status="in_progress")` → возвращает задачи в "В работе".
  - `_validate_priority("HIGH")` → `"высокий"` (regression на сохранённой leniency).
- `tests/test_mcp_api_contract.py`: snapshot `tools/list` для `create_task`; assert `inputSchema.properties.status.enum` содержит 12 значений (RU 6 + EN 6).
- `tests/test_schemas.py`: `WorkItem(..., status="Готово", status_en="done")` валиден; без `status_en` field — `status_en is None` (не валидация-ошибка).
- Регрессия: все existing RU-only тесты pass без изменений.

### Docs (тот же commit)

- `pm-mcp-server/docs/MCP_API.md`: таблица RU↔EN status/priority; пример `create_task` с EN.
- `pm-mcp-server/ARCHITECTURE.md`: упоминание `STATUS_ALIASES` + двух конструкторов WorkItem.
- `pm-mcp-server/AGENTS.md`: bilingual короткая заметка **на английском** (правило репо: AGENTS.md по-английски, исключение — RU task status names как литералы) + ссылка на MCP_API.md.

### Acceptance

- `_validate_status("done")` → `"Готово"` (нормализует EN).
- `_validate_status("Готово")` → `"Готово"` (regression RU).
- `_validate_status("invalid")` → `ApiError(allowed_statuses=[6 RU], allowed_status_inputs=[12])`.
- `_validate_priority("HIGH")` → `"высокий"` (leniency сохранена).
- `read_tasks` / `list_work_items` возвращает каждую задачу с `status` (RU) + `status_en` (EN); `priority` + `priority_en`.
- `WorkItem(status="Готово", status_en="done", priority="высокий", priority_en="high")` валиден; `WorkItem(status="Готово")` (без `status_en`) тоже валиден.
- `move_task(status="done")` И `update_task_status(new_status="done")` — оба публикуют `workflow.run_completed` (verified через `monkeypatch publish_event`).
- `list_work_items(status="in_progress")` → задачи "В работе".
- MCP `tools/list` для `create_task`: `inputSchema.properties.status.enum` содержит 12 значений.
- `uv run ruff check .` + `uv run pytest` green.

## Phase 3 — `_engineering_rules` (отдельная задача после Phase 1+2 закрыта)

**Запрет** обновлять skill с EN ДО реализации Phase 1+2 — иначе агенты начнут передавать `done` раньше сервера и получат `invalid_status`.

- `D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow\SKILL.md`: таблица маппинга RU↔EN; явное «server accepts both».
- Master `AGENTS.md` Section D: bilingual упоминание (короткое).
- AI-memory reference: `propose_memory(kind="fact", lifecycle="durable", agent="stepa", tags=["pm-mcp", "enum", "bilingual", "reference"])` с таблицей.

**Acceptance**: skill SKILL.md содержит таблицу; master AGENTS.md упоминает bilingual; memory entry создана и находится через `search_by_metadata`.

## Risks

| Риск | Mitigation |
|---|---|
| `VALID_TASK_STATUSES` расширение ломает summary/counts | `VALID_TASK_STATUS_INPUTS` отдельно для schema/error; `VALID_TASK_STATUSES` остаётся RU-only |
| `WorkItem extra="forbid"` runtime ValidationError | Явно объявить `status_en`/`priority_en` в `SchemaModel` |
| Lost `workflow.run_completed` event при EN input | Нормализовать `new_status` до `complete = ...`; тест `monkeypatch publish_event` |
| Два конструктора WorkItem рассинхрон | Symmetric tests; в идеале — общая helper-функция dict construction |
| Skill update до серверной поддержки EN | Phase 3 строго после Phase 1+2 `Готово` |
| Priority leniency `.lower()` regression | Тест `_validate_priority("HIGH")` → `"высокий"` |
| Goals enum dragged in by mistake | Goals — out of scope явно (active/archived, не пересекается) |
| Existing consumer parses `"Недопустимый статус: X"` | Текст error message не менять, только расширить `allowed_status_inputs` payload |
| Dynamic `Literal[*tuple]` не экспортируется в JSON-schema | Использовать explicit `Literal["...", "..."]` с 12/8 строками; verified через `tools/list` snapshot в acceptance |
| `list_work_items` status filter ломает multi-domain (где enum другой) | Conditional normalize только для project_tasks; для остальных доменов — pass-through `status: str \| None` |
| Маршрутизация `bulk_update_tasks` через `_change_task_status` ломает атомарность | Использовать shared normalize helper напрямую перед DB write; не вызывать per-task `_change_task_status` в bulk path |

## Acceptance после housekeeping (не текущее состояние)

**Эти шаги выполнены 2026-05-31 в plan-mode сессии Claude:**

- ✓ `D:\GitHub\_engineering_rules\plans\pm-mcp-bilingual-enums.md` содержит canonical v6.
- ✓ `D:\GitHub\_engineering_rules\plans\SUPERSEDED-pm-mcp-bilingual-enums-v1.md` — старый v1 архивирован.
- ✓ `C:\Users\Zaxva\.claude\plans\imperative-beaming-seahorse.md` — одна строка `См. план: ...`.

**Завершение 2026-05-31:**
- ✓ PM-MCP задача `#739` закрыта: Phase 1+2 реализованы в `pm-mcp-server`, `uv run ruff check .` и `uv run pytest` проходят.
- ✓ PM-MCP задача `#740` закрыта: `pm-mcp-task-flow` и master `AGENTS.md` описывают bilingual enum mapping, AI-memory reference создана.
- ✓ План переименован в `DONE-739-pm-mcp-bilingual-enums.md`; harness one-liner обновлён.
