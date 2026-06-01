# DX-аудит PM-MCP task resolver и AI-memory metadata search

**Задача:** #104, follow-up #105  
**Тип:** investigation-only  
**Статус:** reviewed  
**Дата:** 2026-05-17  

---

## 1. Гэп: Resolver task by global id без эвристик / SQLite fallback

### Текущая реализация

`get_work_item(item_id, domain, project_path)` в `app/adapters/registry.py`:

1. Если `project_path` не указан — вызывает `discover_projects_with_tasks()` и сканирует **все** проекты monorepo.
2. Для каждого проекта читает все задачи (`read_project_tasks` → SQLite).
3. Фильтрует по `_matches_requested_work_item_id()` — сравнивает `id` задач с запрошенным `item_id`.
4. Если 0 находок — `work_item_not_found`. Если > 1 — `ambiguous_work_item_id`.

`_normalize_project_task_id()` (registry.py:170-175):  
```python
match = re.fullmatch(r"#?(\d+)", normalized)
return f"#{int(match.group(1)):03d}"
```

### Наблюдаемые пробелы

1. **Нет приоритизации по активности.** Если `project_path` не указан, порядок обхода проектов не гарантирует, что нужный проект будет найден первым. Чем больше проектов, тем больше latency и вероятность `ambiguous_work_item_id`.

2. **Нет прямого SQLite fallback.** `get_work_item` не умеет сделать прямой запрос `SELECT ... FROM tasks WHERE id = ?` — только через полный сканирующий проход всех проектов через registry. В `server.py` уже есть прямой доступ к SQLite через `app.task_store`, но `get_work_item` использует только registry-путь.

3. **`ambiguous_work_item_id` нельзя разрешить эвристиками.** Если в двух проектах есть `#104`, registry отдаёт ошибку, хотя очевидный кандидат — проект текущего активного workflow или последнего modified проекта.

4. **Нормализация не защищает от коллизий.** `#104` и `104` приводятся к `#104` — это корректно, но при сканировании всех проектов коллизия по номеру всё равно возникает.

### Expected behavior

- При наличии `project_path`: точный поиск в одном проекте (уже работает).
- При отсутствии `project_path`: попробовать найти по `project_tasks` домену через прямой SQL-запрос с лимитом, без полного сканирования.
- Если прямая SQL-находка одна — вернуть её.
- Если > 1 найденных по прямому SQL — применить эвристики приоритизации (последний активный workflow run, последний modified task).
- Если > 1 и эвристики не помогли — `ambiguous_work_item_id` с контекстом для клиента.
- Registry-путь остаётся как fallback для non-`project_tasks` доменов.

### Direction options (без выбора)

**Option A — Прямой SQL fallback в `get_work_item`:**
- Расширить `get_work_item` в registry.py: если `domain == "project_tasks"` и найден > 1 или 0 matches, сделать прямой SQL-запрос через `task_store.canonical_task_id()` или аналогичный.
- Требует: передать `task_store` в registry или добавить прослойку.

**Option B — Приоритизация проектов:**
- Если `project_path` не указан, упорядочивать проекты по recency (последняя изменённая задача в SQLite `task_status_history`).
- Первый match останавливает поиск.
- Требует: LEFT JOIN или агрегирующий SQL-запрос для сортировки проектов.

**Option C — Автоопределение последнего активного проекта:**
- Использовать metadata из AI-memory (`get_latest_workflow_memory` / `get_batch_run_history`) для определения проекта, в котором сейчас идёт работа.
- Если найден активный проект — добавить его `project_path` в поиск автоматически.
- Требует: MCP-вызов AI-memory из registry.

**Option D — SQLite fallback на уровне server.py:**
- В `server.py.get_work_item()`: перед вызовом `registry_get_work_item()` попробовать прямой SQL-запрос через `db_canonical_task_id()` или `get_task_record()` для `project_tasks` домена.
- Если SQL-результат однозначный — вернуть его, не трогая registry.
- Минимальные изменения, не затрагивает архитектуру registry.

---

## 2. Гэп: Metadata-aware search без дублирования slug в text

### Текущая реализация

AI-memory search (`memory/search.py`):
- **FTS5 индекс**: индексирует только колонку `text` (db.py `ensure_memory_fts`). 
- **`search_memory`**: FTS5 + LIKE fallback + embedding similarity, всё по колонке `text`.
- **`search_by_workitem`** (search.py:518-551): Читает все строки с `metadata IS NOT NULL`, парсит JSON в Python, фильтрует по `metadata.work_item_id`. SQL-запрос без FTS, без индекса по JSON-полям.
- **`get_recent_memory`**: Простой SELECT по id DESC с фильтром по project.

PM-MCP memory client (`memory_client.py`):
- **`search_project_memory`** (memory_client.py:282-314): Сначала вызывает `get_recent_memory`, если результат не пуст — вызывает `search_memory`. **Баг**: если `get_recent_memory` вернул пустой список, `search_memory` не вызывается даже при наличии query. Это делает поиск бесполезным для проектов, где нет свежих memory-записей.
- **`search_work_item_memory`** (memory_client.py:317-344): Сначала пытается `search_by_workitem`, при ошибке fallback на `search_project_memory(project, query=f"{work_item_id} task_id work item history")`.

### Наблюдаемые пробелы

1. **Metadata (JSON) не участвует в FTS5 поиске.** Поле `metadata` содержит task_id, batch_id, workflow_id, tags и т.д., но поисковый запрос по этим полям невозможен — только через `search_by_workitem` (по exact match) или `search_project_memory` (по тексту, где может быть упоминание ID).

