# Refactor: idea → lazy classification + Obsidian-first bidirectional sync

> Редакция v2 — учтены все 10 замечаний Codex round 1 + два финальных решения
> пользователя: гибридный gateway scope (Local direct / External propose),
> двусторонний Obsidian sync вместо resurfacing-инфраструктуры.

## Context

В плане 590 ([thought-development-system.md](DONE/thought-development-system.md))
система записи идей построена на **eager-классификации**: в момент capture
агент должен решить «это idea или general note». `capture_idea`
([assistant-ui/app/ideas.py:18-63](D:\GitHub\AI-Assistant\assistant-ui\app\ideas.py))
всегда пишет `type=idea` + `lifecycle=incubating`. Bio-style автозапись
ChatGPT идёт другим путём — через `memory_propose`
([gateway/app.py:45](D:\GitHub\AI-Assistant\gateway\gateway\app.py))
с partial metadata.

**Три разрыва:**

1. **UX-разрыв**: пользователь часто не знает в момент capture, что у него —
   идея для проработки или просто факт. Eager-tool заставляет выбрать.
2. **Архитектурный разрыв capture-путей**: ChatGPT (write) задуман как
   Bio-style автозапись, а команда «Зафиксируй» — это **отдельная** прямая
   фиксация (чаще всего — идея, но иногда заметка). Сейчас эти два пути
   используют разные tools с разной семантикой.
3. **Видимость идей**: в плане 590 Obsidian export — on-demand (решение #4
   «не засоряем vault»). Пользователь хочет инвертировать: vault = UI
   для идей, ai-memory = хранилище для семантического поиска. Тогда
   resurfacing-инфраструктура не нужна — пользователь сам видит всё в vault.

**Существующие баги плана 590**, всплывшие при review:

- [validation.py](D:\GitHub\AI-Assistant\ai-memory\memory\validation.py) не
  реализует архитектурное решение #10 плана 590 — нет default-логики для
  `type` note, нет `lifecycle`/`subtype` в whitelist. План 590 формально DONE,
  но валидатор оставляет ключевые поля как arbitrary keys.
- `assistant-ui/app/ideas.py:156` вызывает `get_memory_entry` MCP tool,
  которого **нет в ai-memory** ([search.py:616](D:\GitHub\AI-Assistant\ai-memory\memory\search.py)
  имеет internal `get_memory_by_id`, но он не зарегистрирован как MCP).
  То есть capture-логика плана 590 обращается к несуществующему tool —
  работает только потому, что happy-path не доходит до `skipped_duplicate`
  ветки. Любое тестирование dedupe → 502 lookup_failed.
- План 590 архитектурное решение #15/#20 говорит `search_by_metadata`
  с nested filters `{filters: {...}}`, но реальная реализация
  ([search.py:555](D:\GitHub\AI-Assistant\ai-memory\memory\search.py))
  flat-сигнатура `(field, value, ...)`. Контракт разъехался.
- ADR-0004 ([docs/adrs/0004-idea-memory-contract.md](D:\GitHub\AI-Assistant\docs\adrs\0004-idea-memory-contract.md))
  фиксирует **только** `metadata.type=idea` + `metadata.idea_id` как контракт.
  `lifecycle`/`subtype` — convention плана 590, **не contract**. Любая
  строгая валидация `type=idea` requires `lifecycle/subtype` противоречит
  ADR-0004.

## Цели рефактора

1. Перейти на **lazy classification** с гибридным trigger'ом (пользователь
   или агент): capture-путь единый, классификация — поздняя.
2. Перенести capture/mark-as-idea/list-логику **в ai-memory как доменные
   MCP tools** (исправление архитектурного слоя — assistant-ui становится
   thin client).
3. Сделать Obsidian **первичным UI для идей** через двусторонний sync:
   push при capture/lifecycle change, pull через watchdog при правке `.md`.
4. Починить баги плана 590, всплывшие при review.
5. **По [migration-discipline](D:\GitHub\_engineering_rules\skills\migration-discipline)** —
   удалить старый путь `/api/ideas/*` + `capture_idea` в том же коммите, где
   появляется новый. Без deprecated aliases.

## Целевая модель

| Уровень | Trigger | Tool | Семантика | Direct/Approval |
|---|---|---|---|---|
| 0. Bio-style | агент сам, без явной команды | `memory_propose` (loopback или gateway) | `note` без `type=idea`, default `general`/`summary` | Local: direct. External: proposal+approval |
| 1. Явная фиксация | «Зафиксируй это» от пользователя или агента | новый `capture_memory` MCP tool в ai-memory | `note` с опциональным `as_idea` hint; если hint=true → idea-metadata + auto-export в Obsidian | Local: direct. External: расширенный `memory_propose` с `as_idea` поддержкой → proposal+approval |
| 2. Upgrade позже | пользователь или агент перечитал note | новый `mark_as_idea` MCP tool | same-identity supersede: note → idea, auto-export в Obsidian | Local only (UI или loopback MCP); External — через proposal |
| 3. Sync обратно | пользователь правит .md в Obsidian | watchdog daemon (assistant-ui background) | парсит frontmatter+body → `store_memory` с `supersedes_memory_id` | Local-only (watchdog видит только локальный vault) |

**Кто ставит `as_idea`** (гибрид по подтверждению пользователя):
- Агент анализирует контекст разговора (фразы «у меня идея», «гипотеза», «давай подумать») и ставит автоматически.
- Пользователь может явно сказать «зафиксируй как идею» / «зафиксируй просто как заметку».
- По умолчанию (без сигнала) — `as_idea=false`.

**Obsidian как UI:** при `as_idea=true` capture автоматически создаёт
`.md` файл в `<vault>/Inbox/idea-<short_id>-<slug>.md` (плановая локация
была on-demand, теперь становится автоматической). При update lifecycle/
promote — frontmatter обновляется. Pull-направление: watchdog отслеживает
изменения `.md`, парсит frontmatter+body, обновляет ai-memory через
same-identity supersede.

## Architectural decisions

