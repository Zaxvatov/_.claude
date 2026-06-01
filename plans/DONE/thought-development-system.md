# Система развития мыслей и идей — MVP

## Контекст

Пользователь хочет, чтобы фраза «зафиксируй это» в любом чате
(ChatGPT/Claude/Codex) запускала жизненный цикл идеи: capture → поиск →
развитие → promote-to-task. Сейчас стек (ai-memory, PM-MCP, Assistant-UI,
Obsidian, Todoist, Calendar) существует разрозненно, а у мыслительных
продуктов (идеи, гипотезы, концепции, промпты, исследования) нет ни
единого приёмника, ни механизма «вернуть забытую мысль в нужный момент».

Цель MVP — построить минимальный приёмник идей поверх существующих
сервисов, не плодя новых сущностей и БД-таблиц, чтобы:
- любую идею можно было зафиксировать одной MCP-командой,
- идея сразу попадала в семантический поиск ai-memory,
- идею можно было явно «promote» в Task в PM-MCP,
- идею можно было «выгрузить» в Obsidian для длинного чтения/правок,
- UI давал ручную сортировку, capture-форму, promote и export-кнопку.

Полноценный resurfacing (cron, event-driven подсветка, morning_brief
интеграция) и двусторонний Obsidian-sync — пост-MVP.

## История review

- Codex round 1 (AI-memory `#1386`) указал 4 критичных gap: (1) dedupe
  blocker для same-text supersede, (2) ненадёжный lookup по
  `metadata.idea_id`, (3) поломка существующих `note`-writer'ов из-за
  обязательного `type`, (4) compensation в promote ссылался на
  несуществующий статус `Не актуально`. Также CSRF, path-traversal в
  Obsidian export, YAML-инъекций.
- Codex round 2 (AI-memory `#1387`) указал 6 уточнений: (1) неверные
  аргументы PM-MCP tools (`create_task` без `description`,
  `update_task`/`add_task_tag` через `project_path+task_id`, не
  `item_id`); (2) capture может потерять `idea_id` через `skipped_duplicate`;
  (3) `search_by_metadata` должен возвращать object-shaped payload
  `{status, items, count, filters}`, фильтры allowlist-based;
  (4) `superseded_inplace` ломает контракт `store_memory` (callers ожидают
  `ok`/`skipped_duplicate`), вместо нового статуса вернуть
  `{status:"ok", superseded:true, id, archived_id}`; (5) `/ideas?q=...`
  технически не описан; (6) env var: переиспользовать
  `PM_MCP_OBSIDIAN_VAULT_PATH` либо назвать явно, чтобы не получить drift.
- Codex round 3 (AI-memory `#1388`) указал 5 точечных правок: (1) flat vs
  nested filters payload в `search_by_metadata` — выбрать одно
  (рекомендация: nested `{"filters": {...}}`); (2) capture lookup через
  `search_memory(limit=5)` хрупкий — лучше через `store_memory` →
  `skipped_duplicate` → `get_memory_entry(id)`; (3) `archive()` helper
  открывает отдельное соединение, для одной транзакции нужна
  internal SQL function на том же `conn`; (4) `PyYAML` отсутствует в
  `assistant-ui/pyproject.toml` — добавить в коммит 6 + `uv.lock`;
  (5) старые упоминания `OBSIDIAN_VAULT_PATH` без префикса в тексте.
  Плюс: composite index `(type, lifecycle, subtype)` под `/ideas`.
- Codex round 4 (AI-memory `#1389`) указал 5 точностных правок: (1) три
  оставшихся flat-примера `search_by_metadata` в тексте (acceptance
  коммита 1, шаг 1 коммита 6, шаг 8 verification); (2) capture upgrade
  должен формулироваться шире — любая non-idea note, включая legacy без
  явного `type` после default-логики; (3) добавить тест повторного
  capture текста уже-promoted идеи (должен вернуть существующий
  `idea_id`); (4) `Inbox/` каталог может не существовать, нужен
  `mkdir(parents=True, exist_ok=True)` перед atomic write;
  (5) verification-шаг `get_work_item` лучше с явным `project_path`
  против multi-domain ambiguity.
- Codex round 5 (AI-memory `#1390`) указал 5 финальных правок: (1) ещё
  один flat-пример `search_by_metadata` в acceptance коммита 3;
  (2) `get_memory_entry` уточнить: archived только при
  `include_archived=true`; (3) read-path fallback для legacy note без
  явного `metadata.type` в алгоритме capture (защита от `KeyError`);
  (4) `priority:"medium"` в verification → опустить либо
  использовать реальное enum-значение из PM-MCP (правило D root
  AGENTS.md про невыдумывание enum); (5) cross-subsystem work items
  — каждый коммит как отдельная PM-MCP задача под `project_path`
  затронутого subsystem'а с явными `link_task_dependency`.
- Codex round 6 указал 2 финальные мелочи: (1) acceptance-строка
  про `get_memory_entry` всё ещё противоречила описанию и тестам —
  заменить на конкретные case'ы (active / archived без флага /
  archived с флагом); (2) `за O(log n)` слишком обещающе для SQLite
  JSON expression index — заменить на формулировку через индекс с
  явной проверкой `EXPLAIN QUERY PLAN`.
- Codex round 7 указал 1 практический риск: post-hoc фильтр в
  `/ideas?q=...` записан как `r.metadata.type=="idea"` —
  атрибутный доступ упадёт на dict-shaped результатах `search_memory`,
  а legacy/general note могут не иметь `metadata.type` или иметь
  `metadata=None`. Заменить на безопасный `(r.get("metadata") or {}).get("type") == "idea"`
  и добавить тест на устойчивость к смеси результатов.
