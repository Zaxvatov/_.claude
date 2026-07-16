# PM-MCP legacy storage migration

PM-MCP: #1028, #1029, #1030, #1031, #1032
Planning/review task: #1026
Status: approved 2026-06-20 after Claude review; implementation tasks created
and left in `К выполнению` without execution.

Execution update 2026-06-20:

- Code/docs implementation for #1028-#1031 is complete in `pm-mcp-server`.
- Verification passed: `uv --cache-dir .uv-cache run ruff check .`;
  `uv --cache-dir .uv-cache run pytest -p no:cacheprovider` (241 passed);
  targeted migration/calendar/task suites passed; `git diff --check` reported
  only Git line-ending normalization warnings.
- Live DB migration was verified on a copy of
  `pm-mcp-server/data/pm_mcp.sqlite3`: schema v8, `related_goals` removed,
  backup created, `PRAGMA integrity_check = ok`, reconciliation report recorded
  16 legacy `related_goals` refs and 2 marker refs as unresolvable, with no
  cross-tenant rejects and no inserted supports.
- After user service restart, live `pm_mcp.sqlite3` migrated successfully:
  schema v8, `related_goals` column removed, backup
  `pm_mcp.before-legacy-storage-migration.20260620-192946.sqlite3` created,
  `PRAGMA integrity_check = ok`, reconciliation report recorded 16 legacy
  `related_goals` refs and 2 marker refs as unresolvable with no cross-tenant
  rejects and no inserted supports.
- PM-MCP tasks #1028, #1029, #1030, #1031, and #1032 are closed as `Готово`.
  Outcome is recorded in AI-memory (`#1798` plus final completion update).

## Контекст

Текущая UI-поверхность уже не показывает отдельную модель goals/tasks/calendar,
но backend всё ещё держит compatibility bridge для старых источников записи.

Фактическое состояние на 2026-06-20:

- ADR-0026 уже перевёл goal/task модель в unified WorkItem tree поверх
  `pm-mcp-server/data/pm_mcp.sqlite3`: intent-связи должны жить в
  `tasks.parent_id` и `work_item_supports`.
- В task storage ещё есть legacy поле `tasks.related_goals`, write/update
  контракт принимает `related_goals`, а runtime и registry продолжают читать
  `[goal: ...]` / `[goals: ...]` markers и metadata fallback.
- ADR-0016 оставил Google Calendar sync в отдельном `calendar.db`; это
  допустимый raw sync/cache слой, но active backend bridge всё ещё содержит
  старый calendar adapter (`app/adapters/calendar.py`), legacy env fallback
  `PM_MCP_CALENDAR_IDS` и `PM_MCP_CALENDAR_LOOKAHEAD_DAYS`.
- Store-backed календарный путь уже существует через `app/calendar_store.py` и
  `app/calendar_sync.py`: `list_calendar_events`, `list_calendar_sources`,
  `update_calendar_source_visibility`, `sync_calendar`, `store_search_events`.
- Calendar read bridge в registry сейчас фактически инертен:
  `read_calendar_work_items()` вызывается без injected `search_events` и
  возвращает пустой список. При этом `calendar_context` уже получает
  store-backed `calendar_store_search_events`, а `app/adapters/calendar.py`
  всё ещё содержит write helpers и conflict/free-busy логику для live MCP tools.
  Поэтому миграция должна отделить удаление legacy read bridge от возможного
  переноса calendar write/free-busy helpers.

Этот план не про UI-различия. Это отдельная schema/API migration, которая должна
физически убрать legacy goal/calendar различия из активного PM-MCP backend
контракта и документации.

## Цель

После миграции в runtime-коде, активной документации и тестах должен остаться
один текущий путь:

- goal-like WorkItem связи: только `parent_id` и `work_item_supports`;
- calendar read model: только store-backed путь из `calendar_store` /
  `calendar_sync`;
- legacy storage/API поля и fallback paths не участвуют в чтении, записи,
  scoring, roadmap/dashboard payloads и registry dispatch.