| # | Решение | Обоснование |
|---|---|---|
| 1 | Capture/mark-as-idea/list-логика **переезжает в ai-memory** как MCP tools. assistant-ui становится thin client. | Codex round 1, замечание 1: gateway знает только ai-memory/proposals/pm_mcp ([backends.py:28](D:\GitHub\AI-Assistant\gateway\gateway\backends.py)). Проксирование gateway→assistant-ui ломает архитектуру |
| 2 | Gateway scope: **Hybrid (вариант C)**. Local агенты (loopback) пишут напрямую через `capture_memory` MCP. External (ChatGPT через gateway) идёт через `memory_propose` с расширенным контрактом (поддержка `as_idea` hint) + approval flow в UI. | Сохраняет ADR-0001 D-3 («Proposal queue остаётся внутри ai-memory. Gateway проксирует propose_memory, не имеет собственного staging»). Не вводит новый scope `memory.capture`, не требует изменения OAuth metadata / DCR allowlist / DENIED_TOOLS ([scope_policy.py:18](D:\GitHub\AI-Assistant\gateway\gateway\scope_policy.py)). Local trust-by-default остаётся (кирпичик #5) |
| 3 | `lifecycle`/`subtype`/`obsidian_path` — **convention с нормализацией**, НЕ required для `type=idea` | Codex round 1, замечание 5: противоречие с ADR-0004 (требует только `metadata.idea_id`). Existing идеи в БД остаются валидными. Defaults ставит capture-path в ai-memory, не валидатор |
| 4 | `as_idea` — гибридный bool hint, по умолчанию false | Lazy classification по подтверждению пользователя |
| 5 | Upgrade note → idea через отдельный `mark_as_idea` MCP, не через повторный capture | Семантика разная: capture создаёт, mark-as-idea меняет lifecycle. Использует same-identity supersede плана 590 коммит 1 |
| 6 | **Resurfacing-инфраструктура (`forgotten_memory`/`morning_brief` block/cron) не вводится** в этом плане. Obsidian-first видимость заменяет её. | Решение пользователя: «было бы лучше, если бы я видел все такие идеи в Obsidian, тогда никаких напоминаний не нужно». Resurfacing — отдельный план если когда-нибудь понадобится |
| 7 | `next_review_at`/`importance` поля — **НЕ вводятся** в этом плане | Резарфейс отложен; поля без applicability не нужны. Если понадобятся — добавятся отдельным планом |
| 8 | **Obsidian двусторонний sync** в этом плане: push (existing on-demand `export_idea_to_obsidian` плана 590 коммит 6 становится auto) + pull (новый watchdog daemon в assistant-ui background task) | Решение пользователя: «двусторонний в этом плане» |
| 9 | Watchdog — **background task в assistant-ui**, не отдельный NSSM-сервис | ADR-0001 D-8 (минимизация daemon'ов). Singleton-guard (кирпичик #8) если в будущем выделится |
| 10 | Конфликт-резолюция: **last-write-wins по mtime + loop prevention через short-term cache** «(path, mtime, source)» | Простая семантика, single-user окружение. Conflict-prone случаи (одновременный agent-update + Obsidian-edit) — редки; LWW acceptable |
| 11 | UI `/ideas` URL → 302 redirect на `/memory?type=idea` (user-friendly URL, не legacy) | Inertia для существующих закладок |
| 12 | `/memory` list через **существующий `get_recent_memory`** + новый `list_memory` с фильтрами `(type, lifecycle, project, agent, kind, limit, offset)` | Codex round 1, замечание 7: `search_by_metadata(field, value)` flat-сигнатура не подходит для общего list. `get_recent_memory` уже есть ([pm-mcp-server/app/memory_client.py:292](D:\GitHub\AI-Assistant\pm-mcp-server\app\memory_client.py)), переиспользуем |
| 13 | promote/export endpoints мигрируют с `/api/ideas/{idea_id}/*` на `/api/memory/{memory_id}/*` | Унификация namespace. Lookup по `memory_id` с проверкой `type=idea` внутри (422 `not_an_idea` если нет). migration-discipline — старые URL удаляются |
| 14 | PM-MCP integration через существующий `pm-mcp-server/app/memory_client.py` (`_call_memory_tool`) — **никаких новых HTTP-клиентов** | Codex round 1, замечание 9 |

## План на 5 коммитов

### Коммит 1 — ADR-0005: lazy idea classification + hybrid capture policy

**Цель**: зафиксировать архитектурные решения **до** кода, чтобы коммиты
2-5 имели контракт-якорь.

**Файлы:**
- `D:\GitHub\AI-Assistant\docs\adrs\0005-lazy-idea-classification.md` (новый, **English** по root AGENTS.md):
  - **Status**: accepted
  - **Date**: 2026-05-24
  - **Related ADRs**: ADR-0001 (no direct write через gateway — сохраняем), ADR-0004 (idea contract — расширяем без breaking changes)
  - **Context**: 3 разрыва из секции Context этого плана + баги.
  - **Decision**:
    1. Lazy classification convention: `as_idea` hint при capture, default false.
    2. Hybrid capture policy: loopback → direct `capture_memory` MCP; gateway → расширенный `memory_propose` с `as_idea` поддержкой + approval flow в UI.
    3. ADR-0004 compatibility: `lifecycle`/`subtype` остаются convention, не contract. Только `metadata.type=idea` + `metadata.idea_id` required.
    4. capture/mark-as-idea/list-логика в ai-memory как доменный MCP API; assistant-ui = thin client.
    5. Obsidian двусторонний sync: push (auto при capture/promote), pull (watchdog daemon в assistant-ui).
    6. Resurfacing-инфраструктура — out of scope (Obsidian-first видимость заменяет).
  - **Consequences**:
    - Gateway scope tree остаётся unchanged.
    - assistant-ui упрощается до UI + HTTP-обёртки над MCP.
    - Vault становится источником истины для content идей (text + frontmatter metadata); ai-memory — индекс + cross-agent context.
    - LWW конфликт-резолюция между Obsidian-edits и agent-updates.
  - **Alternatives considered**:
    - A) Новый scope `memory.capture` для direct write через gateway. Отклонено: расширяет security surface (ADR-0001 D-3 нарушается), требует OAuth metadata + DCR allowlist + DENIED_TOOLS-обновление.
    - B) Proposal-only для всех (включая локальные). Отклонено: ломает UX «Зафиксируй» для локальных Claude/Codex.
