# Фильтр work items по статусу и семантика «открытые задачи» для внешних агентов

> Статус: ред. 3, утверждён. Задачи созданы, исполнитель — Codex; код ещё не начат.
> PM-MCP: #794 (pm-mcp-server, Этап 1) · #795 (gateway, Этап 2, зависит от #794)

## 1. Контекст и первопричина

Триггер: ChatGPT на вопрос «какие у меня сейчас задачи» ответил, что «не видит в
доступном интерфейсе PM-MCP параметра фильтрации по статусу» и не может получить
только открытые задачи. Это воспроизводимо и упирается в **два независимых слоя**.

### Слой 1 — gateway не объявляет `status` (именно сюда упёрся ChatGPT)

Внешние агенты ходят в PM-MCP только через gateway (`POST /mcp` и профили
`/mcp/read`, `/mcp/write`). Дескриптор инструмента `pm_list_work_items`
объявляет в `inputSchema` всего два поля:

- `gateway/gateway/app.py:836-841` — `_mcp_tool("pm_list_work_items", ..., {"project_path": "string", "limit": "integer"})`. Поля `status` нет.

При этом труба до native-инструмента уже сквозная:

- `gateway/gateway/backends.py:30` — маршрут `/pm/list_work_items → list_work_items` на бэкенде `pm_mcp`.
- `gateway/gateway/backends.py:67-68` — `_tool_arguments` прокидывает в native-tool **любой** ключ, кроме `route`/`tool_name`.
- `gateway/gateway/app.py:880` — `inputSchema.additionalProperties: True`.

То есть `status` уже доедет до native-инструмента, если его прислать. Модель
просто не присылает то, чего нет в `inputSchema`. Это чисто **discoverability**:
параметр не объявлен в `tools/list`.

### Слой 2 — нет понятия «открытые» (бьёт по всем агентам, даже loopback)

Native `list_work_items` параметр `status` имеет, но фильтр — точное совпадение
**одного** статуса:

- `pm-mcp-server/server.py:3626-3644` — сигнатура `list_work_items(domain, project_path, status, include_blocked, limit)`; `status: str | None` без `Literal`/enum и без описания (для write-tools enum уже добавили в #704/#739, для этого read-tool — нет).
- `pm-mcp-server/app/adapters/registry.py:511` — `if status is not None and work_item.status != status: return False` (одно значение).
- `pm-mcp-server/app/adapters/registry.py:474-487` — `_normalize_project_task_status_filter` нормализует вход через `normalize_status_input`, неизвестное значение → ошибка `invalid_status`.
- Нормализация статуса применяется **только** к домену `project_tasks`; остальные домены (`_list_todoist_work_items` и т.д., `registry.py:305+`) сравнивают `work_item.status` с сырым `status`.

«Только открытые задачи» по AGENTS.md K.3 — это множество
`{Бэклог, На согласовании, К выполнению, В работе}` (карта статусов
`pm-mcp-server/app/task_store.py:15-22` → `backlog, proposed, ready, in_progress`),
минус терминальные `{Готово, Не актуально}` (`done, obsolete`). В один вызов это
сейчас не выразить: только 4 отдельных вызова или фильтрация на клиенте.

Ближайший существующий внутренний путь — `task_queue` (`server.py:4219`), он уже
отсекает терминальные, но возвращает «actionable для workflow», а не чистый
список открытых, и наружу через gateway не проброшен.

### Кто через какую поверхность ходит

| Агент | Транспорт | Видит `status`? | Чего не хватает |
|---|---|---|---|
| ChatGPT | gateway `/mcp/read` → `pm_list_work_items` | нет | объявить параметр (слой 1) + семантика open (слой 2) |
| Open-WebUI | native `127.0.0.1:8766/mcp-streamable/mcp` (`tools/README.md:79-80`) | да, точное совпадение | enum-подсказка + семантика open (слой 2) |
| Hermes | не нашёл конфига в репо — **подтвердить транспорт** | зависит | если gateway → оба слоя; если loopback → слой 2 |

## 2. Цель

Любой агент (ChatGPT через gateway, Open-WebUI/Hermes через loopback) одним
вызовом `list_work_items` / `pm_list_work_items` получает только открытые задачи,
а словарь допустимых значений статуса виден прямо в схеме инструмента.

## 3. Ограничения и опоры

- **tech-stack #5 (FastMCP, loopback, hybrid trust)**: изменение остаётся в
  read-периметре (`pm.read` / read-only tool). Прямой write через gateway не
  добавляем, `DENIED_TOOLS` не трогаем → **новый ADR не требуется**. Имена
  инструментов не меняются → `runtime_contract.EXPECTED_TOOLS` не затрагивается.
- **tech-stack #14 (hybrid consumption)**: один сервис-слой обслуживает и
  loopback, и gateway — менять надо native-логику + gateway-дескриптор, не
  плодить вторую реализацию фильтра.
- **AGENTS.md K.3**: канонический источник определения «открытые/терминальные».
- **AGENTS.md K.4**: кросс-субсистемная работа → отдельный PM-MCP work item на
  каждую субсистему (`pm-mcp-server`, `gateway`), плюс dependency-link.
- **AGENTS.md L.1/L.2**: обновить docs в том же коммите; **English — только
  `AGENTS.md`** (root и subsystem). `pm-mcp-server/README.md`, `docs/MCP_API.md`,
  `gateway/README.md` уже русскоязычные — сохраняем существующий язык/стиль
  субсистемы, не переводим (Codex, ред. 3).
- **AGENTS.md E.1/E.2**: комментарии/докстринги — по-русски, идентификаторы —
  по-английски, существующие имена не переименовывать.
- **Совместимость**: текущий точечный фильтр `status="В работе"` и bilingual
  RU/EN aliases (#739) должны продолжать работать без изменений.

## 4. Согласованные решения (с Codex)

**D1 — групповые ключи `open`/`closed` в `status`, project_tasks-only.** `status`
дополнительно принимает `open` (= 4 открытых) и `closed` (= 2 терминальных) поверх
точечных RU/EN значений. **`active` как группа отвергнут**: это уже реальный статус
Todoist (`todoist.py:222`), Obsidian (`obsidian.py:93`) и Gmail (`gmail.py:51`),
общий enum был бы двусмысленным. Состав групп — по словарю project_tasks (K.3).

**D2 — группы `open`/`closed` подразумевают project_tasks (Codex, ред. 3).**
Прежняя формулировка «группа разрешена только при явном `domain="project_tasks"`»
конфликтовала с критерием `list_work_items(status="open")` **без** `domain`
(в registry `domain=None` = все домены). Финальное правило:
- `domain=None` + `status ∈ {open, closed}` → **effective domain авто-сужается до
  `project_tasks`** (без шума в diagnostics по прочим доменам). Это ключевой кейс
  ChatGPT: «какие открытые задачи» без указания домена.
- `domain="project_tasks"` + группа → фильтр по множеству.
- `domain=<non_project>` + группа → ошибка `invalid_status` (группы не определены
  для todoist/calendar/obsidian/budget/gmail).
- Точечный `status` (RU/EN) с `domain=None` работает как прежде по всем доменам.
- Per-domain mapping (`active`/`scheduled`/`done`) — на будущее, вне MVP.

**D3 — gateway = parity обоих контрактов, а не «добавить поля».** Контракты сейчас
рассинхронены: `openapi.yaml` для `/pm/list_work_items` имеет
`domain/status/include_blocked`, но **без `limit`** (`openapi.yaml:104-107`); MCP
descriptor имеет только `project_path/limit` (`app.py:836-841`). Цель — привести оба
к общему набору `domain, project_path, status, include_blocked, limit` с enum/description.

**D4 — один общий инкремент.** Срочности нет ⇒ без временного «только точечный
status через gateway». Порядок: pm-mcp-server (семантика) → gateway
(parity/discoverability) → docs + live.

## 5. Этапы

### Этап 1 — pm-mcp-server: семантика статуса (слой 2)

1. **Отдельный read-only resolver** в `app/task_store.py`: `OPEN_STATUSES`/
   `CLOSED_STATUSES` (RU canonical, состав — AGENTS.md K.3) +
   `WORK_ITEM_STATUS_FILTER_ALIASES` / `resolve_work_item_status_filter()`,
   принимающий точечный RU/EN alias **или** группу `open`/`closed` и возвращающий
   множество RU canonical статусов. **`STATUS_ALIASES` не трогаем** — иначе группы
   протекут в write-path (`move_task`/`update_task_status` через
   `normalize_status_input`) (Codex #1).
2. `app/adapters/registry.py`: применить resolver к фильтру `status` (матч по
   **множеству**, `registry.py:511` → `work_item.status not in resolved_set`).
   По D2: групповой ключ + `domain=None` → сузить `_selected_work_item_domains`
   до `(project_tasks,)`; групповой ключ + явный non-project `domain` → ошибка
   `invalid_status`. Существующую ошибку дополнить групповыми ключами в
   `allowed_status_inputs`.
3. **Native schema через Literal, не docstring (Codex #5):** ввести
   `WorkItemStatusFilterInput` = Literal из точечных RU+EN + `open`/`closed` и
   проставить его типом `status` в `list_work_items` (`server.py:3661`).
   **`TaskStatusInput` (`server.py:112`) не менять** — это вход write-tools.
   `WorkItemDomain` (`server.py:136`) уже Literal.
4. Тесты `tests/test_adapters_registry.py`: `open`/`closed` для project_tasks;
   `status="open"` при `domain=None` → авто-сужение до project_tasks (нет прочих
   доменов в diagnostics); точечный RU/EN не сломан; `include_blocked`; групповой
   ключ + явный non-project `domain` → ошибка. `tests/test_http_transport.py`:
   enum в схеме `tools/list` + end-to-end вызов. Характеризационные тесты — до правок.
5. Docs: `docs/MCP_API.md` (статусы + групповые ключи + ограничение
   project_tasks-only), `README.md` (сигнатура `list_work_items`).
6. Валидация: `cd pm-mcp-server; uv run ruff check .; uv run pytest`.

### Этап 2 — gateway: contract parity (слой 1). Зависит от Этапа 1

1. Расширить билдер `_mcp_tool` (`gateway/gateway/app.py:864-890`): разрешить в
   `properties` значение-словарь (полная схема свойства с `type`/`enum`/
   `description`), сохранив обратную совместимость со строковым типом.
2. Привести **оба** контракта `list_work_items` к общему набору `domain`,
   `project_path`, `status`, `include_blocked`, `limit` с enum/description
   (`status` enum = как в native, `domain` enum = `WorkItemDomain`):
   - MCP descriptor `pm_list_work_items` (`app.py:836-841`) — сейчас только
     `project_path/limit`, добавить `domain/status/include_blocked`;
   - `gateway/openapi.yaml` (`/pm/list_work_items`, ~строки 104-107) — сейчас
     `domain/status/include_blocked`, добавить `limit`.
   Маршрутизацию/бэкенд не трогаем (уже сквозные, `backends.py:67`).
3. `gateway/README.md` (+ `ARCHITECTURE.md` при необходимости): фильтр статуса в
   `pm_list_work_items`, в т.ч. `status=open`.
4. Тесты: `test_gateway_contract.py` — поля + enum в `tools/list` и OpenAPI parity;
   `test_integration.py` — pass-through `status=open` в бэкенд. `/mcp/read`
   по-прежнему отдаёт `pm_list_work_items`.
5. Валидация: `cd gateway; uv run ruff check .; uv run python -m unittest discover -s tests`.

### Этап 3 — docs-свод и live-проверка

- **Admin-граница (Codex #6):** агент **не перезапускает NSSM сам**. Выдаёт
  пользователю elevated-PowerShell snippet (`Restart-Service AI-Assistant-PM-MCP-server`,
  `AI-Assistant-Gateway`) и продолжает только после подтверждения, что сервисы
  перечитаны (brick #1/#8).
- Loopback: `list_work_items(project_path=..., status="open")` → только открытые.
- Gateway-read коннектор: `pm_list_work_items` показывает `status` в схеме и
  возвращает только открытые при `status=open`.
- Если live-проверку выполнить нельзя — отметить явно (AGENTS.md M.1).

## 6. Риски

- **Семантический дрейф «open».** Единственный источник состава групп — K.3.
  Закрепить тестом, чтобы resolver не разъехался с документацией.
- **Протечка групп в write-path.** Если группу добавить в `STATUS_ALIASES`,
  `move_task(status="open")` стал бы валиден. Митигировано: отдельный read-only
  resolver, `STATUS_ALIASES`/`TaskStatusInput` не трогаем (Codex #1, #5).
- **Смешение доменов при группе.** `status="closed"` без ограничения вернул бы
  закрытые project tasks + активные прочие домены. Митигировано: группа +
  `domain=None` авто-сужается до project_tasks, явный non-project `domain` →
  ошибка (Codex #2 / ред. 3).
- **Тройной дрейф контракта.** Контракт повторяется в native schema, MCP
  descriptor и OpenAPI. Митигировано parity + contract-тестом; кандидат на
  hook/contract-check (обсудить после реализации).
- **Совместимость.** Регресс точечного фильтра и bilingual — закрыть
  характеризационными тестами до правок.

## 7. Критерии приёмки

- [ ] `list_work_items(status="open")` **без `domain`** (loopback) авто-сужается
      до project_tasks и возвращает только
      `{Бэклог, На согласовании, К выполнению, В работе}`, исключая
      `{Готово, Не актуально}`, без шума по прочим доменам.
- [ ] MCP descriptor и `openapi.yaml` для `list_work_items` имеют единый набор
      `domain/project_path/status/include_blocked/limit`; `status` с enum и
      описанием; `/mcp/read` отдаёт `pm_list_work_items`.
- [ ] Через gateway вызов с `status=open` возвращает только открытые задачи.
- [ ] Точечный фильтр (`status="В работе"`/`in_progress`) и bilingual aliases
      работают как раньше; `STATUS_ALIASES`/`TaskStatusInput` не изменены.
- [ ] Групповой ключ при явном non-project `domain` → `invalid_status`, покрыто тестом.
- [ ] `docs/MCP_API.md`, `pm-mcp-server/README.md`, `gateway/README.md`,
      `gateway/openapi.yaml` описывают только текущую модель (без legacy).
- [ ] ruff + тесты зелёные в обеих субсистемах; live-проверка Этапа 3 пройдена
      или явно отмечена как невыполнимая.

## 8. Тест-команды

```powershell
cd D:\GitHub\AI-Assistant\pm-mcp-server; uv run ruff check .; uv run pytest
cd D:\GitHub\AI-Assistant\gateway; uv run ruff check .; uv run python -m unittest discover -s tests
```

## 9. PM-MCP задачи (созданы, K.4)

- **#794** `pm-mcp-server` (Этап 1, `К выполнению`): read-only resolver
  `open/closed` + авто-сужение домена + `WorkItemStatusFilterInput` Literal +
  тесты + docs.
- **#795** `gateway` (Этап 2, `К выполнению`): parity descriptor↔OpenAPI +
  `_mcp_tool` builder + contract-тесты. **Зависит от #794**
  (`dependency_project_path="pm-mcp-server"`, link установлен).

Исполнитель — Codex. Обе в `К выполнению`; порядок обеспечивает dependency-link
(#795 не начинать раньше #794).