2. **`search_by_workitem` не масштабируется.** Читает все строки с metadata, парсит JSON — O(n) по числу memory-записей. При тысячах записей latency растёт линейно.

3. **Баг в `search_project_memory`.** Если `get_recent_memory` пуст (нет недавних записей), `search_memory` полностью пропускается. Это делает семантический поиск недоступным для проектов без свежих memory-записей.

4. **Контракт metadata расходится между PM-MCP и AI-memory.** `pm-mcp-server` в `log_task_event()` пишет идентификатор задачи в `metadata.task_id`, а `ai-memory.search_by_workitem()` фильтрует только `metadata.work_item_id`. Из-за этого специализированный поиск по work item может не находить записи, созданные PM-MCP, даже если они относятся к нужной задаче.

5. **Дублирование slug в text — не подтверждено.** Проведён аудит: `_project_name()` (memory_client.py) используется только для передачи `project=` параметра в MCP-вызовы AI-memory. В `text` memory-записей проект **не дублируется**. `store_project_memory()` и `log_task_event()` не вставляют имя проекта в `text`. Проблема была потенциальной гипотезой, но по факту дублирования нет.

### Expected behavior

- **Metadata search**: возможность искать по `metadata.*` полям (work_item_id, task, batch_id) через FTS или JSON query.
- **`search_by_workitem`**: использовать согласованный контракт идентификатора work item (`work_item_id` и/или legacy `task_id`) и индекс по `metadata` (JSON1 или generated columns), а не фильтровать в Python.
- **`search_project_memory`**: не зависеть от `get_recent_memory` — вызывать `search_memory` независимо, если задан query. `get_recent_memory` должен быть только для возврата последних записей, не как gate для поиска.
- **Дублирование в text**: не требуется фикс — дублирования нет. Документировать как verified-no-gap.

### Direction options (без выбора)

**Option A — Generated column для work_item_id:**
- Добавить в SQLite `memory` таблицу generated column `searchable_work_item_id` из JSON `metadata.work_item_id`.
- Добавить в FTS5 индекс эту колонку.
- `search_by_workitem` переписывается на FTS5 запрос.
- + быстрый поиск, − миграция схемы, требует перестроения FTS.

**Option B — JSON1 `json_extract` в WHERE:**
- В `search_by_workitem` использовать `WHERE json_extract(metadata, '$.work_item_id') = ? OR json_extract(metadata, '$.task_id') = ?` вместо чтения всех строк, если `task_id` остаётся поддерживаемым legacy-полем.
- + минимальные изменения, + быстрее текущего O(n).
- − не участвует в FTS, только exact match.

**Option C — Composite FTS5 индексация метаданных:**
- Создать дополнительную колонку `searchable_metadata` (TEXT), конкатенирующую ключевые поля из metadata.
- Добавить в FTS5 `searchable_metadata` наравне с `text`.
- + FTS по metadata, − миграция, синхронизация при обновлении metadata.

**Option D — Исправить `search_project_memory` gate bug:**
- `search_project_memory` вызывает `search_memory` всегда, если задан `query`, независимо от результата `get_recent_memory`.
- + критический багфикс, − очень маленькое изменение (убрать gate).

**Option E — Согласовать metadata contract:**
- При записи из PM-MCP добавлять `metadata.work_item_id` рядом с текущим `metadata.task_id` или мигрировать контракт на единое поле.
- `search_by_workitem` должен явно учитывать выбранный контракт и иметь тест на записи, созданные `pm-mcp-server.log_task_event()`.
- + закрывает фактическую потерю результатов поиска, − требует договориться, считается ли `task_id` legacy alias.

---

## 3. Acceptance criteria (для будущей реализации)

### Resolver by global id
- [ ] `get_work_item` для `domain="project_tasks"` с `project_path=None` использует прямой SQL-запрос или эвристики перед полным сканированием registry.
- [ ] При 1 результате от прямого запроса — вернуть найденный work item без полного сканирования registry.
- [ ] При 0 результатов от прямого запроса — fallback на registry для всех доменов.
- [ ] `ambiguous_work_item_id` возвращает контекст для клиента (список проектов с task_title).
- [ ] Существующие тесты `test_task_flow.py::test_get_work_item*` проходят без изменений.
- [ ] Добавлен тест: `get_work_item("#104")` без `project_path` находит задачу в правильном проекте.

### Metadata search
- [ ] `search_by_workitem` не читает все строки с metadata — использует индекс или JSON extract.
- [ ] PM-MCP и AI-memory используют согласованный metadata contract: записи с `metadata.task_id` и/или `metadata.work_item_id` находятся через `search_by_workitem`.
- [ ] `search_project_memory` вызывает `search_memory` при наличии query, даже если `get_recent_memory` пуст.
- [ ] Существующие тесты `test_memory_client.py` и `test_task_flow.py` (части с memory) проходят без изменений.

---

## Сводка findings

| # | Гэп | Серьёзность | Подтверждён |
|---|---|---|---|
| 1.1 | Resolver без `project_path` сканирует все проекты | Medium | Да |
| 1.2 | Нет прямого SQLite fallback для `get_work_item` | Medium | Да |
| 1.3 | `ambiguous_work_item_id` без эвристик | Low | Да |
| 2.1 | Metadata не участвует в FTS поиске | Medium | Да |
| 2.2 | `search_by_workitem` O(n) по памяти | Medium | Да |
| 2.3 | `search_project_memory` gate bug (пропускает `search_memory` при пустом `get_recent`) | High | Да |
| 2.4 | Несовпадение `metadata.task_id` в PM-MCP и `metadata.work_item_id` в AI-memory | High | Да |
| 2.5 | Дублирование slug в text | None | **Не подтверждён** — дублирования нет |