- `D:\GitHub\AI-Assistant\docs\adrs\README.md` — добавить ссылку на ADR-0005.

**Acceptance:**
- ADR-0005 прошёл grammar/structure check (Markdown lint, заголовки в порядке ADR-0001/0002/0003/0004).
- README обновлён.
- Никаких изменений кода в этом коммите (чистый ADR-коммит).

### Коммит 2 — ai-memory: capture domain MCP API + validation fixes

**Цель**: перенести capture/mark/list-логику в ai-memory как доменные
MCP tools. Починить bagги плана 590.

**Файлы:**

- [ai-memory/memory/search.py](D:\GitHub\AI-Assistant\ai-memory\memory\search.py):
  - Новая функция `get_memory_entry(memory_id: int, *, include_archived: bool = False) -> dict`: обёртка над `get_memory_by_id` (existing line 616). Возвращает object-shaped `{"status":"ok","item":{...}}` или `{"status":"not_found"}`. По умолчанию active-only; archived только при явном флаге.
  - Новая функция `list_memory(*, type_: str | None = None, lifecycle: str | None = None, project: str | None = None, agent: str | None = None, kind: str | None = None, limit: int = 50, offset: int = 0) -> dict`: возвращает object-shaped `{"status":"ok","items":[...],"count":N,"filters":{...},"limit":L,"offset":O}`. Sort по `created_at desc`. Использует JSON1 индексы (см. ниже).
  - **Не трогаем** `search_by_metadata` (flat-сигнатура остаётся; ADR-0004 совместимо).

- [ai-memory/memory/storage.py](D:\GitHub\AI-Assistant\ai-memory\memory\storage.py):
  - Новая функция `capture_memory(*, text, project, agent, as_idea=False, subtype=None, source=None, tags=None, goal_id=None) -> dict`:
    1. Metadata builder: `as_idea=False` → `{type:"general", source, tags}`. `as_idea=True` → `{type:"idea", idea_id:uuid4, subtype:subtype or "concept", lifecycle:"incubating", source, tags, goal_id}`.
    2. Вызов `store_memory(text, project, agent, kind="note", metadata)`.
    3. Если `ok` → возврат `{ok:true, idea_id (если as_idea), memory_id, deduplicated:false}`.
    4. Если `skipped_duplicate` с existing_id → `get_memory_by_id(existing_id)`:
       - existing `kind=="note"` + `metadata.type=="idea"` → идемпотентность: возврат existing `idea_id` + `deduplicated:true`.
       - existing `kind=="note"` + `type!="idea"` + `as_idea=True` → повторный `store_memory` с `supersedes_memory_id=existing_id` (same-identity supersede плана 590 коммит 1). Возврат `upgraded_from:existing_type`.
       - existing `kind=="note"` + `type!="idea"` + `as_idea=False` → возврат existing id без upgrade.
  - Новая функция `mark_as_idea(*, memory_id: int, subtype: str = "concept", lifecycle: str = "incubating") -> dict`:
    1. `get_memory_by_id(memory_id)` → если не найден или archived → 404.
    2. Если уже `type=idea` → 409 с existing `idea_id`.
    3. Same-identity supersede: `store_memory(text=existing.text, project=existing.project, agent=existing.agent, kind=existing.kind, metadata={...existing.metadata, type:"idea", lifecycle, subtype, idea_id:uuid4, supersedes_memory_id:memory_id})`.
    4. Возврат `{ok:true, idea_id, memory_id:new_id, archived_id:memory_id}`.

