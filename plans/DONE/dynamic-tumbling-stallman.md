# AI-memory: graph read-tools (edge traversal + tag co-occurrence)

PM-MCP: #797

## Context

`ai-memory` хранит ~1550 активных записей в SQLite. Retrieval уже сильный
(hybrid semantic+recency+kind+exact, recall@5/mrr@5=1.0 после
`paraphrase-multilingual-MiniLM-L12-v2`). В `metadata` уже зашит латентный граф
связей между записями, но **нет API, чтобы ходить по этим рёбрам** — только
поиск отдельных строк (`search_memory`, `search_by_metadata`, `search_by_workitem`)
и series-истории (`get_workflow_run_history`, `get_batch_run_history`,
`get_portfolio_run_history`).

Это «Flavor A (+A.5)» из обсуждения: дешёвая, детерминированная навигация по
**уже объявленным** рёбрам, без LLM-извлечения сущностей и без graph-БД
(полноценный GraphRAG отклонён — нет доказанной боли, высокая hygiene/стоимость,
конфликт с инвариантами). Цель — закрыть единственный реальный пробел: сборку
lineage (эволюции решения через supersedes) и раскрытие digest/summary, плюс
лёгкая entity-навигация по тегам.

## Goal

Два новых **read-only** MCP-инструмента (loopback + stdio bridge):

1. `traverse_memory` — обход reference-рёбер между записями.
2. `get_tag_graph` — ко-встречаемость тегов (узлы=теги, рёбра=совместная встречаемость).

## Scope boundaries (что НЕ делаем — во избежание дублирования)

- Series/grouping-рёбра (`workflow_id`/`run_id`, `batch_id`, `portfolio_run_id`,
  `work_item_id`, произвольное metadata-поле) **уже покрыты** — не переписываем.
- PM-MCP сущности (`task`/`goal_id`/`epic_id`/`initiative_id`) только
  **выставляем как есть** в payload узла; резолв графа задач/целей остаётся в
  PM-MCP (subsystem boundary, root AGENTS.md).
- Нет LLM, нет извлечённых сущностей, нет graph-БД, нет write-путей.

## Tech-stack alignment

Без отклонений и новых bricks:
- **#3** SQLite + JSON1 — обход по существующим `metadata` id-спискам. Один
  metadata-scan в выбранном scope (**active-only при `include_archived=False`,
  active+archived при `True`** — см. Approach), затем обход по adjacency-картам в
  памяти: JSON1 скалярный индекс не покрывает membership в массиве /
  `summary_of.item_ids`, при ~1550 строках scan дёшев.
- **#5** FastMCP loopback + `runtime_contract.EXPECTED_TOOLS` + stdio-contract-check;
  read-only, loopback trust-by-default.

## Approach

### Tool 1 — `traverse_memory`

Аргументы:
- `start_id: int` (обязателен) — якорная запись.
- `edge_types: list[str] | None` — подмн. `{"supersedes","digest","summary"}`;
  default — все.
- `direction: "forward" | "backward" | "both"` (default `both`).
- `depth: int = 3` — макс. число прыжков; отдельный жёсткий cap ≤10 (это не «limit»).
- `include_archived: bool = True` — **обязательно True по умолчанию**: цепочка
  supersedes ведёт в archived-строки; без этого lineage обрывается на текущем tip.
- `limit: int = 50` — макс. число узлов через существующий `validate_limit`
  (cap `MAX_LIMIT = 100`, [config.py:16](D:\GitHub\AI-Assistant\ai-memory\memory\config.py);
  бросает ValueError при превышении, не клампит). Новый validator не нужен.

Рёбра (симметрично forward/backward):
- `supersedes`: `metadata.supersedes_memory_id` (int | list[int]). forward R→A
  (R заменил A, A архивирован); backward A→R (какая активная запись заменила A).
- `digest`: `metadata.digest_of` (list[int]). forward D→sources; backward source→D.
- `summary`: `metadata.summary_of.item_ids` (list[int]). аналогично digest.

