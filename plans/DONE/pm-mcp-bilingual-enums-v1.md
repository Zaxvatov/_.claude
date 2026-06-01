# PM-MCP bilingual enums — статусы и приоритеты на RU+EN, явный schema

> **Статус**: draft. Подготовлен Claude 2026-05-27 для последующего обсуждения с Codex.
> До approval — PM-MCP задачи не создавать, в коде ничего не менять.
> Связанный draft: [ai-memory-hygiene-v2.md](ai-memory-hygiene-v2.md) (независим, можно делать параллельно).

## 1. Контекст

Агенты (Claude, Codex) при работе с PM-MCP регулярно догадываются о значениях `status`/`priority`, потому что:

- БД хранит EN: `backlog` / `proposed` / `ready` / `in_progress` / `done` / `obsolete` (см. `pm-mcp-server/app/task_store.py:15` `RU_TO_DB_STATUS`).
- API наружу возвращает только RU (`Бэклог`, `На согласовании`, `К выполнению`, `В работе`, `Готово`, `Не актуально`) через `status_to_ru()`.
- `_validate_status` (`server.py:501`) принимает **только RU** — `_validate_status("done")` падает с `invalid_status` несмотря на то, что `done` — каноническое имя в БД.
- MCP tool signatures: `status: str | None` без `Literal` / enum / docstring перечисления — агент не видит валидных значений в schema.
- Skill `pm-mcp-task-flow` запрещает гадать (правильно), но **не содержит таблицу** значений — нечем сверяться без grep по коду.

Конкретный недавний пример (текущая сессия): grep `"status":\s*"(open|in_progress|blocked|review|todo|backlog|ready)"` по дампу `read_tasks` вернул 0 совпадений, потому что значения в JSON были `"Готово"` / `"Не актуально"`. Потеря 1-2 итераций до выясения схемы.

### 1.1 Что уже есть в коде

- Канонические EN-имена в БД (`task_store.py:15`).
- Bidirectional helpers `status_to_db()` / `status_to_ru()` (`task_store.py:65,69`).
- Аналогично для priority (`priority_to_db`/`priority_to_ru` в `task_store.py:73,79`).
- Constants `READY_STATUS` / `IN_PROGRESS_STATUS` / `DONE_STATUS` / `BACKLOG_STATUS` / `APPROVAL_STATUS` / `OBSOLETE_STATUS` (`server.py:88-94`).
- 85 `@mcp.tool()` decorators в `server.py`, из них принимают `status`/`priority`: 7-8 (perimeter изменений ниже).

## 2. Цель

Сделать так, чтобы агенты (и любые external клиенты PM-MCP):
1. **Видели валидные значения статуса/приоритета в schema** — без grep по коду.
2. **Могли передавать оба варианта** (RU и EN) и получать одинаковое поведение.
3. **Видели оба варианта в выдаче** (`status` + `status_en`) — grep по обеим формам работает.
4. **Не ломали обратную совместимость** для существующих RU-only consumers (assistant-ui, gateway, тесты).

## 3. Ограничения и non-goals

### Ограничения
- Канонический формат в БД остаётся EN. Менять storage layer — не цель.
- Output backwards-compatible: поле `status` продолжает содержать **RU** (как сейчас). EN добавляется параллельно.
- Tech stack по `tech-stack-choices.md` #5 (FastMCP) — используем `Literal` в type-hints как нативный механизм schema-экспорта; не пишем кастомные JSON-schema хелперы.
- Не трогать `task_store.py` helpers — они корректны.

### Non-goals
- **Не** менять canonical в БД на RU.
- **Не** переименовывать поля в существующих consumer code (assistant-ui читает `status` — оставляем RU там).
- **Не** добавлять новые статусы в taxonomy (это отдельное обсуждение).
- **Не** трогать `goal_review` / `list_goals` если у goals другой статус-набор (проверить — может, у них `active`/`archived`, отдельная enum).
- **Не** менять process_state (другой enum, отдельная подсистема).

## 4. Объём изменений (детальная инвентаризация)

### 4.1 `app/task_store.py`
- Добавить `VALID_STATUS_ALIASES: dict[str, str]` — объединение RU→EN и EN→EN (identity) для нормализации:
  ```python
  STATUS_ALIASES: dict[str, str] = {
      **{ru: en for ru, en in RU_TO_DB_STATUS.items()},
      **{en: en for en in DB_TO_RU_STATUS.keys()},
  }
  PRIORITY_ALIASES: dict[str, str] = { ... }  # симметрично
  ```
- Helper `normalize_status_input(value: str) -> str` — возвращает RU canonical (для API внутренних проверок), `KeyError` если не найдено.
- Symmetric `normalize_priority_input`.

### 4.2 `server.py` constants и validation
- `VALID_TASK_STATUSES` расширить: `tuple[*RU, *EN]` (12 значений) ИЛИ оставить RU и проверять через `normalize_status_input` (предпочтительно — меньше дрейф).
- `_validate_status`: первой строкой `status = normalize_status_input(status.strip())` (KeyError → ApiError(`invalid_status`) с расширенным `allowed_statuses` включающим EN).
- Симметрично для priority.