- Codex round 8 указал симметричный риск в capture: после
  `get_memory_entry` использовался attribute-style доступ
  `existing.kind`, `existing.metadata.type`. `MCPClients.call_memory`
  возвращает dict, а `get_memory_entry` — object-shaped payload.
  Зафиксирован безопасный unwrap (`lookup["item"]`, `metadata.get(...)`)
  + явная обработка `lookup.status != "ok"` как 502/diagnostic
  (`error:"lookup_failed"`) для race-condition между
  `skipped_duplicate` и `get_memory_entry`. Тесты дополнены.
- Все замечания учтены ниже.

## Принятые архитектурные решения

| # | Решение | Обоснование |
|---|---|---|
| 1 | Идея = `MemoryEntry(kind="note")` с дискриминатором `metadata.type="idea"` | Новые `kind` запрещены ADR-001 D-4 (5 фиксированных). `note` семантически ближе всего к идее |
| 2 | Lifecycle через `metadata.lifecycle` (convention) | Нет нужды в БД-enum; whitelist в `validate_metadata` |
| 3 | ai-memory = «первая остановка мысли» (Модель 2) | Capture-to-search путь короткий; основные потребители — чат-агенты с `search_memory` |
| 4 | Obsidian on-demand, не auto | Не засоряем vault при каждом capture; пользователь сам решает, какие идеи нуждаются в длинном тексте |
| 5 | Promote = композит в `assistant-ui` | Уже владеет `MCPClients.pm` + `MCPClients.memory`; gateway stateless |
| 6 | Resurfacing pull-only через `search_memory`/`search_by_metadata` | Явный выбор пользователя; cron/event-driven — пост-MVP |
| 7 | Capture-tool — convention-only, без нового MCP tool | Дублировал бы `store_memory` 1:1; контракт валидируется в одной точке |
| 8 | Переиспользуем существующие metadata-поля | `work_item_id` (после promote), `goal_id`, `supersedes_memory_id`, `summary_of` |
| 9 | `metadata.type` обязательно ТОЛЬКО для `kind=note` | Остальные kinds сами по себе типы. Whitelist для note: `{"idea","general","summary","audit-finding"}` |
| 10 | `validate_metadata` сам ставит default `type` для note без явного значения | Не ломает существующие `note`-writer'ы (`store_summary`, proposal `approve`): если `metadata.summary_of` есть → `summary`, иначе → `general`. `idea` всегда требует явного указания через `capture_idea` |
| 11 | Same-identity supersede через явный транзакционный путь | Текущий dedupe в `storage.py:81` срабатывает раньше `supersedes_memory_id` (`storage.py:135`), значит metadata-only update «тихо» отбрасывается. Чиним в коммите 1 |
| 12 | Lookup идеи по `metadata.idea_id` через `search_by_metadata` + JSON1 индексы | FTS индексирует только `text` (`db.py:141`); `search_by_workitem` фильтрует только `work_item_id` и делает Python scan (`search.py:518`). Без надёжного lookup `promote`/`export` будут «не находить» старые идеи |
| 13 | Compensation в promote — без `Не актуально` | В PM-MCP whitelist статусов: `Бэклог`, `На согласовании`, `К выполнению`, `В работе`, `Готово` (`server.py:82`). `close_task` требует `В работе/Готово` (`server.py:4589`). Откат — через `add_task_tag` + `update_task(description=...)`, без авто-закрытия |
| 14 | Унификация `metadata.task_id` vs `metadata.work_item_id` — отдельный план | Используем `work_item_id` (уже в whitelist `validation.py`). Известный гэп фиксируется в открытых вопросах |
| 15 | `search_by_metadata` контракт: object-shaped payload + allowlist фильтров | Соответствует существующему контракту collection-tools ai-memory; allowlist предотвращает JSON-path injection и завязывает фильтры на индексы |
| 16 | `store_memory` для supersede возвращает `{status:"ok", superseded:true, ...}`, без нового статуса | Сохраняет обратную совместимость с `store_batch` и существующими callers, которые считают `ok` как успех |
| 17 | `capture_idea` идемпотентен по `(text, project, agent)` | При повторном capture того же текста — возврат существующего `idea_id` без новых записей. General-note того же writer'а апгрейдится до idea через same-identity supersede |
| 18 | `/ideas?q=...` идёт через `search_memory` (FTS+embedding), без `q` — через `search_by_metadata` (exact JSON1) | `search_by_metadata` не имеет текстового поиска; `search_memory` не имеет точного metadata-фильтра. Развилка по `q` даёт лучшее из обоих |
| 19 | PM-MCP env: fallback `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` → `PM_MCP_OBSIDIAN_VAULT_PATH` | Не вводим общий `OBSIDIAN_VAULT_PATH` (drift-риск); переиспользуем существующую переменную PM-MCP когда явная не задана |
| 20 | `search_by_metadata` — nested payload `{"filters": {...}}` | Единый stable parameter в tool schema; call sites не разъезжаются между flat/nested |
| 21 | Same-identity supersede в одной транзакции через private `_archive_within_conn(conn, ...)` | Публичный `archive()` открывает отдельное соединение — не atomic. Нужна internal helper-функция на одном `conn` под `with conn:` |
| 22 | Capture lookup — через `store_memory → skipped_duplicate → get_memory_entry`, не через FTS-поиск | Детерминированный dedupe (по unique index на `text/project/agent/kind`) точнее FTS ranking; не зависит от качества embedding |
| 23 | Composite index `idx_memory_ideas_listing(type, lifecycle, subtype)` | Основной use-case `/ideas` listing с фильтрами — без composite index деградирует на больших объёмах. `goal_id` пока без индекса (MVP-acceptable) |
| 24 | PyYAML добавляется в `assistant-ui/pyproject.toml` | `yaml.safe_dump`/`safe_load` нужны для frontmatter; raw-string concatenation запрещён для защиты от YAML-инъекций. `uv.lock` обновляется одним коммитом с кодом |