`direction` — направление **обхода от `start_id`**, а не семантика ребра. Поле
`direction` в `edges` фиксирует, как дошли до соседа: например, семантическое
ребро supersedes — R→A, но backward-обход идёт A→R.

Алгоритм (одно сканирование, не scan-на-каждом-шаге BFS): один проход по строкам
строит forward/backward adjacency-карты — `superseded_id → [replacers]`,
`digest_source_id → [digests]`, `summary_source_id → [summaries]`; forward-рёбра
берутся из самой metadata строки. Затем BFS от `start_id` по этим картам с
visited-set (cycle-guard), cap по depth (≤10) и числу узлов.

**Archived-контракт (критично, Codex #2).** `get_memory_by_id` и
`_iter_metadata_rows` сейчас active-only (`archived_at IS NULL`,
[search.py:756](D:\GitHub\AI-Assistant\ai-memory\memory\search.py)). Для корректного
lineage при `include_archived=True`:
- root lookup должен находить и archived `start_id`;
- adjacency-scan должен включать archived-строки, иначе цепочка
  `A ← B(archived) ← C(active)` рвётся на archived-промежутке B.

Реализовать через archived-aware варианты: `_get_rows_by_ids(ids, include_archived)`
(batch `WHERE id IN (...)`) и archived-aware scan для построения карт. При
`include_archived=False` обе выборки фильтруют `archived_at IS NULL` (отдельный
тест на обрыв цепочки).

Payload узла: к стандартным колонкам **добавить `archived_at` и `archive_reason`**
(сейчас не выбираются) + `{depth, via_edges}` — чтобы было видно active tip vs
archived predecessor. Ответ object-shaped:
`{status, root_id, nodes, edges:[{from,to,type,direction}], count, truncated, filters}`,
`not_found` если `start_id` отсутствует.

Переиспользовать: `_normalize_optional_filter`, `validate_limit`, `_row_to_dict`.
Разбор id-ссылок `int|list` — локальный helper по тому же правилу, что
`validation._normalize_metadata_id_reference_field` ([validation.py:513](D:\GitHub\AI-Assistant\ai-memory\memory\validation.py))
/ `storage._extract_superseded_ids`; приватные функции других модулей напрямую не
импортировать (при реальной общей нужде — вынести публичный helper).

### Tool 2 — `get_tag_graph`

Аргументы: `tag: str` (обязателен), `project: str | None`, `agent: str | None`,
`limit: int = 20` (макс. соседей), `min_count: int = 1`.

Поведение: scan активных строк (`_iter_metadata_rows`), отобрать те, где
`metadata.tags` содержит `tag` (нормализация lower/strip как в
`metadata_tag_match_bonus`), посчитать ко-встречаемость остальных тегов, вернуть
ранжированный список `{tag, count, sample_ids}`. Чистая деривация, без LLM.
Стабильный tie-break (`count` desc, затем `tag` asc), cap на `sample_ids` (≤5),
фильтр `min_count`. Ответ: `{status, tag, neighbors, source_count, filters}`.

## Files (повторяющийся паттерн нового read-tool; образец — #590/#591, #736)

1. `memory/search.py` — `traverse_memory(...)`, `get_tag_graph(...)`, helper `_get_rows_by_ids`.
2. `memory/mcp_app.py` — **две точки**: closures + dict в `register_read_tools()`
   **и** async-прокси в `build_bridge_stdio_server()` (иначе stdio-клиенты не увидят tool — урок #736).
3. `memory/runtime_contract.py` — имена в `EXPECTED_TOOLS`.
4. `scripts/check_stdio_contract.py` — те же имена в локальном `EXPECTED_TOOLS`.
5. `memory/cli.py` — подкоманды `traverse` и `tag-graph` по образцу ветки
   `metadata-history` ([cli.py:513](D:\GitHub\AI-Assistant\ai-memory\memory\cli.py)).
6. Тесты: `tests/test_search.py` (логика — см. «Тест-кейсы» ниже),
   `tests/test_runtime_contract.py` (новые имена в контракте),
   `tests/test_server_cli.py` (CLI-ветки).
7. Доки: `ARCHITECTURE.md` (таблица MCP-инструментов + подраздел traversal/tag-graph
   + Change hazards), `docs/CONTRACT.md`, `docs/RUNTIME.md` (перечисляет публичные
   tools — иначе новый tool выпадет из runtime-доков), `README.md`; при
   необходимости список read-tools в `AGENTS.md`. **`digest_of` сейчас в данных
   есть, но в metadata-контракте описан слабее, чем `summary_of`/`supersedes_memory_id`** —
   раз tool официально поддерживает `digest`, добавить `digest_of` в контракт
   (`ARCHITECTURE.md` + `docs/CONTRACT.md`).
8. ADR (рекомендуется, lightweight, по прецеденту ADR-0004): новый **root-level**
   `docs/adrs/0010-memory-graph-read-contract.md` (текущий максимум — ADR-0009
   intent-tree; ADR лежат в корне репо, не в `ai-memory/docs/`).

## Risks / hazards

- `include_archived=True` по умолчанию + archived-aware root lookup и adjacency-scan —
  иначе supersedes-lineage молча обрывается (особенно через archived-промежуток).
  Зафиксировать в docstring и Change hazards.
- Обязательны cycle-guard (visited) + cap по depth (≤10) и числу узлов (`validate_limit` ≤100).
- Не плодить дубли series-tools — они out of scope by design.
- Контракт-дрейф: синхронно обновить `runtime_contract.EXPECTED_TOOLS`,
  `check_stdio_contract.py` **и** stdio-bridge регистрацию.
- Boundary: PM-MCP refs выставляем, не резолвим.
- `get_tag_graph`: стабильный tie-break и cap на `sample_ids`, иначе недетерминированная выдача.

## Acceptance criteria

- `traverse_memory` собирает полную supersedes-цепочку, включая archived-предков;
  cycle-safe; уважает depth/limit; `not_found` на неизвестный id.
- `get_tag_graph` возвращает ранжированные ко-теги с counts и sample_ids.
- Оба tool'а видны в `tools/list` по stdio и через daemon.
- CLI `traverse` и `tag-graph` работают.
- `uv run ruff check .` чисто; `uv run pytest` зелёные (новые тесты включены).
- Доки и контракт перечисляют новые tool'ы; `digest_of` описан в контракте.

Тест-кейсы (минимум):
- `traverse_memory`: archived `start_id`; multi-hop `A ← B(archived) ← C(active)`;
  `include_archived=False` обрывает цепочку; cycle-guard; truncation по depth и по limit;
  forward/backward/both; `not_found`.
- `get_tag_graph`: нормализация тегов lower/strip; `min_count`; стабильный tie-break;
  cap на `sample_ids`.

## Verification

```powershell
cd D:\GitHub\AI-Assistant\ai-memory
uv run ruff check .
uv run pytest tests/test_search.py tests/test_runtime_contract.py tests/test_server_cli.py
uv run python scripts/check_stdio_contract.py        # оба имени в выводе tools/list
# Живые якоря из текущей базы для ручной проверки:
uv run python -m memory.cli traverse --start-id 1443   # 1443 supersedes 1186 (archived)
uv run python -m memory.cli traverse --start-id 1495   # digest_of=[1088,...]
uv run python -m memory.cli tag-graph --tag chatgpt-write
```

## After approval (process, central-plan-workflow)

- Подтвердить, что новый tech-stack brick / hook не нужен (план уже на
  существующих #3/#5). Если реализация выявит повторяемый паттерн — предложить
  отдельно и только с подтверждением.
- Создать/обновить PM-MCP задачу (project `ai-memory`), записать `PM-MCP: #<id>` сюда.
- После выполнения — outcome в AI-memory, архитектурную причину в ADR/docs,
  затем перенос плана в `plans/DONE/`.