### 4.3 `_task_to_dict` и `WorkItem` pydantic
- Добавить computed/explicit field `status_en` рядом со `status`.
- Аналогично `priority_en` (если priority в выдаче).
- `WorkItem` pydantic model — добавить optional поля; consumers не сломаются (extra='ignore' / extra='allow' по умолчанию).

### 4.4 MCP tool signatures (Literal в type-hints)
Перевести `status: str` / `new_status: str` / `priority: str | None` на `Literal[<все 12 RU+EN>] | None`:
- `create_task` — `status: Literal[...] = BACKLOG_STATUS`, `priority: Literal[...] | None`
- `move_task` — `status: Literal[...]`
- `update_task_status` — `new_status: Literal[...]`
- `list_work_items` — `status: Literal[...] | None`
- `list_goals` — `status: Literal[...] | None` (**проверить enum goals отдельно**)
- `goal_review` — то же про goals
- `update_task` / `bulk_update_tasks` — fields dict, тут Literal не помогает; нормализация внутри.

**Открытый вопрос**: подхватит ли FastMCP `Literal` в JSON-schema (пункт 5.1 — точка для проверки в первой коммите).

### 4.5 Тесты
- `tests/test_tasks_db.py` — добавить параметрические assertions, что EN-вариант работает как RU.
- `tests/test_mcp_api_contract.py` — проверить что schema MCP tools содержит enum (если FastMCP экспортирует Literal).
- `tests/test_task_flow.py` — повторить ключевые flow с EN-вариантом.
- Регрессия: существующие RU-only тесты должны проходить без изменений.

### 4.6 Документация
- `pm-mcp-server/docs/MCP_API.md` — обновить таблицу статусов: RU canonical + EN alias.
- `pm-mcp-server/AGENTS.md` — короткая заметка про bilingual.
- `D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow\SKILL.md` — добавить таблицу маппинга (это и так стоит сделать независимо).

### 4.7 Consumer проверки (search-only, не правки)
Найти и проверить:
- `assistant-ui/app/**` — где читается `task["status"]` или сравнивается с RU-строкой → продолжает работать (output не меняется).
- `gateway/**` — есть ли scope-validation на тексте статуса.
- `ai-memory/**` — нет (memory не валидирует PM статусы).

## 5. Этапы

### Phase 0 — schema reconnaissance (15 минут, до основной работы)
- **Цель**: подтвердить, что FastMCP экспортирует `Literal` в JSON-schema MCP tool.
- **Метод**: маленький experiment — заменить `status: str` на `Literal["a","b"]` в одном tool, прочитать получившуюся MCP tool schema через `tools/list`.
- **Risk**: если FastMCP не подхватывает Literal — переход к docstring-only approach (см. §6 риск 1).

### Phase 1 — input нормализация (1 коммит)
- `STATUS_ALIASES` / `PRIORITY_ALIASES` в `task_store.py`.
- `normalize_status_input` / `normalize_priority_input` helpers.
- `_validate_status` / `_validate_priority` в `server.py` принимают оба варианта, нормализуют в RU canonical.
- Расширенный `allowed_statuses` в error response.
- Тесты: параметрические для каждого RU+EN значения.
- **Backward compat**: существующие RU calls работают как раньше.

### Phase 2 — output дополнение (1 коммит)
- `_task_to_dict` добавляет `status_en` / `priority_en`.
- `WorkItem` pydantic — новые optional поля.
- Тесты: assert на presence обоих полей.
- **Backward compat**: `status` остаётся RU, ни один consumer не теряет данные.

### Phase 3 — MCP schema enums (1 коммит, зависит от Phase 0)
- Signatures `Literal[...]` на 6-7 tools.
- Если Phase 0 показал, что FastMCP не подхватывает — fallback на docstring со списком значений (всё равно полезно).
- Тесты: snapshot MCP tools schema, ожидаемый enum/перечисление.

### Phase 4 — docs + skill (1 коммит)
- `MCP_API.md` обновлён.
- `pm-mcp-server/AGENTS.md` обновлён.
- `SKILL.md` skill `pm-mcp-task-flow` обновлён таблицей.
- Reference memory сохранена через `ai-memory-capture` (kind=`fact`, lifecycle=`durable`, tags=`pm-mcp,enum,bilingual`).

## 6. Риски