## Контракт записи идеи

```json
{
  "kind": "note",
  "text": "<краткая формулировка идеи, до MAX_TEXT_LENGTH>",
  "project": "portfolio | <project name>",
  "agent": "claude | codex | user | chatgpt",
  "metadata": {
    "type": "idea",
    "lifecycle": "incubating | developing | mature | dormant | promoted | archived",
    "idea_id": "<uuid4>",
    "subtype": "concept | hypothesis | prompt | research | decision-draft | goal-seed",
    "source": "<canonical agent id, lowercase>",
    "tags": ["..."],
    "goal_id": "<optional>",
    "work_item_id": "<optional; устанавливается при promote>",
    "obsidian_path": "<optional; относительный путь к .md после export>",
    "supersedes_memory_id": "<optional; id предыдущей итерации>"
  }
}
```

Cross-field правила в `validate_metadata`:
- Если `kind == "note"` и `type` отсутствует → default: `summary` (если есть `summary_of`), иначе `general`.
- Если `kind == "note"` и `type` указан → whitelist `{"idea","general","summary","audit-finding"}`.
- Если `metadata.type == "idea"` → дополнительно обязательны `lifecycle`, `subtype`, `idea_id`.

## План на 6 атомарных коммитов

### Коммит 1 — Foundation: same-identity supersede + metadata lookup

Это технический фундамент. Без него любые metadata-only updates (promote/export) «тихо» отбрасываются текущим dedupe, а lookup идеи по `idea_id` ненадёжен. Коммит ни от чего не зависит и полезен сам по себе для существующих use-cases.

**Файлы:**
- `ai-memory/memory/storage.py`:
  - Перед dedupe-проверкой (`storage.py:81`) обработать `supersedes_memory_id` из `metadata`. Если указан и целевая запись активна и совпадает по (text, project, agent, kind) — выполнить в **одной транзакции на одном `conn`**: internal SQL-функция `_archive_within_conn(conn, old_id, reason)` (новый private helper; **не** публичный `archive()`, который открывает отдельное соединение) + `insert(new_entry)`. Используется `with conn:` context для atomic commit/rollback.
  - Возвращать **обратно-совместимо**: `{status: "ok", superseded: true, id: new_id, archived_id: old_id}`. Без нового статуса — `store_batch` и существующие callers продолжат считать `ok` как успех.
  - Если same-identity ситуация без `supersedes_memory_id` — поведение прежнее (`skipped_duplicate` с возвратом `id` существующей записи; этот контракт уже работает и используется в коммите 3 для capture-идемпотентности).
  - Перенести embedding старой записи в новую при metadata-only update (если `text` идентичен и embedding не пустой), чтобы не делать повторный encode.
- `ai-memory/memory/db.py`:
  - Поднять `CURRENT_SCHEMA_VERSION` с 8 до 9.
  - Миграция 8→9: добавить SQLite-индексы:
    - `idx_memory_metadata_type` (JSON1 `json_extract(metadata,'$.type')`).
    - `idx_memory_metadata_idea_id` (JSON1 `json_extract(metadata,'$.idea_id')`).
    - `idx_memory_metadata_work_item_id` (JSON1 `json_extract(metadata,'$.work_item_id')`).
    - Composite `idx_memory_ideas_listing` на `(json_extract(metadata,'$.type'), json_extract(metadata,'$.lifecycle'), json_extract(metadata,'$.subtype'))` — под основной use-case `/ideas` listing с фильтрами.
    - Все партиальные (`WHERE archived_at IS NULL`).
- `ai-memory/memory/search.py`:
  - Новая функция `search_by_metadata(*, filters: dict, project=None, agent=None, kind=None, limit=DEFAULT_LIMIT) -> dict`. **Nested payload**: filters передаются единым parameter dict (call sites используют `{"filters": {"type":"idea","idea_id":...}}`). Возвращает **object-shaped payload** в соответствии с контрактом collection-tools ai-memory: `{"status":"ok","items":[...],"count":N,"filters":{...},"limit":L}`. Фильтры **allowlist-based**: разрешённые ключи `{"type","idea_id","work_item_id","lifecycle","subtype","goal_id"}` — произвольный ключ → `ValueError` (защита от JSON-path injection). Индексы из коммита 1 покрывают `type`/`idea_id`/`work_item_id` напрямую и `lifecycle`/`subtype` через composite `idx_memory_ideas_listing`; `goal_id` пока без индекса (sequential scan, MVP-acceptable). Возвращает активные записи, сортированные по `created_at desc`.
  - Новая функция `get_memory_entry(memory_id: int, *, include_archived: bool = False) -> dict`. Возвращает object-shaped `{"status":"ok","item":{...}}` или `{"status":"not_found"}`. По умолчанию ищет только активные (`archived_at IS NULL`); архивированные возвращает только при явном `include_archived=true`.
- `ai-memory/memory/mcp_app.py`:
  - Зарегистрировать новые MCP tools: `search_by_metadata`, `get_memory_entry`. Schema input/output по аналогии с существующими.
- `ai-memory/memory/runtime_contract.py`:
  - Обновить публичный список tools (если есть инвариант на минимум публичных tools — расширить).
- `ai-memory/tests/test_storage.py`:
  - Тест «same text, metadata-only supersession» — два store с одинаковым text, второй с supersedes_memory_id → старая архивирована, новая активна с обновлённым metadata. Результат: `{status:"ok", superseded:true, ...}`.
  - Тест «embedding копируется при same-text supersede».
  - Тест: `store_batch` корректно учитывает supersede как успех (через существующий `ok`-accounting).