- [ai-memory/memory/validation.py](D:\GitHub\AI-Assistant\ai-memory\memory\validation.py):
  - Добавить в `RECOMMENDED_METADATA_STRING_FIELDS` (line 9-30): `lifecycle`, `subtype`, `obsidian_path` (нормализация trim/strip).
  - Добавить в `RECOMMENDED_METADATA_CANONICAL_STRING_FIELDS` (line 34-40): `lifecycle`, `subtype` (lowercase нормализация).
  - **Soft whitelist** (warning, не ValueError) для `lifecycle ∈ {"incubating","developing","mature","dormant","promoted","archived"}` и `subtype ∈ {"concept","hypothesis","prompt","research","decision-draft","goal-seed"}`. Soft потому что ADR-0004 не требует этих значений; план 590 ввёл convention.
  - Default-логика для note `type` (решение #10 плана 590, не реализованное):
    - Опциональный `kind`: `validate_metadata(metadata, *, kind=None)`.
    - `kind=="note"` + `type` отсутствует → default: `summary` если `summary_of`, иначе `general`.
    - `kind=="note"` + `type` указан → whitelist `{"idea","general","summary","audit-finding"}`.
  - Все вызовы `validate_metadata` (`mcp_app.py`, `storage.py`, `proposals/*`, `search.py:store_summary`) пробросить `kind`.

- [ai-memory/memory/db.py](D:\GitHub\AI-Assistant\ai-memory\memory\db.py):
  - schema_version 8 → 9. JSON1 partial индексы:
    - `idx_memory_metadata_type` на `json_extract(metadata,'$.type')` WHERE `archived_at IS NULL`.
    - `idx_memory_metadata_lifecycle` на `json_extract(metadata,'$.lifecycle')` WHERE `archived_at IS NULL`.
    - `idx_memory_metadata_idea_id` на `json_extract(metadata,'$.idea_id')` WHERE `archived_at IS NULL`.
    - `idx_memory_metadata_work_item_id` на `json_extract(metadata,'$.work_item_id')` WHERE `archived_at IS NULL`.

- [ai-memory/memory/mcp_app.py](D:\GitHub\AI-Assistant\ai-memory\memory\mcp_app.py):
  - Зарегистрировать новые MCP tools: `capture_memory`, `mark_as_idea`, `get_memory_entry`, `list_memory`. Schema по аналогии с существующими.

- [ai-memory/memory/runtime_contract.py](D:\GitHub\AI-Assistant\ai-memory\memory\runtime_contract.py):
  - Расширить публичный список tools, если есть invariant.

**Тесты** (расширения / новые):
- `ai-memory/tests/test_capture.py` (новый):
  - happy `as_idea=False` → type=general, no idea_id.
  - happy `as_idea=True` → type=idea, idea_id.
  - идемпотентность повторного capture(as_idea=True) того же текста → тот же idea_id.
  - upgrade general → idea (skipped_duplicate path).
  - capture(as_idea=False) на тексте уже-idea → возврат existing memory_id, без upgrade.
- `ai-memory/tests/test_mark_as_idea.py` (новый): happy, 404 на отсутствующем, 409 на уже-idea.
- `ai-memory/tests/test_get_memory_entry.py` (новый): active возвращается; archived → not_found; с include_archived → возвращается.
- `ai-memory/tests/test_list_memory.py` (новый): фильтры (type, lifecycle, project, agent, kind), pagination через offset, sort по created_at desc.
- `ai-memory/tests/test_validation.py` (расширение):
  - `validate_metadata({}, kind="note")` → type=general.
  - `validate_metadata({"summary_of":...}, kind="note")` → type=summary.
  - `validate_metadata({"type":"idea"}, kind="note")` → проходит (only idea_id required по ADR-0004); запись валидна.
  - `validate_metadata({"lifecycle":"unknown"}, kind="note")` → warning (через `logger.warning`), не raises.
- `ai-memory/tests/test_db.py` (расширение): миграция 8→9 идемпотентна, индексы созданы.

**Acceptance:**
- Все существующие `note`-writer'ы (`store_summary`, proposal `approve`, batch) работают без изменений.
- `capture_memory` MCP вызывается через ai-memory loopback.
- `get_memory_entry`/`list_memory` зарегистрированы как MCP tools.
- `EXPLAIN QUERY PLAN` для `search_by_metadata(field="idea_id", value=...)` показывает использование `idx_memory_metadata_idea_id`.
- `uv run pytest && uv run ruff check .` зелёные в `ai-memory/`.

### Коммит 3 — assistant-ui: thin client + URL миграция + UI /memory

**Цель**: переписать `capture_idea` в thin client поверх ai-memory MCP.
Удалить `/api/ideas/*`. Сделать UI `/memory` с capture-формой.

**Файлы:**

- [assistant-ui/app/ideas.py](D:\GitHub\AI-Assistant\assistant-ui\app\ideas.py) → **переименовать в `memory_capture.py`** (НЕ `memory.py` — последний занят chat-context [memory.py:1](D:\GitHub\AI-Assistant\assistant-ui\app\memory.py), Codex round 1 замечание 4):
  - `capture_idea` → **удаляется**.
  - Новая `capture_memory(clients, *, text, project, agent, as_idea=False, ...) -> dict`: один MCP вызов `clients.call_memory("capture_memory", {...})`. Возврат проксируется.
  - Новая `mark_as_idea(clients, *, memory_id, ...) -> dict`: один MCP вызов.
  - `promote_idea` → `promote_memory(clients, *, memory_id, project_path, title, ...) -> dict`:
    1. `clients.call_memory("get_memory_entry", {"memory_id": memory_id})`.
    2. Проверка `metadata.type=="idea"` → если нет → 422 `not_an_idea`.
    3. Логика создания PM-MCP задачи + same-identity supersede memory (как в [ideas.py:66-116](D:\GitHub\AI-Assistant\assistant-ui\app\ideas.py)) без изменений.
  - `find_idea` → `find_memory_by_idea_id(clients, idea_id)`: оставляется (используется в Obsidian export для legacy lookup), но переписан на `search_by_metadata(field="idea_id", value=idea_id)` (flat-сигнатура, по факту реальной).
  - **Удаление**: `_get_memory_entry` helper (line 156) больше не нужен — `get_memory_entry` теперь полноценный MCP tool. Прямой `clients.call_memory("get_memory_entry", ...)`.

- [assistant-ui/app/main.py](D:\GitHub\AI-Assistant\assistant-ui\app\main.py):
  - **Удалить** старые routes: `POST /api/ideas/capture`, `POST /api/ideas/{idea_id}/promote`, `POST /api/ideas/{idea_id}/export-to-obsidian`.
  - **Добавить**:
    - `POST /api/memory/capture` (Pydantic `CaptureMemoryRequest` с `as_idea: bool = False`).
    - `POST /api/memory/{memory_id}/mark-as-idea` (Pydantic `MarkAsIdeaRequest`).
    - `POST /api/memory/{memory_id}/promote` (Pydantic `PromoteMemoryRequest`).
    - `POST /api/memory/{memory_id}/export-to-obsidian` (manual trigger; auto-export — в C5).
    - `GET /memory` (HTMLResponse, общий list через `list_memory` MCP).
    - `GET /api/memory` (JSON-вариант).
    - `GET /memory/{memory_id}` (HTMLResponse, detail).
    - `GET /api/memory/{memory_id}` (JSON detail через `get_memory_entry`).
    - `GET /ideas` → 302 на `/memory?type=idea`.
    - `GET /api/ideas` → 302 на `/api/memory?type=idea`.

- [assistant-ui/app/security.py](D:\GitHub\AI-Assistant\assistant-ui\app\security.py) line 125: protected paths обновить:
  - Удалить `/api/ideas/*`.
  - Добавить `/api/memory/*`, `/memory`, `/memory/*`.

- [assistant-ui/app/obsidian.py](D:\GitHub\AI-Assistant\assistant-ui\app\obsidian.py): сигнатура принимает `memory_id`, lookup через `get_memory_entry` MCP. Проверка `metadata.type==idea` остаётся.

- [assistant-ui/app/dashboard.py](D:\GitHub\AI-Assistant\assistant-ui\app\dashboard.py): `ideas_view_model` → `memory_view_model`. Принимает результат `list_memory` MCP, формирует view-данные.

- `assistant-ui/app/templates/memory.html` (новый, extends `base.html`):
  - Capture-форма: textarea + checkbox «зафиксировать как идею для развития» + (если checked) subtype dropdown + tags input.
  - Фильтры: type, lifecycle, project (inline selector через `list_projects` PM-MCP MCP — НЕ `prompt("project_path")` как в коде плана 590), agent, search `q`.
  - List карточек с pagination (limit/offset), empty state, error state, loading state.
  - Actions на карточке: `mark_as_idea` (если type≠idea), `promote` (если type=idea), `open in Obsidian` (если obsidian_path).
  - XHR fetch с `X-CSRF-Token` header.

- `assistant-ui/app/templates/ideas.html` — **удаляется** (заменяется 302 redirect на `/memory?type=idea`).

- `assistant-ui/app/templates/base.html`: nav-линк на `/memory` (опционально shortcut «идеи» в submenu).

- Design-system compliance: использовать существующие Material Web Components и tokens из общего Design-system (`D:/GitHub/Design-system`), без новых локальных CSS. Если требуется — подключение через [design-system-integration](D:\GitHub\_engineering_rules\skills\design-system-integration) skill.

**Тесты** (заменяют старые `test_ideas_*.py`):
- `assistant-ui/tests/test_memory_capture.py`:
  - happy `as_idea=false` → type=general.
  - happy `as_idea=true` → type=idea, idea_id.
  - идемпотентность через ai-memory MCP мок.
  - 404 на старых `/api/ideas/*` URL (миграция гарантирована).
- `assistant-ui/tests/test_memory_mark_as_idea.py`: happy, 409 на уже-idea.
- `assistant-ui/tests/test_memory_promote.py`: happy, 422 not_an_idea, 409 already_promoted, compensation.
- `assistant-ui/tests/test_memory_obsidian.py`: happy, path-traversal protection, frontmatter update (manual trigger — auto в C5).
- `assistant-ui/tests/test_memory_view.py`:
  - `GET /memory` рендерит list (мок `list_memory`).
  - `GET /memory?type=idea` фильтрует.
  - `GET /ideas` → 302.
  - Capture-форма (XHR с CSRF) → MCP call.
  - Empty state, error state.
  - post-hoc safety: мик результатов с `metadata=None`/без ключей → не падает.
  - Anonymous request → redirect на login.
  - POST без CSRF → 403.

**Acceptance:**
- `curl POST /api/ideas/capture` → 404.
- `curl POST /api/memory/capture` body `{text,as_idea:true}` → 200, запись через MCP.
- `/memory` рендерит общий list с pagination, empty state, inline project selector.
- `/ideas` → 302.
- `uv run pytest && uv run ruff check .` зелёные в `assistant-ui/`.

### Коммит 4 — gateway: расширение memory_propose для idea hint

**Цель**: дать ChatGPT через gateway возможность пометить proposal как
idea (для последующего approval с правильной семантикой).

**Файлы:**

- [ai-memory/memory/proposals/](D:\GitHub\AI-Assistant\ai-memory\memory):
  - Расширить `propose_memory` контракт: принимать опциональные поля
    `as_idea: bool = False`, `subtype: str | None`. Сохраняются в proposal
    payload. При approval — если `as_idea=true` → applied как
    `capture_memory(as_idea=true, subtype, ...)`, иначе — обычный `store_memory`.
  - Тесты `ai-memory/tests/test_proposals_idea.py`: propose с `as_idea` →
    approval создаёт idea-запись с правильной metadata.

- [gateway/gateway/app.py](D:\GitHub\AI-Assistant\gateway\gateway\app.py):
  - `MCP_TO_ROUTE` (line 42-49): без изменений — `memory_propose` остаётся
    единственным write-tool через gateway.
  - Schema `memory_propose` расширяется поддержкой `as_idea`, `subtype`
    в payload. Никаких новых routes / scopes / DENIED_TOOLS-изменений.

- [gateway/gateway/backends.py](D:\GitHub\AI-Assistant\gateway\gateway\backends.py):
  - Route `/memory/propose` (line 31) проксирует расширенный payload
    в `propose_memory` MCP (без изменений route).

- assistant-ui UI inbox (если уже существует — посмотреть в плане 590; если
  нет — добавить минимальный inbox `/proposals` для approval idea-флага):
  - При approval idea-proposal → вызов `capture_memory(as_idea=true, ...)`
    через MCP (а не raw `store_memory`).

**Тесты** `gateway/tests/test_propose_idea.py` (новый):
- `memory_propose` с `as_idea=true` через gateway → 200, payload содержит флаг.
- scope-policy: `memory_propose` остаётся в `memory.propose` scope.

**Acceptance:**
- ChatGPT через gateway: `memory_propose(text, as_idea=true)` → proposal в очереди с idea-флагом.
- assistant-ui inbox: approval idea-proposal → запись идёт через `capture_memory`, не raw `store_memory`.
- Никаких изменений ADR-0001 / scope_policy.py / OAuth metadata.
- `uv run pytest` зелёный в `gateway/` и `ai-memory/`.

### Коммит 5 — Obsidian двусторонний sync

**Цель**: vault = UI для идей. Push при capture/promote/mark-as-idea.
Pull через watchdog при правке `.md`.

**Файлы:**

- [assistant-ui/app/obsidian.py](D:\GitHub\AI-Assistant\assistant-ui\app\obsidian.py):
  - Существующий `export_idea_to_obsidian` (план 590 коммит 6) — без изменений ядра, но добавляется auto-trigger hook:
    - `auto_export_after_capture(clients, memory_id)` — вызывается в `capture_memory` MCP hook (после успешного capture), если `as_idea=true` и `vault_path` сконфигурирован. Создаёт `.md` файл, обновляет `metadata.obsidian_path` через same-identity supersede.
    - `auto_update_after_promote(clients, memory_id)` — вызывается в `promote_memory` hook, обновляет frontmatter (`lifecycle:promoted`, `work_item_id`).
    - `auto_update_after_mark_as_idea(clients, memory_id)` — вызывается в `mark_as_idea` hook (если был общий note, теперь стал idea → создание `.md`).
  - Новая функция `import_from_obsidian(vault_path: Path, relative_path: str) -> dict`:
    1. Чтение `.md` файла.
    2. Парсинг frontmatter (`yaml.safe_load`) + body.
    3. Lookup ai-memory через `find_memory_by_idea_id(frontmatter.idea_id)`.
    4. Сравнение: если `(file_text != memory.text)` или `(frontmatter != memory.metadata)` → same-identity supersede через `store_memory(supersedes_memory_id=memory.id, ...)`.
    5. Возврат `{ok:true, updated:true|false, memory_id:new_id|existing}`.

- `assistant-ui/app/obsidian_watcher.py` (новый модуль):
  - Класс `ObsidianWatcher(vault_path, clients)`:
    - Использует `watchdog` Python library ([pypi.org/project/watchdog](https://pypi.org/project/watchdog/)) — добавить в `assistant-ui/pyproject.toml` + `uv.lock`.
    - Подписывается на FILE_MODIFIED/FILE_CREATED события в `<vault>/Inbox/` (только idea-файлы).
    - **Loop prevention**: short-term cache `Dict[(path, mtime), datetime]` ("recently pushed by us"). При событии — если `(path, mtime)` в cache (< 5 sec назад) → ignore. Иначе — вызов `import_from_obsidian(vault, relative_path)`.
    - **Конфликт-резолюция (LWW по mtime)**: если ai-memory `updated_at` > file `mtime` → ignore Obsidian-edit (наш push новее); иначе — apply.
    - Запуск как FastAPI startup-task (`@app.on_event("startup")`), graceful shutdown через `@app.on_event("shutdown")`.

- [assistant-ui/app/main.py](D:\GitHub\AI-Assistant\assistant-ui\app\main.py):
  - При startup — инициализация `ObsidianWatcher`, если `obsidian_vault_path()` не None.
  - При shutdown — остановка watcher.

- [assistant-ui/app/config.py](D:\GitHub\AI-Assistant\assistant-ui\app\config.py):
  - Новая env `ASSISTANT_UI_OBSIDIAN_WATCH_ENABLED` (default true если vault_path задан). Опция отключить watchdog для отладки.

- [assistant-ui/pyproject.toml](D:\GitHub\AI-Assistant\assistant-ui\pyproject.toml) + `uv.lock`: добавить `watchdog>=3.0` dependency.

- ai-memory MCP hooks (`capture_memory`, `promote_memory`, `mark_as_idea`):
  - В ai-memory side — НЕ вызываем Obsidian export (ai-memory не знает про vault).
  - Auto-export hooks живут в assistant-ui `memory_capture.py` thin client:
    после успешного MCP-вызова — вызов `auto_export_after_capture(...)` etc.
  - Это значит: ChatGPT через gateway → `memory_propose(as_idea=true)` →
    approval в UI → assistant-ui inbox вызывает `capture_memory` MCP → после
    успеха вызывает auto-export hook. Цепочка работает.

**Тесты** `assistant-ui/tests/test_obsidian_sync.py` (расширение существующих + новые):
- Push: capture(as_idea=true) → `.md` создан в `<vault>/Inbox/`, frontmatter валиден.
- Push при promote: frontmatter обновлён (`lifecycle:promoted`, `work_item_id`).
- Pull: модификация `.md` (через `tmp_path`) → `import_from_obsidian` обновляет ai-memory через supersede.
- Loop prevention: после auto-push событие watchdog не триггерит import (cache hit).
- LWW: если memory.updated_at > file.mtime → ignore (мок time).
- Watcher startup/shutdown: graceful.
- Edge cases: malformed YAML → fail-soft (warning, не падение); отсутствие `idea_id` в frontmatter → ignore.

**Acceptance:**
- POST `/api/memory/capture {text:"X", as_idea:true}` → 200, `.md` файл создан автоматически в `<vault>/Inbox/idea-<short_id>-x.md`.
- POST `/api/memory/{id}/promote {...}` → frontmatter обновлён автоматически.
- Правка `<vault>/Inbox/idea-<id>-x.md` (изменение body или frontmatter) → ai-memory обновляется через supersede в течение нескольких секунд.
- Циклы (push → trigger → pull → trigger → ...) НЕ возникают.
- При недоступном vault (env не задан) — capture работает, export skipped (warning).
- `uv run pytest` зелёный в `assistant-ui/`.

## Migration & backward compat

- **Данные**: существующие `type=idea` в БД остаются — проходят (soft) валидацию после коммита 2.
- **API**:
  - `/api/ideas/capture`, `/api/ideas/{id}/promote`, `/api/ideas/{id}/export-to-obsidian` — **удаляются** в коммите 3.
  - `/api/memory/*` — новые, единственный путь.
  - `/ideas` HTML URL — 302-redirect на `/memory?type=idea`.
- **MCP gateway**: scope tree без изменений; `memory_propose` расширяется обратно совместимо (`as_idea` опционален).
- **БД-миграция**: schema_version 8 → 9 (JSON1 индексы). Новые поля опциональны.
- **Vault**: существующие `.md` файлы плана 590 (если есть) остаются. Watchdog при старте сканирует Inbox/ и поднимает уже существующие в short-term cache, чтобы не было ложного триггера на startup.

## Verification scenario (end-to-end)

1. Поднять `ai-memory` (8765), `pm-mcp-server` (8766), `assistant-ui` (8000), `gateway` (8780). Установить `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` или `PM_MCP_OBSIDIAN_VAULT_PATH`.
2. **Bio-style автозапись** (ChatGPT через gateway): `memory_propose({text:"факт о OAuth"})` → proposal в очереди.
3. **Approval в UI**: assistant-ui inbox → Approve → запись с `type=general`. **Obsidian не создаётся** (as_idea=false).
4. **Явная фиксация как note локально**: `POST /api/memory/capture {text:"тривиальная заметка", as_idea:false}` → `type=general`. **Obsidian не создаётся**.
5. **Явная фиксация как idea локально**: `POST /api/memory/capture {text:"гипотеза Y", as_idea:true, subtype:"hypothesis"}` → `type=idea`, `idea_id`. **`.md` файл автоматически создан** в `<vault>/Inbox/idea-<short>-gipoteza-y.md` с валидным frontmatter.
6. **ChatGPT idea capture через gateway**: `memory_propose({text:"идея Z", as_idea:true, subtype:"concept"})` → proposal с idea-флагом. Approval в UI → запись с `type=idea` через `capture_memory` MCP → `.md` создан.
7. **Upgrade post-hoc**: `POST /api/memory/{id}/mark-as-idea {subtype:"concept"}` на записи из (4) → `type=idea`, `.md` создан автоматически.
8. **Promote**: `POST /api/memory/{memory_id}/promote {project_path, title}` для idea из (5) → 200, PM-MCP задача создана, frontmatter обновлён (`lifecycle:promoted`, `work_item_id`). На memory_id из (4 после переноса в idea) → ok. На memory_id из (2) → 422 `not_an_idea`.
9. **Удалённые endpoints**: `POST /api/ideas/capture` → 404. `POST /api/ideas/{id}/promote` → 404. (migration-discipline verification).
10. **UI**: `GET /memory` → все записи с pagination, empty/error states. `GET /memory?type=idea` → только идеи. `GET /ideas` → 302 на `/memory?type=idea`.
11. **Двусторонний sync (pull)**:
    - Открыть `<vault>/Inbox/idea-<short>-gipoteza-y.md` в Obsidian.
    - Изменить body (добавить раздел `## Развитие` с новым текстом).
    - В течение 2-3 секунд → ai-memory обновлена через supersede; `search_by_metadata(field="idea_id", value=<id>)` возвращает новую запись с обновлённым текстом, старая archived.
12. **Loop prevention**: capture новой idea → `.md` создан → watchdog НЕ триггерит import (cache hit), новых супер-седов не создаётся.
13. **Идемпотентность capture**: повторить (5) → тот же `idea_id`, `deduplicated:true`, `.md` файл не пересоздан.
14. **Тесты**: `uv run pytest` зелёный во всех 4 субпроектах.

## Tech-stack кирпичики

(см. [tech-stack-choices.md](D:\GitHub\_engineering_rules\tech-stack-choices.md))

- **#3 SQLite + FTS5 + WAL** + **JSON1 partial индексы для metadata** (см. подсекцию в #3) — индексы `idx_memory_metadata_<field>` на `json_extract(metadata,'$.<field>')` WHERE `archived_at IS NULL` для type/lifecycle/idea_id/work_item_id; verification через `EXPLAIN QUERY PLAN`.
- **#4 namespaced env vars** — переиспользуем `ASSISTANT_UI_OBSIDIAN_VAULT_PATH` + `PM_MCP_OBSIDIAN_VAULT_PATH` (fallback из плана 590); новая `ASSISTANT_UI_OBSIDIAN_WATCH_ENABLED`.
- **#5 FastMCP в loopback** + **Hybrid trust model** (см. подсекцию в #5) — новые MCP tools (`capture_memory`, `mark_as_idea`, `get_memory_entry`, `list_memory`) через `@mcp.tool()` в ai-memory loopback (local trust). External ChatGPT через gateway идёт через расширенный `memory_propose` + approval, **не direct write** (ADR-0001 D-3 сохраняется).
- **#6 FastAPI + Jinja2 + Material Web** — `/memory` UI на тех же примитивах. Inline project selector через `list_projects` PM-MCP MCP (не `prompt()` как в коде плана 590).
- **#7 uv + per-subsystem venv** — `watchdog>=3.0` добавляется в `assistant-ui/pyproject.toml` + `uv.lock`.
- **#8 Singleton-guard для daemon'ов** — отдельный mutex для watcher НЕ нужен: см. #12.
- **#10 Design-system** — UI `/memory` использует MD3 tokens и `<md-*>` components из общего Design-system; никаких локальных CSS / hex-литералов.
- **#11 Filesystem watch (watchdog) + loop-prevention cache** — реализуется в C5 `assistant-ui/app/obsidian_watcher.py`. Cache TTL ~5 sec; eviction — LRU при превышении 1000 записей.
- **#12 Background tasks как FastAPI startup/shutdown** — `ObsidianWatcher` поднимается через `@app.on_event("startup")` в assistant-ui; lifecycle привязан к UI-процессу, отдельный NSSM-сервис не вводится (ADR-0001 D-8).
- **#13 YAML frontmatter через safe_dump/safe_load** — реализуется в C5 для push (`export_idea_to_obsidian` уже использует `safe_dump` после плана 590) и pull (`import_from_obsidian` использует `safe_load`).

**Отклонения от каталога**: нет.

## Открытые вопросы (для следующих планов)

1. **LLM-classifier для агента**: автоматическое определение «это idea?» по контексту разговора. Сейчас агент сам решает на основе фраз; полноценный classifier — отдельный план.
2. **Resurfacing для не-idea записей**: если потребуется push-уведомления о «забытых» Bio-style notes — отдельный план (`forgotten_memory` + `morning_brief` + delivery channel).
3. **PM-MCP cancel-status** для compensation — открытый вопрос плана 590 #7.
4. **Унификация `metadata.task_id` vs `metadata.work_item_id`** — открытый вопрос плана 590 #6.
5. **Backfill старых записей** `type=idea` без `lifecycle`/`subtype` — опциональный CLI (не блокирует, soft-warning достаточно).
6. **Двусторонний sync конфликт-резолюция**: текущая LWW по mtime — простая. Если будут реальные конфликты — улучшить через `git`-style merge или явный prompt пользователю.
7. **`memory.capture` scope для прямого write через gateway** — отклонено в этом плане; если в будущем понадобится для интеграции с другим external клиентом — отдельный ADR с пересмотром ADR-0001.
8. **Inline diff между ai-memory entry и Obsidian .md** в UI `/memory/{id}` — для прозрачности sync status.

## Cross-subsystem work items

По правилу J.4 root AGENTS.md — каждый коммит как отдельная PM-MCP задача под `project_path` затронутого subsystem'а:

- **Коммит 1** → задача в `D:\GitHub\AI-Assistant` (ADR-0005 в `docs/adrs/`, root-level). Никаких зависимостей.
- **Коммит 2** → задача в `D:\GitHub\AI-Assistant\ai-memory` (MCP tools + validation). Зависит от 1 (ADR-якорь).
- **Коммит 3** → задача в `D:\GitHub\AI-Assistant\assistant-ui` (thin client + URL миграция + UI). Зависит от 2.
- **Коммит 4** → задача в `D:\GitHub\AI-Assistant\gateway` + задача в `D:\GitHub\AI-Assistant\ai-memory` (proposals расширение). Обе зависят от 2; gateway-задача зависит от ai-memory-задачи через `link_task_dependency` с `dependency_project_path`.
- **Коммит 5** → задача в `D:\GitHub\AI-Assistant\assistant-ui` (Obsidian sync). Зависит от 3.

Связи оформляются через `mcp__PM-MCP-server__link_task_dependency`. Использовать [pm-mcp-task-flow](D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow) skill для безопасного создания.

## Критичные файлы

**ai-memory:**
- `D:\GitHub\AI-Assistant\ai-memory\memory\validation.py` — расширение whitelist + cross-field + `kind` параметр (soft-warning).
- `D:\GitHub\AI-Assistant\ai-memory\memory\storage.py` — `capture_memory`, `mark_as_idea`.
- `D:\GitHub\AI-Assistant\ai-memory\memory\search.py` — `get_memory_entry`, `list_memory`.
- `D:\GitHub\AI-Assistant\ai-memory\memory\db.py` — schema_version 9 + JSON1 индексы.
- `D:\GitHub\AI-Assistant\ai-memory\memory\mcp_app.py` — регистрация 4 новых MCP tools.
- `D:\GitHub\AI-Assistant\ai-memory\memory\runtime_contract.py` — обновление публичного списка.
- `D:\GitHub\AI-Assistant\ai-memory\memory\proposals/` — расширение propose для `as_idea`/`subtype`.
- `D:\GitHub\AI-Assistant\ai-memory\tests\` — `test_capture.py`, `test_mark_as_idea.py`, `test_get_memory_entry.py`, `test_list_memory.py`, `test_validation.py`, `test_db.py`, `test_proposals_idea.py`.

**assistant-ui:**
- `D:\GitHub\AI-Assistant\assistant-ui\app\ideas.py` → **переименование** в `memory_capture.py` (НЕ `memory.py` — занят chat-context).
- `D:\GitHub\AI-Assistant\assistant-ui\app\obsidian.py` — auto-export hooks + `import_from_obsidian`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\obsidian_watcher.py` (новый) — watchdog daemon.
- `D:\GitHub\AI-Assistant\assistant-ui\app\config.py` — `ASSISTANT_UI_OBSIDIAN_WATCH_ENABLED`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\main.py` — удалить старые `/api/ideas/*`, добавить `/api/memory/*`, redirect `/ideas`, watcher startup/shutdown.
- `D:\GitHub\AI-Assistant\assistant-ui\app\security.py` — protected paths.
- `D:\GitHub\AI-Assistant\assistant-ui\app\dashboard.py` — `memory_view_model`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\templates\memory.html` (новый), `ideas.html` (удалить), `base.html` (nav).
- `D:\GitHub\AI-Assistant\assistant-ui\pyproject.toml` + `uv.lock` — добавление `watchdog>=3.0`.
- `D:\GitHub\AI-Assistant\assistant-ui\tests\` — `test_memory_capture.py`, `test_memory_promote.py`, `test_memory_mark_as_idea.py`, `test_memory_obsidian.py`, `test_memory_view.py`, `test_obsidian_sync.py` (старые `test_ideas_*.py` удаляются).

**gateway:**
- `D:\GitHub\AI-Assistant\gateway\gateway\app.py` — schema `memory_propose` (`as_idea`, `subtype` опциональные поля).
- `D:\GitHub\AI-Assistant\gateway\tests\test_propose_idea.py` (новый).

**root:**
- `D:\GitHub\AI-Assistant\docs\adrs\0005-lazy-idea-classification.md` (новый, English).
- `D:\GitHub\AI-Assistant\docs\adrs\README.md` — индекс.

## Соответствие правилам репозитория

- ADR-0005 — на English (root AGENTS.md).
- Code comments / commit messages / task statuses — на русском.
- Каждый коммит атомарен в `main` (J.4) и закрывает свою PM-MCP задачу.
- Никаких новых daemon'ов: `ObsidianWatcher` — FastAPI startup task в существующем процессе assistant-ui.
- [migration-discipline](D:\GitHub\_engineering_rules\skills\migration-discipline): старые `/api/ideas/capture`, `/api/ideas/{id}/promote`, `/api/ideas/{id}/export-to-obsidian` endpoints + `capture_idea` функция + `_get_memory_entry` workaround helper удаляются в коммите 3 в том же коммите, что и появление новых.
- [tech-stack-choices.md](D:\GitHub\_engineering_rules\tech-stack-choices.md) сверен — отклонений нет.
- ADR-0001 (D-3): gateway scope tree сохраняется без изменений.
- ADR-0004: idea contract расширяется обратно совместимо (`lifecycle`/`subtype` — convention, не required).

## Точки для следующего Codex review

Я бы попросил Codex round 2 проверить:
1. Соответствие ADR-0005 формату ADR-0001/0002/0003/0004 (язык, секции, ссылки).
2. Не вводит ли расширение `propose_memory` с `as_idea` нежелательную семантику в `memory.propose` scope.
3. Loop prevention в watchdog: корректность cache TTL (5 sec — достаточно ли? И нужна ли cache eviction policy при большом vault'е).
4. Конкурентность: что если capture(as_idea=true) и Obsidian-edit того же `.md` происходят одновременно — какая race condition возможна.
5. Backfill — действительно ли soft-warning достаточно, или есть use case, где legacy idea без `lifecycle` ломает promote/export.