| Риск | Влияние | Mitigation |
|------|---------|------------|
| FastMCP не экспортирует `Literal` в JSON schema | Среднее: schema-discovery не сработает, только docs | Phase 0 проверка; fallback на docstring enum-перечисление |
| Existing consumer parsing `"Недопустимый статус: X"` по тексту | Низкое-среднее: ломается error handling | Не менять текст error message, только расширить `allowed_statuses` payload |
| Pydantic `WorkItem` strict mode отвергнет новые поля | Низкое: добавление optional field обычно безопасно | Проверить `model_config` — если `strict`, добавить explicit поля |
| `bulk_update_tasks` (fields dict) пропустит нормализацию | Среднее: можно записать `status="done"` напрямую в БД и сломать matching | Нормализация в `_change_task_status` / общем update path, не только в tool entry-points |
| Goals использует другую enum (`active`/`archived`?) и мы её сломаем | Среднее | Phase 0.5 — проверить `list_goals` / `goal_review` существующие значения; goals out of scope если другая taxonomy |
| Дрейф `STATUS_ALIASES` от `RU_TO_DB_STATUS` (две правды) | Низкое: появляется при добавлении нового статуса | Генерировать `STATUS_ALIASES` из `RU_TO_DB_STATUS` (computed, не дублирующий литерал) |
| EN-имена коллидируют с natural-language input («done», «ready» — короткие, могут совпасть со случайным текстом) | Низкое | Не страшно — нормализация работает только для exact match (case-sensitive) |

## 7. Критерии приёмки

- `_validate_status("done")` возвращает `"Готово"` (нормализует).
- `_validate_status("Готово")` возвращает `"Готово"` (как и раньше).
- `_validate_status("invalid")` падает с `ApiError("invalid_status", ..., allowed_statuses=[12 значений])`.
- `read_tasks` / `list_work_items` выдаёт каждый task с `status` (RU) и `status_en` (EN).
- MCP `tools/list` для `create_task` показывает enum в `inputSchema.properties.status` (если Phase 0 success).
- Все существующие тесты проходят без изменений.
- Новые параметрические тесты покрывают все 6 статусов + 4 приоритета в обеих формах.
- `MCP_API.md` и `pm-mcp-task-flow/SKILL.md` содержат таблицу маппинга.

## 8. Вопросы к Codex (для обсуждения)

1. **Field naming**: `status_en` (короткое) или `status_canonical` / `status_db` (semantic)? Я склоняюсь к `status_en` — симметрично и понятно агентам.
2. **API canonical в output**: оставляем `status=RU, status_en=EN` или инвертируем (`status=EN, status_ru=RU`)? Я за статус-кво — меньше регрессий.
3. **`Literal` vs Enum vs docstring**: проверить в Phase 0. Если FastMCP не подхватывает Literal, пробовать `enum.Enum` (FastMCP его обычно понимает). Что предпочитаешь fallback'ом — docstring или Enum?
4. **Goals** (`list_goals`, `goal_review`): у них тот же enum или другой? Если другой — отдельный плановый item или out of scope?
5. **`bulk_update_tasks` нормализация**: внутри tool entry-point (рано) или в shared `_change_task_status` (поздно, но единая точка)? Я за shared.
6. **Skill update сейчас или после Phase 4**: можем обновить `pm-mcp-task-flow/SKILL.md` сейчас (уровень 1 из обсуждения), не дожидаясь кода. Делать?
7. **AI-memory reference memory**: записать таблицу маппинга как `kind=fact, lifecycle=durable, agent=stepa, tags=[pm-mcp, enum, reference]` для cross-agent доступа? Связан с `ai-memory-hygiene-v2.md` (lifecycle ещё не введён — пока без lifecycle).
8. **Process state и другие enums**: распространять bilingual pattern на process_state, kind workflow и пр., или это отдельные планы?

## 9. Out of scope (явно)

- Process state manager (PSM) enums — отдельная подсистема.
- Memory kinds (`fact`/`decision`/...) — это AI-memory, не PM-MCP, и они уже EN.
- Domain enum (`project_tasks`/`todoist`/`calendar`/...) — уже EN, не требует bilingual.
- Type enum для tasks (`task`/`bug`/etc.) — проверить отдельно, вне scope этого плана.
- Перенос canonical в RU в БД.
- Schema migration.

## 10. Связанные документы

- `D:\GitHub\AI-Assistant\pm-mcp-server\app\task_store.py` (canonical mappings)
- `D:\GitHub\AI-Assistant\pm-mcp-server\server.py:80-94, 501-514` (validation)
- `D:\GitHub\AI-Assistant\pm-mcp-server\docs\MCP_API.md` (документация контракта)
- `D:\GitHub\_engineering_rules\skills\pm-mcp-task-flow\SKILL.md`
- `D:\GitHub\_engineering_rules\tech-stack-choices.md` пункт 5 (FastMCP)
- Параллельный draft: [ai-memory-hygiene-v2.md](ai-memory-hygiene-v2.md) (независим)

## 11. После approval

1. Codex и Claude итеративно ревьюят план, фиксируют ответы на вопросы §8 в этом же файле.
2. Создать PM-MCP задачи по фазам (Phase 0-4 → 5 задач, можно объединить 3+4 если хочется).
3. Phase 0 — отдельный тонкий research-task, результат влияет на Phase 3 implementation.
4. После назначения первого ID — переименовать файл в `<id>-pm-mcp-bilingual-enums.md`, harness в `C:\Users\Zaxva\.claude\plans\pm-mcp-bilingual-enums.md` со ссылкой.
5. Skill update (`pm-mcp-task-flow/SKILL.md`) — можно сделать прямо сейчас, до approval основного плана (independent low-risk improvement).