- `ai-memory/tests/test_search.py`:
  - Тесты `search_by_metadata` (фильтр по type, по idea_id, по двум полям, по проекту). Проверка object-shaped payload.
  - Тест allowlist: фильтр с произвольным ключом (`{"random_key":"x"}`) → `ValueError`.
  - Тест `get_memory_entry`: active возвращается по умолчанию; archived → `{"status":"not_found"}` без `include_archived`; с `include_archived=true` → возвращается.
- `ai-memory/tests/test_db.py`:
  - Тест миграции 8→9: индексы созданы, идемпотентность повторного запуска.

**Acceptance:**
- Старая запись с `text="X"`, `metadata={"foo":1}` → новая запись с `text="X"`, `metadata={"foo":2,"supersedes_memory_id":<old_id>}` → старая `archived_at IS NOT NULL`, новая активна с `foo=2`.
- `search_by_metadata({"filters":{"type":"idea","idea_id":"abc"}})` находит запись через индекс (план запроса проверяется через `EXPLAIN QUERY PLAN`; ожидаем `SEARCH ... USING INDEX idx_memory_metadata_idea_id`, без `SCAN`).
- `get_memory_entry(<active_id>)` возвращает active; `get_memory_entry(<archived_id>)` → `{"status":"not_found"}`; `get_memory_entry(<archived_id>, include_archived=true)` возвращает archived.
- `uv run pytest && uv run ruff check .` зелёные в `ai-memory/`.

### Коммит 2 — Idea contract + валидатор + миграция + ADR

**Файлы:**
- `ai-memory/memory/validation.py` — расширить `validate_metadata`:
  - Принимает опциональный `kind` (как контекст для cross-field).
  - Нормализация `type` (canonical lowercase) — при наличии.
  - Нормализация `lifecycle` (canonical lowercase, whitelist 6 значений).
  - Нормализация `subtype` (canonical lowercase, whitelist 6 значений).
  - Нормализация `idea_id` (string + min-length 8).
  - Нормализация `obsidian_path` через `_normalize_metadata_string_field`.
  - Cross-field 1: если `kind == "note"`:
    - Если `type` отсутствует → default: `summary` (если `summary_of`), иначе `general`.
    - Если `type` указан → должен быть в whitelist `{"idea","general","summary","audit-finding"}`.
  - Cross-field 2: если `type == "idea"` → `lifecycle`, `subtype`, `idea_id` обязательны.