`calendar.db` остаётся допустимым raw Google sync store по ADR-0016, пока
отдельный ADR не решит переносить raw events, sync tokens и watch channels в
`pm_mcp.sqlite3`. В этой миграции его роль должна быть явно отделена от
legacy WorkItem bridge: `calendar.db` хранит sync state, а не альтернативную
goal/task модель.

## Ограничения

- Перед реализацией после approval создать/обновить PM-MCP work item через
  `pm-mcp-task-flow`, перевести его в `В работе` и записать `PM-MCP: #<id>` в
  этот файл.
- Соблюдать migration discipline: сначала построить и проверить новый путь,
  затем перенести данные/клиентов, затем удалить старый путь и сделать docs
  sweep.
- Не менять UI как самостоятельную цель. Assistant-UI трогать только если
  backend API removal требует убрать legacy payload/input names.
- Не вводить новый storage engine. Использовать существующие bricks:
  SQLite storage discipline, `uv` per-subsystem, Google Calendar read-only sync,
  pytest/ruff. Не приписывать WAL `pm_mcp.sqlite3`: текущий task DB connection
  включает `foreign_keys = ON`, а WAL явно включён только для `calendar.db`.
- Если понадобится единый DB вместо `calendar.db`, сначала обновить/заменить
  ADR-0016 и явно согласовать scope: это более широкая migration, чем удаление
  legacy bridge.
- До любого деструктивного изменения `pm_mcp.sqlite3` сделать timestamped backup
  runtime DB через SQLite-native backup primitive (`VACUUM INTO` или online
  backup API) и проверить `PRAGMA integrity_check = ok`. Без backup drop/rebuild
  не выполняется.
- Primary execution model для destructive `pm_mcp.sqlite3` migration: automatic
  startup migration внутри `tasks_db.initialize()` после schema bump. Backup
  создаёт сама migration routine первым действием до любой записи, затем
  backfill/rebuild выполняются в одном initialize-connection до приёма HTTP/MCP
  трафика демоном. Если destructive шаг выносится в offline script, PM-MCP HTTP
  daemon сначала останавливается через Process State Manager; ручной rebuild
  при работающем daemon запрещён.
- Rollback path: остановить PM-MCP HTTP daemon, восстановить timestamped backup
  `pm_mcp.sqlite3`, затем снова запустить daemon.
- Backfill goal links должен быть tenant-scoped: `tenant_id` сохраняется на
  каждой связи, cross-tenant links отвергаются и попадают в reconciliation
  report, а не молча нормализуются.
- `app/adapters/calendar.py` не удалять целиком, пока Stage 1 не подтвердит,
  что calendar write helpers и conflict/free-busy logic перенесены или больше
  не используются live-инструментами.

## Релевантные workflow, skills и hooks

- `central-plan-workflow` — ведение этого plan-файла, PM-MCP id после approval,
  retrospective перед закрытием и архивирование в `DONE`.
- `migration-discipline` — порядок удаления legacy storage/API path.
- `pm-mcp-task-flow` — создание, запуск и закрытие PM-MCP work item после
  утверждения плана.
- `ai-memory-recall` — обязательный свежий контекст перед продолжением работы.
- `ai-memory-capture` — запись outcome после выполнения миграции.
- Существующие hooks достаточны на старте: `context_gate`, `write_edit_guard`,
  `require_active_task`, `python_env_guard`, `plan_retrospective`,
  `plan_archive_reminder`. Новый hook не нужен до появления повторяемого
  deterministic trigger.

## Этапы

### 1. Инвентаризация legacy path

Собрать короткий evidence snapshot до изменений одним read-only script, чтобы
получить diffable before/after артефакт:

- schema/data counts:
  - `tasks.related_goals != '[]'`;
  - tasks/raw markers с `[goal: ...]` / `[goals: ...]`;
  - `work_item_supports` по tenant и количество уже нормализованных supports;
  - `calendar_sources`, `calendar_events`, `calendar_sync_state`,
    `calendar_watch_channels` в `calendar.db`;
  - наличие индексов для bounded calendar reads:
    `idx_calendar_events_source_start` / `idx_calendar_events_range`;