- Все вызовы `validate_metadata` (mcp_app.py, storage.py, proposals/*, search.py:store_summary) пробросить `kind` параметром.
- `ai-memory/memory/db.py` — миграция 8→9 (уже включена в коммит 1 для индексов) НЕ трогает данные `note`-записей: дефолтный `type` теперь проставляется на лету `validate_metadata` при следующем write. Если нужна предварительная backfill — добавить отдельный CLI `python -m memory.cli backfill-metadata-type` (опционально, не блокирует MVP, потому что валидатор делает дефолт автоматически).
- `ai-memory/tests/test_validation.py`:
  - `validate_metadata({}, kind="note")` → проходит с `type="general"` (default).
  - `validate_metadata({"summary_of":{...}}, kind="note")` → проходит с `type="summary"` (default).
  - `validate_metadata({"type":"idea"}, kind="note")` → ValueError про lifecycle.
  - `validate_metadata({"type":"idea","lifecycle":"incubating","subtype":"concept","idea_id":"abc-12345"}, kind="note")` → проходит.
  - `validate_metadata({}, kind="fact")` → проходит без type.
  - `validate_metadata({"type":"unknown"}, kind="note")` → ValueError.
- `ai-memory/tests/test_summary.py` (или существующий) — `store_summary` всё ещё работает, в записи появляется `type="summary"` автоматически.
- `ai-memory/tests/test_proposals_promote.py` — proposal approve пишет note с `type="general"` (или прокидываемым).
- `ai-memory/docs/CONTRACT.md` — секции про idea-convention и type для note.
- `docs/adrs/0004-idea-memory-contract.md` (новый, English) фиксирует:
  - выбор «note + metadata.type=idea» против отдельного kind/domain,
  - lifecycle convention,
  - обязательность `type` для `kind=note` с auto-default,
  - переиспользование существующих metadata-полей,
  - known limitations: ChatGPT-capture через `memory_propose` без полного metadata.
- `docs/adrs/README.md` — индекс.

**Acceptance:**
- Все существующие `note`-writer'ы (`store_summary`, proposal `approve`, batch) работают без изменений — `type` проставляется автоматически.
- Cross-field validation покрыт тестами.
- ADR-0004 опубликован.
- `uv run pytest && uv run ruff check .` зелёные в `ai-memory/`.

### Коммит 3 — Capture path (assistant-ui)

**Файлы:**
- `assistant-ui/app/ideas.py` (новый модуль) — `capture_idea(clients, *, text, subtype, source, tags, project, agent=None, goal_id=None) -> dict`. Алгоритм (опирается на детерминированный dedupe ai-memory, а не FTS ranking):
  1. Сгенерировать `idea_id = uuid4()`. Попытка обычного `store_memory(text, project, agent, kind="note", metadata={type:"idea", lifecycle:"incubating", idea_id, subtype, source, tags, goal_id})`.
  2. Если ответ `{status:"ok"}` → новая запись, вернуть `{"ok":true,"idea_id","memory_id":<id>,"deduplicated":false,"upgraded_from":null}`.
  3. Если ответ `{status:"skipped_duplicate", id:<existing_id>}` (контракт ai-memory уже возвращает `id` существующей записи при duplicate) → `lookup = clients.call_memory("get_memory_entry", {memory_id: existing_id})`.
     - **Безопасный unwrap dict-shaped payload** (`MCPClients.call_memory` возвращает обычный dict, не object):
       ```
       if lookup.get("status") != "ok":
           # diagnostic: skipped_duplicate был, но get_memory_entry не вернул запись
           # (race condition / БД-несогласованность)
           return 502 с {"ok":false,"error":"lookup_failed","status":lookup.get("status"),"memory_id":existing_id}
       existing = lookup["item"]
       metadata = existing.get("metadata") or {}
       existing_kind = existing.get("kind")
       existing_type = metadata.get("type") or ("summary" if metadata.get("summary_of") else "general")
       existing_idea_id = metadata.get("idea_id")
       ```
       **Read-path fallback для `existing_type`** покрывает legacy/general записи без явного `metadata.type`: default-логика `validate_metadata` коммита 2 работает на write-path, но БД-записи, прочитанные до повторного write, могут не иметь `type` — fallback защищает от `KeyError`/`AttributeError`.
     - Если `existing_kind == "note"` и `existing_type == "idea"` (любой `lifecycle`, включая `promoted`/`dormant`) → **идемпотентность**: вернуть `{"ok":true,"idea_id":existing_idea_id,"memory_id":existing_id,"deduplicated":true,"upgraded_from":null}`. Никаких новых записей. Это покрывает повторный capture того же текста для уже-promoted идеи.
     - Если `existing_kind == "note"` и `existing_type != "idea"` → повторный `store_memory(...)` с `metadata.supersedes_memory_id = existing_id`. Сработает same-identity supersede из коммита 1 — старая не-idea note архивируется, новая idea активна. Вернуть `{"ok":true,"idea_id","memory_id":<new_id>,"deduplicated":false,"upgraded_from":existing_type}`.
  4. Возврат всегда содержит `deduplicated:bool` и `upgraded_from:str|null` — UI/тесты опираются на эти флаги.
- `assistant-ui/app/main.py` — `POST /api/ideas/capture` (Pydantic `CaptureIdeaRequest`).
- `assistant-ui/app/security.py` — добавить `/api/ideas/capture` в protected paths (если не покрыт wildcard `/api/*`).
- `assistant-ui/tests/test_ideas_capture.py` — мок-тесты:
  - happy path (`store_memory` → `ok` → новая запись).
  - идемпотентность incubating idea: `store_memory` → `skipped_duplicate` + existing.metadata.type=="idea" / lifecycle=="incubating" → возврат существующего `idea_id`, `deduplicated:true`, без повторных store-вызовов.
  - **идемпотентность promoted idea**: повторный capture текста уже-promoted идеи → возврат существующего `idea_id` (с `work_item_id` в metadata), `deduplicated:true`, новой incubating-записи не создаётся.
  - upgrade general-note → idea: `store_memory` → `skipped_duplicate` + existing.metadata.type=="general" → второй `store_memory` с `supersedes_memory_id` → `upgraded_from:"general"`.
  - upgrade summary-note → idea: то же, но `upgraded_from:"summary"`.
  - корректность call-граф: проверка, что `get_memory_entry` вызывается только при `skipped_duplicate`.
  - **безопасный unwrap dict-shaped payload**: `get_memory_entry` возвращает `{"status":"ok","item":{...}}` с `metadata=None` или без ключа `type` → capture не падает на `AttributeError`/`KeyError`, корректно применяет read-path fallback.
  - **lookup_failed diagnostic**: `get_memory_entry` возвращает `{"status":"not_found"}` после `skipped_duplicate` (имитация race condition) → capture возвращает 502 с `error:"lookup_failed"`, без второго `store_memory`.

**Acceptance:**
- POST с минимальным body → 200, запись в ai-memory корректна.
- Повторный POST с тем же text/project/agent → тот же `idea_id`, `deduplicated:true`, новых записей в БД нет.
- POST с text, для которого уже есть general-note того же writer'а → старая архивирована, новая idea активна с новым `idea_id`.
- `search_by_metadata({"filters":{"type":"idea","idea_id":...}})` находит запись.
- `uv run pytest && uv run ruff check .` зелёные в `assistant-ui/`.

**Known limitation:** ChatGPT через gateway → `memory_propose` принимает только partial metadata. В MVP capture от ChatGPT — без `idea_id`/`subtype`/`lifecycle`. Пост-MVP — расширить proposal-контракт.

### Коммит 4 — Promote operation (assistant-ui)

**Файлы:**
- `assistant-ui/app/ideas.py` — `promote_idea(clients, *, idea_id, project_path, title, priority=None, description=None, tags=None, related_goals=None) -> dict`:
  1. Lookup через `clients.call_memory("search_by_metadata", {"filters":{"type":"idea","idea_id":idea_id}})`. Если запись не найдена / `lifecycle == "promoted"` → 404 / 409.
  2. Создать Task через `clients.call_pm("create_task", {project_path, title, priority, tags, related_goals})`. **Важно**: `create_task` (см. `pm-mcp-server/server.py:3631`) НЕ принимает `description` — принимает `project_path, title, status, priority, assignee, tags, related_goals`. Получить `task_id` (PM-MCP `#NNN`).
  3. Если `description` задан — отдельный вызов `clients.call_pm("update_task", {project_path, task_id:<#NNN>, fields:{"description":<description>}})`. `update_task` (см. `server.py:3658`) принимает `project_path, task_id, fields`. Если этот шаг падает — переходим к compensation (как ниже).
  4. Записать новую memory с теми же `text`/`project`/`agent`/`kind`, `metadata = {...old, lifecycle:"promoted", work_item_id:"#NNN", supersedes_memory_id:<old_id>}`. Сработает same-identity supersede из коммита 1 — старая запись архивируется, новая активна, embedding копируется.
  5. Если шаг 3 или 4 упал после успешного шага 2 — compensation БЕЗ `close_task` (нет статуса `Не актуально` в whitelist `pm-mcp-server/server.py:82`):
     - `clients.call_pm("add_task_tag", {project_path, task_id:<#NNN>, tag:"assistant-managed-rollback"})` (см. `server.py:3705` — сигнатура `project_path, task_id, tag`).
     - `clients.call_pm("update_task", {project_path, task_id:<#NNN>, fields:{"description":"ROLLBACK: promote из idea <idea_id> упал. Задача создана в Бэклоге, требуется ручное закрытие или повторный promote."}})`.
     - Возврат 502 с `{"ok":false,"error":"...","compensated":true,"task_id":<#NNN>,"action_required":"Проверь #<task_id> в PM-MCP"}`.
  6. Hook `_after_promote(old_entry, new_entry)` — пустая реализация; расширяется в коммите 6.
- `assistant-ui/app/main.py` — `POST /api/ideas/{idea_id}/promote` (Pydantic `PromoteIdeaRequest`).
- `assistant-ui/tests/test_ideas_promote.py` — happy + compensation + idempotency (409).

**Acceptance:**
- Happy: Task создан в PM-MCP (статус `Бэклог`), новая memory с `lifecycle=promoted` + `work_item_id`, старая в `archived_at IS NOT NULL`.
- Compensation: при искусственной ошибке шага 3 — task в Бэклоге с тегом `assistant-managed-rollback` и diagnostic description, HTTP 502 с `action_required`.
- Идемпотентность: повторный promote той же `idea_id` → 409 с указанием существующего `work_item_id`.

**Открытый вопрос (для следующей итерации):** добавление официального cancel-status в PM-MCP (отдельный ADR), чтобы compensation мог авто-закрывать задачу.

### Коммит 5 — UI: `/ideas` view (assistant-ui)

**Файлы:**
- `assistant-ui/app/templates/ideas.html` (новый, extends `base.html`) — форма capture, фильтры (`lifecycle`, `subtype`, search), список карточек с кнопками «promote» и «export to Obsidian» (последняя disabled до коммита 6).
- `assistant-ui/app/dashboard.py` — `ideas_view_model(memory_items, filters) -> dict`. Проекция полей.
- `assistant-ui/app/main.py`:
  - `GET /ideas` (HTMLResponse). Логика выбора source-вызова:
    - Если query-param `q` пуст → `clients.call_memory("search_by_metadata", {"filters":{"type":"idea", ...metadata_filters}})` (точный exact-фильтр через JSON1 индексы из коммита 1).
    - Если `q` задан → `clients.call_memory("search_memory", {query: q, kind:"note", limit:50})` (hybrid FTS+embedding по тексту) + post-hoc Python filter через **безопасный dict-доступ**: `(r.get("metadata") or {}).get("type") == "idea"` и аналогично для прочих фильтров `(r.get("metadata") or {}).get(<key>) == <value>`. Результаты `search_memory` приходят dict-shaped, среди них могут быть legacy/general note без `metadata.type` или с `metadata=None` — атрибутный доступ или прямой ключ упадёт; `.get(... or {})` защищает. Это даёт семантический поиск по тексту идеи; точный exact-фильтр недоступен в `search_memory`, поэтому идём post-hoc для прочих metadata-полей.
    - В обоих случаях прогон через `ideas_view_model` (фильтры, sort, проекция).
  - `GET /api/ideas` (JSON). Та же логика.
- `assistant-ui/app/security.py` — добавить `/ideas` в protected paths (`security.py:125`).
- HTML-форма capture/promote шлёт XHR `fetch` с заголовком `X-CSRF-Token` (CSRF middleware требует его для write methods, `security.py:109`); не plain form POST.
- Линк «идеи» в общий nav (`templates/base.html` или `templates/overview.html`).
- `assistant-ui/tests/test_ideas_view.py` — мок-тесты маршрутов + проверка protected-path / CSRF + **тест безопасности post-hoc фильтра**: `search_memory` возвращает смесь записей (idea, general note без `type`, запись с `metadata=None`); `GET /ideas?q=...` не падает (`KeyError`/`AttributeError`), в ответ попадают только записи с `metadata.type == "idea"`.

**Acceptance:**
- `/ideas` рендерит идеи через точный `search_by_metadata`.
- Фильтры `?lifecycle=incubating&subtype=concept&q=...` работают.
- Кнопка «promote» (XHR с CSRF) → `/api/ideas/{id}/promote` → redirect на `/work-item/{new_task_id}`.
- Форма capture (XHR с CSRF) → `/api/ideas/capture` → перезагрузка списка.
- Anonymous запрос на `/ideas` → редирект на login.
- POST без CSRF → 403.

### Коммит 6 — Obsidian export (on-demand)

**Файлы:**
- `assistant-ui/app/obsidian.py` (новый модуль) — функции:
  - `export_idea_to_obsidian(clients, *, idea_id, vault_path: Path) -> dict`:
    1. Lookup через `clients.call_memory("search_by_metadata", {"filters":{"type":"idea","idea_id":idea_id}})`.
    2. Если `metadata.obsidian_path` уже задан и файл существует → 409 «уже экспортирована».
    3. Безопасный slug: lowercase ASCII fallback, фильтр Windows reserved (`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`), длина ≤ 80, запрет `..` / `/` / `\\`. Имя: `idea-<short_id>-<slug>.md`.
    4. Путь: `(vault_path / "Inbox" / filename).resolve()`. Проверка `is_relative_to(vault_path.resolve())` — если нет → ValueError (path-traversal protection). Затем `(vault_path / "Inbox").mkdir(parents=True, exist_ok=True)` — каталог `Inbox/` может ещё не существовать при первом export.
    5. Frontmatter через `yaml.safe_dump(data, allow_unicode=True, sort_keys=False)` (никакого raw-string concatenation). Поля: `idea_id`, `lifecycle`, `subtype`, `source`, `created_at`, `tags`, `project`, опц. `work_item_id`. **PyYAML** добавляется в `assistant-ui/pyproject.toml` как dependency (`pyyaml>=6.0`); `uv.lock` обновляется этим же коммитом.
    6. Тело: текст идеи + раздел `## История` (пустой шаблон) + footer-ссылка на `/work-item/<id>` (если promoted).
    7. Atomic write: запись в `<path>.tmp` + `os.replace(<path>.tmp, <path>)`.
    8. Запись в memory metadata-update через `clients.store_memory(supersedes_memory_id=...)` (same-identity supersede из коммита 1).
  - `update_obsidian_frontmatter(vault_path: Path, relative_path: str, updates: dict) -> bool`:
    - Тот же resolve + is_relative_to check.
    - Прочитать файл, распарсить frontmatter (`yaml.safe_load`), смерджить с updates, перезаписать через `safe_dump` + atomic write.
    - Если файла нет — warning в лог, вернуть False (fail-soft, чтобы promote не падал).
- `assistant-ui/app/config.py` (**новый файл**, в `app/` его сейчас нет) — `obsidian_vault_path: Path | None`. Fallback-цепочка env переменных:
  1. `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` — явно для assistant-ui (приоритет).
  2. `PM_MCP_OBSIDIAN_VAULT_PATH` — переиспользование существующей переменной PM-MCP (см. `pm-mcp-server/app/config.py`, `app/adapters/obsidian.py`). В норме pm-mcp и assistant-ui работают с одним vault.
  3. Иначе `None` → endpoint 503, кнопка disabled.
  
  Решение «не вводить общий `OBSIDIAN_VAULT_PATH`» избегает name-collision с другими сервисами и согласуется с существующей конвенцией префиксов.
- `assistant-ui/app/ideas.py` — расширить `_after_promote(old_entry, new_entry)`: если `obsidian_path` в metadata и `vault_path` задан, вызвать `update_obsidian_frontmatter(vault, path, {"lifecycle":"promoted","work_item_id":<id>})`.
- `assistant-ui/app/main.py` — `POST /api/ideas/{idea_id}/export-to-obsidian`.
- `assistant-ui/app/templates/ideas.html` — активировать кнопку «export to Obsidian»; если `obsidian_path` есть → показывать `obsidian://open?vault=...&file=...` (best-effort URI).
- `assistant-ui/tests/test_ideas_obsidian.py` — мок + `tmp_path`:
  - happy path,
  - path-traversal попытки (`../`, absolute) → ValueError,
  - Windows reserved names в slug → fallback,
  - YAML с `#`, `:` в work_item_id корректно сериализуются через `safe_dump`,
  - frontmatter update сохраняет тело файла нетронутым.

**Acceptance:**
- POST `/api/ideas/<uuid>/export-to-obsidian` создаёт файл с безопасным slug и валидным YAML; повторный POST → 409.
- Без `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` и `PM_MCP_OBSIDIAN_VAULT_PATH` endpoint → 503 с понятной ошибкой; кнопка disabled.
- После promote экспортированной идеи frontmatter обновлён, тело нетронуто.
- Path-traversal попытки отклонены тестом.
- Ручное переименование файла в vault → frontmatter update — fail-soft (warning).

## Verification scenario (end-to-end вручную)

1. Поднять `ai-memory` (8765), `pm-mcp-server` (8766), `assistant-ui` (8000).
2. POST `/api/ideas/capture` body `{"text":"проверить гипотезу X","subtype":"hypothesis","tags":["mvp"],"source":"manual","project":"portfolio"}` → 200 с `idea_id`.
3. `/ideas` — запись видна, `lifecycle=incubating`.
4. Задать `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` (или иметь `PM_MCP_OBSIDIAN_VAULT_PATH` для переиспользования). POST `/api/ideas/<uuid>/export-to-obsidian` → 200, файл в `<vault>/Inbox/idea-...md`, `metadata.obsidian_path` записан.
5. POST `/api/ideas/<uuid>/promote` `{"project_path":"D:\\GitHub\\AI-Assistant\\ai-memory","title":"Проверить гипотезу X"}` → 200 с `task_id="#NNN"`. Поле `priority` опущено намеренно — реальные значения enum-поля проверяются через `mcp__PM-MCP-server__list_work_items` или схему `create_task`, не выдумываются (см. root AGENTS.md, D).
6. `mcp__PM-MCP-server__get_work_item(item_id="#NNN", project_path="D:\\GitHub\\AI-Assistant\\ai-memory")` — задача в `Бэклог` (явный `project_path` обходит multi-domain ambiguity).
7. `GET /ideas` — `lifecycle=promoted` + ссылка на task; в Obsidian-нота frontmatter `lifecycle: promoted`, `work_item_id: "#NNN"`, тело нетронуто.
8. `search_by_metadata({"filters":{"type":"idea","idea_id":...}})` — только новая активная запись, старая `archived_at IS NOT NULL`.
9. Повторный POST `/api/ideas/<uuid>/promote` → 409 с `work_item_id`.
10. Anonymous GET `/ideas` → редирект на login.
11. POST без `X-CSRF-Token` → 403.
12. Стресс compensation: замокать ошибку step-3 promote → task в Бэклоге с тегом `assistant-managed-rollback`, HTTP 502 с `action_required`.

## Открытые вопросы (для пост-MVP)

1. Расширение `memory_propose` контракта для ChatGPT capture с полным metadata.
2. PM-MCP `project_path` dropdown в форме promote (через `list_projects`).
3. Resurfacing (cron / event-driven / morning_brief) — отдельный план.
4. Двусторонний Obsidian sync (watchdog + конфликт-резолюция) — отдельный план.
5. Расширение `metadata.type` whitelist на другие kinds — отдельный ADR при появлении нужды.
6. Унификация `metadata.task_id` vs `metadata.work_item_id` — отдельный план.
7. Официальный cancel-status в PM-MCP (`Отменено`/`Не актуально`) для авто-закрытия rollback-задач — отдельный ADR.
8. CLI `python -m memory.cli backfill-metadata-type` для опциональной предварительной backfill старых note-записей (не блокирует MVP).

## Критичные файлы для изменений

- `D:\GitHub\AI-Assistant\ai-memory\memory\storage.py` — same-identity supersede + embedding carry-over.
- `D:\GitHub\AI-Assistant\ai-memory\memory\search.py` — `search_by_metadata`, `get_memory_entry`.
- `D:\GitHub\AI-Assistant\ai-memory\memory\db.py` — schema_version 9 + JSON1 индексы.
- `D:\GitHub\AI-Assistant\ai-memory\memory\validation.py` — расширение `validate_metadata` (cross-field + default type).
- `D:\GitHub\AI-Assistant\ai-memory\memory\mcp_app.py` — регистрация новых MCP tools.
- `D:\GitHub\AI-Assistant\ai-memory\memory\runtime_contract.py` — обновление публичного списка.
- `D:\GitHub\AI-Assistant\ai-memory\tests\test_storage.py`, `test_search.py`, `test_db.py`, `test_validation.py`, `test_summary.py`, `test_proposals_promote.py` — расширения.
- `D:\GitHub\AI-Assistant\ai-memory\docs\CONTRACT.md` — секции про idea-convention и type для note.
- `D:\GitHub\AI-Assistant\docs\adrs\0004-idea-memory-contract.md` — новый ADR (English).
- `D:\GitHub\AI-Assistant\docs\adrs\README.md` — индекс.
- `D:\GitHub\AI-Assistant\assistant-ui\app\ideas.py` — новый модуль (capture + promote + after-hook).
- `D:\GitHub\AI-Assistant\assistant-ui\app\obsidian.py` — новый модуль (export + frontmatter updater).
- `D:\GitHub\AI-Assistant\assistant-ui\app\config.py` (**новый файл**) — `obsidian_vault_path` с fallback `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` → `PM_MCP_OBSIDIAN_VAULT_PATH`.
- `D:\GitHub\AI-Assistant\assistant-ui\pyproject.toml` + `uv.lock` — добавление `pyyaml>=6.0` (коммит 6).
- `D:\GitHub\AI-Assistant\assistant-ui\app\security.py` — `/ideas` в protected paths.
- `D:\GitHub\AI-Assistant\assistant-ui\app\main.py` — четыре новых endpoint'а + два view'а.
- `D:\GitHub\AI-Assistant\assistant-ui\app\dashboard.py` — `ideas_view_model`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\templates\ideas.html` — новый шаблон.
- `D:\GitHub\AI-Assistant\assistant-ui\app\templates\base.html` — линк в nav.
- `D:\GitHub\AI-Assistant\assistant-ui\tests\test_ideas_capture.py`, `test_ideas_promote.py`, `test_ideas_view.py`, `test_ideas_obsidian.py` — новые тесты.

## Соответствие правилам репозитория

- ADR-0004 — на English (root AGENTS.md).
- Code comments / commit messages / task statuses — на русском.
- CONTRACT.md в `ai-memory/docs/` — русскоязычный (по факту существующего файла), допустимо.
- Cross-subsystem work items: каждый коммит создаётся как **отдельная PM-MCP задача под `project_path` затронутого subsystem'а** (по правилам root AGENTS.md J.4 + K.4):
  - Коммит 1 → задача в `D:\GitHub\AI-Assistant\ai-memory` (foundation: supersede + lookup).
  - Коммит 2 → задача в `D:\GitHub\AI-Assistant\ai-memory` (contract + validator + ADR-0004, размещаемый в root `docs/adrs/`). Зависит от коммита 1 через `link_task_dependency`.
  - Коммит 3 → задача в `D:\GitHub\AI-Assistant\assistant-ui` (capture path). Зависит от коммита 2.
  - Коммит 4 → задача в `D:\GitHub\AI-Assistant\assistant-ui` (promote). Зависит от коммита 3.
  - Коммит 5 → задача в `D:\GitHub\AI-Assistant\assistant-ui` (UI /ideas). Зависит от коммита 4.
  - Коммит 6 → задача в `D:\GitHub\AI-Assistant\assistant-ui` (Obsidian export). Зависит от коммита 5.
  Связи между задачами оформляются через `mcp__PM-MCP-server__link_task_dependency` с `dependency_project_path` для cross-subsystem-связей (коммит 2→1 в рамках `ai-memory`, остальные внутри `assistant-ui`).
- Каждый коммит атомарен в `main` (J.4) и закрывает свою PM-MCP задачу.
- Никаких новых daemon'ов / процессов — изменения только в коде существующих сервисов.
- Никаких новых публичных tools в gateway — ChatGPT в MVP пользуется существующим `memory_propose` (known limitation).
- Новые MCP tools (`search_by_metadata`, `get_memory_entry`) — в ai-memory loopback, не в gateway. Пост-MVP можно прокинуть в scope `memory.read`.