- runtime references:
  - `related_goals`, `set_task_related_goals`, `goal_ids`, goal marker parsing;
  - `read_calendar_work_items`, `calendar_ids`, `calendar_lookahead_days`,
    `PM_MCP_CALENDAR_IDS`, `PM_MCP_CALENDAR_LOOKAHEAD_DAYS`;
  - registry/calendar domain dispatch and recommendation calendar context;
  - calendar write/free-busy helpers:
    `create_calendar_event`, `update_calendar_event`, `delete_calendar_event`,
    conflict detection and available-window/busy-block helpers;
- active docs references in `pm-mcp-server/README.md`,
  `pm-mcp-server/ARCHITECTURE.md`, `pm-mcp-server/docs/MCP_API.md`.

Выход этапа: read-only snapshot + таблица
"legacy source -> target path -> delete condition".

Gate: если Stage 1 показывает нулевые counts по `tasks.related_goals`,
goal-marker links и другим legacy goal-link источникам, Stage 3 data backfill
фиксируется как no-op. В этом случае остаются schema bump, column drop,
cleanup кода/docs и проверки, а тяжёлая backfill-ветка не выполняется.

### 2. Зафиксировать целевой API/schema contract

Добавить ADR или обновить существующие ADR, если изменение укладывается в них:

- ADR-0026 extension: `related_goals` и text markers являются только migration
  input, не active runtime contract.
- ADR-0016 clarification: `calendar.db` является raw sync/cache store; active
  PM-MCP calendar read API строится только через `calendar_sync` /
  `calendar_store`.
- Calendar read/context contract должен принимать или выводить явное bounded
  окно `[time_min, time_max]`; default window фиксируется в API contract до
  удаления `calendar_lookahead_days`.
- Legacy calendar env vars после удаления не должны молча игнорироваться:
  migration release должен давать fail-closed config error, чтобы stale
  prod-config был виден и не создавал скрытый compatibility layer.
- Если принимается решение переносить calendar raw state в `pm_mcp.sqlite3`,
  оформить новый ADR и отдельный data migration для `calendar_sources`,
  `calendar_events`, `calendar_sync_state`, `calendar_watch_channels`.

Выход этапа: утверждённый target contract и список breaking/compatible API
изменений.

### 3. Миграция goal legacy storage

Реализовать schema migration для `pm_mcp.sqlite3`:

- перед изменением runtime DB сделать backup через `VACUUM INTO`
  `data/pm_mcp.before-legacy-storage-migration.<timestamp>.sqlite3` или SQLite
  online-backup API и проверить, что backup открывается с
  `PRAGMA integrity_check = ok`;
- перед удалением перенести все `related_goals` в `work_item_supports`, если
  такие строки ещё есть;
- для marker-based связей `[goal: ...]` / `[goals: ...]` выполнить один
  explicit backfill в `work_item_supports` или зафиксировать, что live data уже
  пустая;
- backfill выполняется одной транзакцией, tenant-scoped, идемпотентно
  (`INSERT OR IGNORE` / существующий unique key) и с reconciliation report:
  исходные legacy links, вставленные supports, skipped duplicates,
  rejected cross-tenant links, unresolvable marker targets;
- migration report/audit должен отмечать источник backfill
  (`legacy-related-goals` / `legacy-goal-marker`) без перезаписи существующей
  истории задач;
- удалить active write/update поддержку `related_goals`;
- убрать `set_task_related_goals` как runtime alias либо оставить только
  migration-only helper вне MCP/API контракта;
- в marker parser удалить только goal-ветку; parsing `[strategic]`, `[due]`,
  `[epic]` и других non-goal markers сохранить;
- обновить `WorkItem`/registry/scoring так, чтобы strategic alignment считался
  только по `parent_id`, `supports` и нормализованным WorkItem relationships.

Удалять колонку `tasks.related_goals` только после idempotent migration test и
проверки live counts. Primary path для удаления колонки — существующий
versioned table-rebuild pattern в `tasks_db.py` (`_copy_legacy_tasks` /
schema bump), с `related_goals` исключённым из новой схемы и INSERT. Прямой
`ALTER TABLE ... DROP COLUMN` не использовать как основной путь, чтобы не
расходиться с текущей дисциплиной миграций и сохранить явный контроль над
`global_task_number`, `tenant_id`, `parent_id`, `work_item_supports`,
history/dependencies, metadata и audit history.

### 4. Миграция calendar backend bridge

Перевести calendar domain на store-backed path:

- registry calendar read сейчас no-op; удалить этот inert dispatch либо, если
  calendar WorkItems всё ещё нужны как multi-domain registry output, заменить
  его на `calendar_sync.list_calendar_events` / store-backed projection с явным
  bounded window вместо старого injected `search_events` adapter;
- перенести `calendar_context` из legacy adapter или переписать его как
  store-backed helper, чтобы recommendation scoring больше не зависел от
  `app/adapters/calendar.py`;
- подтвердить через `EXPLAIN QUERY PLAN` или targeted test, что bounded
  calendar context использует существующие индексы
  `idx_calendar_events_source_start` / `idx_calendar_events_range`, а не делает
  полный scan calendar table;
- удалить `PM_MCP_CALENDAR_IDS` и `PM_MCP_CALENDAR_LOOKAHEAD_DAYS` из active
  config/docs/tests; `PM_MCP_CALENDAR_SELECTED_IDS` остаётся sync allowlist;
- при наличии legacy env vars после удаления выдавать fail-closed config error,
  а не silently ignore;
- сохранить `calendar.db` schema v4 как ADR-0016 sync/cache store, если не
  утверждён отдельный single-DB ADR;
- сохранить `app/adapters/calendar.py` только для write helpers/free-busy либо
  перенести эти функции в новый store-backed модуль до удаления adapter import;
- проверить, что `list_calendar_events`, roadmap calendar payload и policy
  scoring читают один и тот же store-backed источник.

### 5. Client/API cleanup

Обновить клиентов и docs под новый contract:

- Assistant-UI goal create/update должен писать `parent_id` и
  `set_work_item_supports`, а не `related_goals`;
- roadmap/dashboard могут держать derived `goal_ids` только как view-model,
  если они вычисляются из `parent_id`/`supports`, а не из legacy storage;
- `pm-mcp-server/docs/MCP_API.md`, `README.md`, `ARCHITECTURE.md` должны
  описывать только текущий путь; legacy упоминания допустимы только в
  исторических ADR или migration notes.

### 6. Тесты и live verification

Минимальные проверки:

- `pm-mcp-server`: `uv --cache-dir .uv-cache run ruff check .`;
- iteration loop: targeted pytest first:
  `tests/test_tasks_db.py`, `tests/test_task_flow.py`,
  `tests/test_calendar_*.py`, `tests/test_unified_tree_import.py`
  with `-p no:cacheprovider`;
- final gate: `pm-mcp-server`: `uv --cache-dir .uv-cache run pytest -p no:cacheprovider`;
- targeted tests:
  - schema migration from DB with `related_goals`;
  - tenant-scoped backfill and cross-tenant rejection;
  - idempotent migration replay does not duplicate `work_item_supports`;
  - no registry/scoring dependency on goal markers;
  - non-goal marker parser still preserves `[strategic]`, `[due]`, `[epic]`;
  - calendar config no longer accepts legacy fallback;
  - calendar domain and policy context read from store-backed events;
  - calendar write/free-busy helpers still work or are explicitly removed from
    the public MCP contract;
  - `list_goals`, `get_goal_tree`, `get_goal_decomposition`,
    `list_calendar_events`, `list_calendar_sources` contracts.
- если Assistant-UI touched:
  - relevant Assistant-UI pytest/API tests;
  - live `/api/roadmap` or browser smoke check for calendar/goal payloads.

Read-only live checks before close:

- `list_goals(status="active")` returns expected active goal-like WorkItems;
- `get_goal_decomposition` for the life root returns no missing-link diagnostics;
- `list_calendar_sources` and `list_calendar_events` return store-backed data
  or a structured empty/unavailable payload;
- before/after snapshot reconciliation shows legacy goal-link counts accounted
  for by normalized supports, explicit skips, rejected cross-tenant links or
  unresolvable marker targets;
- `rg` over active code/docs shows no runtime references to removed legacy
  paths, excluding historical ADR and migration test fixtures.
- `git diff --check` passes.

## Риски

- Dropping `tasks.related_goals` can lose historical intent links if backfill is
  incomplete. Mitigation: count, backfill, compare supports, then drop.
- Destructive schema migration can corrupt the live SQLite DB if interrupted.
  Mitigation: timestamped SQLite-native backup with `integrity_check`, one
  transaction for backfill/schema bump, startup execution before daemon traffic,
  and rollback by restoring the backup before restarting the daemon.
- Running table-rebuild against a live task DB can hit `database is locked` or
  expose transient `tasks_legacy` state, especially if WAL is enabled later.
  Mitigation: primary path is startup migration inside one initialize connection;
  offline execution requires stopping PM-MCP HTTP through Process State Manager
  first.
- Backfill can break tenant isolation if links are resolved globally without
  `tenant_id`. Mitigation: tenant-scoped queries, cross-tenant rejection and a
  regression test.
- Calendar registry needs an explicit time window; the old adapter implied a
  lookahead default. Mitigation: define default window in API contract before
  deleting `calendar_lookahead_days`.
- Store-backed calendar context can regress performance if it scans all
  `calendar_events`. Mitigation: bounded window and index/plan verification.
- Full single-DB calendar migration would touch Google sync tokens and watch
  channel state. Mitigation: keep `calendar.db` as ADR-0016 raw sync store unless
  a separate ADR approves the larger move.
- Assistant-UI may still use `goal_ids` as view-model. Mitigation: allow derived
  field in UI payloads only after confirming it is not a storage/write source.
- Calendar adapter also owns write helpers and conflict/free-busy math.
  Mitigation: inventory and preserve/move that surface before deleting imports
  or module files.

## Acceptance criteria

- No active PM-MCP runtime path reads goal links from `tasks.related_goals`,
  `[goal: ...]` markers or goal metadata fallback.
- Before/after reconciliation is numeric: every legacy goal link is represented
  by `work_item_supports`, skipped as duplicate, or listed as rejected with a
  tenant/scope/unresolvable-target reason.
- `tasks.related_goals` column is removed only after backup + migration replay
  test + live count reconciliation. Backup must be created by the migration
  routine itself before the first write in startup execution, or by the offline
  script after stopping the daemon through Process State Manager; backup uses
  `VACUUM INTO` or SQLite online-backup API and passes
  `PRAGMA integrity_check = ok`.
- No active PM-MCP runtime path uses `PM_MCP_CALENDAR_IDS`,
  `PM_MCP_CALENDAR_LOOKAHEAD_DAYS` or `read_calendar_work_items`.
- Calendar write/free-busy MCP behavior is either preserved with tests or
  explicitly removed from the public contract in ADR/docs before code deletion.
- Goal write path is exclusively `create_task` / `update_task` with
  `parent_id`/`horizon` plus `set_work_item_parent` and
  `set_work_item_supports`.
- Calendar roadmap/scoring/read tools use one store-backed source:
  `calendar_store` / `calendar_sync`.
- Current docs describe only the post-migration contract.
- All required subsystem checks pass, or any skipped check is explicitly
  documented with the reason.

## Pre-close retrospective placeholder

Заполнить перед закрытием PM-MCP task:

| Axis | Verdict | Notes |
| --- | --- | --- |
| `tech-stack-choices.md` | no-change | Использованы существующие bricks: SQLite schema migration, uv cache, pytest/ruff, Calendar read-only sync. Новый reusable brick не появился. |
| Design-system | no-change | UI не менялся. |
| Skills | no-change | `migration-discipline`, `central-plan-workflow`, `pm-mcp-task-flow` покрыли процесс; отдельный skill не нужен. |
| Hooks | follow-up-task | Создан follow-up PM-MCP #1033 на guard против возврата legacy storage fallback после live migration. |
