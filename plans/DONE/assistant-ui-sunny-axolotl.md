# Два независимых плана: вкладка «Память» + правила plan mode

Файл держит **два независимых плана** (разный blast radius, закрываются по отдельности). **Первый шаг
исполнения** — физически разнести этот файл в два plan-файла по slug ниже и завести две отдельные
PM-MCP задачи (чтобы central-plan lifecycle их не смешивал):

- ПЛАН 1 — `memory-tab-assistant-ui` (ai-memory → assistant-ui → при необходимости Design-system).
- ПЛАН 2 — `plan-mode-rules-hooks` (`_engineering_rules` + master AGENTS.md + `~/.claude/settings.json`).

Каждый план: своя PM-MCP задача и свои **коммиты в `main`** (отдельные репозитории; ветки/PR — только
по явному запросу пользователя, по AGENTS.md). После создания задач plan-файлы переименовать в
`<id>-<slug>.md`, в harness-файле оставить строку `См. план: …` (по `central-plan-workflow`).

## Context

Нужно две независимые вещи:

1. **Вкладка «Память»** в Assistant UI: просмотр имеющихся записей AI-memory (с поиском; все
   реквизиты доступны через основные колонки + раскрываемую деталь строки) и панель кандидатов из
   очереди AI-memory write/proposals с кнопками «Утвердить» (промоут в AI-memory) и «Отменить»
   (reject). Сейчас экрана нет: `propose_memory` уже есть в loopback MCP, но **admin review surface
   (list/approve/reject) отсутствует** в loopback MCP и в Assistant UI — есть только CLI/Python
   (`proposals-list/-approve/-reject`). Assistant UI ходит в AI-memory исключительно по MCP, поэтому
   фичу нельзя сделать в одном репозитории: сначала открыть review-surface по MCP в `ai-memory`,
   затем потребить её в Assistant UI.

2. **Глобальные правила plan mode**: зафиксировать, что работа над планом (создание/изменение)
   идёт в plan mode и опирается на `D:\GitHub\_engineering_rules\tech-stack-choices.md` и релевантные
   skills/hooks; плюс автоматический пост-анализ утверждённого плана на предмет новых инструментов
   (кирпичики в tech-stack-choices.md / новые хуки под повторяющиеся действия).

## Подтверждённые решения

- Кандидаты включаем сразу. `propose_memory` уже есть в `mcp_app.py`/`runtime_contract.py` —
  новое только чтение/approve/reject очереди. Имена тулз делаем необщими:
  `list_memory_proposals`, `approve_memory_proposal`, `reject_memory_proposal`.
- «Отменить» = `reject` (мягко): `status='rejected'`, аудит сохраняется, авто-очистка через 30 дней
  штатным retention (`ProposalRepository.purge_expired`). Не физическое удаление.
- Plan mode: правило в глобальном AGENTS.md + расширение `context_gate` (подсказка на входе) +
  хук пост-анализа после утверждения. Хук не включает plan mode принудительно — только подсказывает;
  вход в plan mode остаётся за агентом/пользователем.
- Оговорка по докам Claude Code: `PostToolUse` умеет отдавать `hookSpecificOutput.additionalContext`,
  но отдельного события «plan approved» нет, а `ExitPlanMode` в списке матчеров официально не
  задокументирован. Поэтому надёжный механизм пост-анализа — шаг в `central-plan-workflow` + правило
  AGENTS.md; `PostToolUse`-хук с matcher `ExitPlanMode` добавляем как best-effort и проверяем
  эмпирически (см. ПЛАН 2 §2C).

---

## ПЛАН 1 — Вкладка «Память» (slug: `memory-tab-assistant-ui`)

Порядок реализации: `ai-memory` (1A) → `assistant-ui` (1B–1D) → при необходимости Design-system (1C).

### 1A. AI-memory: открыть очередь proposals по MCP
Репозиторий `D:\GitHub\AI-Assistant\ai-memory`. Переиспользуем существующую логику —
новые тулзы только тонкие обёртки; новое — лишь чтение/approve/reject (write/propose уже есть).

- `memory/proposals/proposals.py` уже даёт: `ProposalRepository.list_proposals(status, limit)`,
  `get_proposal(id)`, `mark_rejected(id, reason)`, `mark_approved(id, memory_id)`.
- `memory/proposals/promote.py` уже даёт `promote_proposal(proposal_id)` (полный промоут:
  `store(...)` + `mark_approved` + аудит).
- В `memory/mcp_app.py` добавить три тулзы и зарегистрировать их **и** в локальной сборке
  (`register_read_tools`/новый `register_proposal_admin_tools`, используется `build_mcp_server` →
  фоновый daemon 8765 и локальный stdio) **и** в `build_bridge_stdio_server` (проксирует на 8765),
  чтобы они были доступны в обоих режимах `resolve_stdio_server_mode()`:
  - `list_memory_proposals(status=None, limit=50)` → `ProposalRepository().list_proposals(...)`,
    обёрнуть в `_build_collection_response`; распарсить `tags_json`/`*_patterns_json` в списки.
    **Clamp** `limit` (1..200); **валидировать** `status` по enum
    {proposed, reviewed_clean, reviewed_suspicious, approved, rejected} → невалидный = структурная
    ошибка, не 500.
  - `approve_memory_proposal(proposal_id)` → `promote_proposal(proposal_id, decision_actor="assistant-ui")`.
  - `reject_memory_proposal(proposal_id, reason=None)` → `ProposalRepository().mark_rejected(...)`.
  - Ошибки структурные, не 500: `get_proposal is None` → `{"status":"not_found"}`; недопустимый
    переход (`ValueError` из `promote_proposal`/`mark_rejected`) → `{"status":"invalid_state","reason":…}`.
- `memory/proposals/promote.py`: добавить параметр `decision_actor` в `promote_proposal`
  (по умолчанию `"cli"`; audit-actor сейчас захардкожен `"cli"` на строке ~92), UI-approve передаёт
  `"assistant-ui"`.
- Добавить три имени в `memory/runtime_contract.py` (`EXPECTED_TOOLS`) и обновить
  `scripts/check_stdio_contract.py`/`tests/test_runtime_contract.py`.
- Тесты: расширить proposals-тесты и контрактные тесты (`uv run pytest`).
- Документация (файла `docs/AI-MEMORY-WRITE.md` в дереве нет): обновить proposal-контракт в
  `ai-memory/docs/CONTRACT.md`, `ai-memory/ARCHITECTURE.md`, `ai-memory/README.md` (списки тулз).

### 1B. Assistant UI: backend
Репозиторий `D:\GitHub\AI-Assistant\assistant-ui`.

- `app/mcp_client.py`: добавить `list_memory_proposals`, `approve_memory_proposal`,
  `reject_memory_proposal` в `AI_MEMORY_TOOL_NAMES`; тонкие convenience-методы на `MCPClients` по
  образцу `search_memory`/`get_recent_memory`.
- `app/main.py` — новый роут страницы и API (паттерн как у `/budget`, `/api/budget/*`):
  - `GET /memory` → `templates.TemplateResponse(request, "memory.html", {"active": "memory", …})`,
    лёгкий серверный первичный рендер (recent records + pending proposals) с fail-open
    (ошибка → пустой список + баннер), как делает `budget_page`.
  - `GET /api/memory/entries?q=&project=&kind=&lifecycle=&limit=` → при `q` →
    `clients.call_memory("search_memory", …)`, иначе `get_recent_memory`; всё через
    `run_in_threadpool`. (Возвращает только активную память — см. оговорку в 1C. `search`/`get_recent`
    принимают только `limit`; offset/keyset нет — для MVP не добавляем.)
  - `GET /api/memory/proposals?status=&limit=` → `call_memory("list_memory_proposals", …)`.
  - `POST /api/memory/proposals/{id}/approve` → `call_memory("approve_memory_proposal", {"proposal_id": id})`.
  - `POST /api/memory/proposals/{id}/reject` (body `{reason}`) → `call_memory("reject_memory_proposal", …)`.
  - Write-эндпоинты соблюдают существующий CSRF/Auth-флоу (как budget POST/PUT). Структурные ошибки
    тулз маппить в HTTP: `not_found` → 404, `invalid_state` → 409 (не 500).
- `app/tool_router.py`: **не** добавлять approve/reject (и list) в memory-toolset чата. Сейчас
  `MEMORY_TOOLS = ["search_memory", "get_recent_memory", "store_memory"]` (`tool_router.py:8`) —
  инвариант сохранить и закрепить тестом (см. 1D).

### 1C. Assistant UI: frontend
- `app/templates/memory.html`: `{% extends "ds/base.html" %}`, `_assistant_head`, `_sidebar`,
  `_assistant_scripts`; `md-tabs` с двумя вкладками — «Записи» и «Кандидаты» (паттерн budget/settings).
  - Вкладка **«Записи»** (показывает только **активную** память: `search_memory`/`get_recent_memory`
    фильтруют `archived_at IS NULL` в `search.py`, поэтому `archived_at`/`archive_reason` не выводим —
    они были бы пустой декорацией; read-all для архива — отдельный admin-tool вне scope):
    - поиск с **debounce** (`md-outlined-text-field`) + фильтры project/kind/lifecycle (`md-menu`);
      «показать ещё» через увеличение `limit` (underlying `search`/`get_recent` поддерживают только
      `limit`; offset/keyset — отдельная доработка ai-memory вне MVP).
    - **основные колонки**: `id`, `created_at`, `project`, `agent`, `kind`, `lifecycle`, превью `text`;
    - **раскрываемая деталь строки** (не широкая таблица): полный `text`, `tags`, `task/tasks`,
      `files`, `source`, `commit`, `embedding_model` и полный `metadata` в свёрнутом диагностическом
      блоке (frontend-инвариант: сырой JSON не как основной контент). Так все реквизиты доступны без
      горизонтального скролла.
  - Вкладка **«Кандидаты»**:
    - **основные колонки**: `id`, `proposed_at`, `status`, `kind`, `project`, `tags`, `reason`,
      безопасное (escape) превью `text`; **раскрываемая деталь**: `text_length`,
      `stage2a_decision`/`stage2a_done_at`, `stage2b_decision`/`stage2b_done_at`, `promoted_memory_id`,
      `promoted_at`, `review_reason`, `intake_patterns`. Для `reviewed_suspicious`/`rejected` текст
      по умолчанию скрыт и раскрывается по клику. `text` может быть **`null`** (retention
      `sanitize_promoted` очищает текст у approved через 7 дней) — показывать плейсхолдер
      «(текст очищен retention)», не падать.
    - кнопки **«Утвердить»**/**«Отменить»** в строке (существующий md-dialog confirm-helper).
      Для **терминальных статусов** `approved`/`rejected` обе кнопки скрыты/disabled (бессмысленны на
      закрытых rows). **«Утвердить» дополнительно disabled**, когда `status == reviewed_suspicious`
      или (`status == proposed` и `stage2a_decision != 'clean'`) — зеркалит гард `promote_proposal`
      (`promote.py:50`). Фильтр по `status`.
- `_sidebar.html`: добавить ссылку `память` с тем же паттерном `active`/`aria-current`.
- JS страницы — новый файл `../Design-system/assets/assistant-memory.js` (как у dashboard/budget),
  подключить через `{{ ds_static }}/assets/assistant-memory.js`: переключение вкладок, fetch обеих
  таблиц, debounced-поиск/фильтры/пагинация, действия approve/reject с подтверждением и перезагрузкой.
- **Cross-repo неатомарность**: Design-system — отдельный репозиторий. Порядок: сначала добавить и
  закоммитить ассет в Design-system, затем ссылаться из Assistant UI. В Design-system уже есть
  **незакоммиченные** изменения (`assets/assistant-dashboard.js`, `assets/assistant-ui.css`) —
  не перетереть их; `assistant-memory.js` создаём как новый файл.
- Новый CSS не создавать; переиспользовать существующие классы таблиц/вкладок/бейджей. Если
  стиль действительно нужен — добавить в `../Design-system/assets/assistant-ui.css` (репо Design-system).

### 1D. Тесты и документация Assistant UI
- `tests/test_api_endpoints.py`: расширить `FakeMCPClients.call_memory` под `search_memory`/
  `get_recent_memory`/`list_memory_proposals`/`approve_memory_proposal`/`reject_memory_proposal`;
  тесты: `GET /memory` → 200; `/api/memory/entries` зовёт search vs recent по наличию `q`;
  `/api/memory/proposals` зовёт `list_memory_proposals`; approve/reject зовут нужные тулзы (с CSRF);
  ошибки маппятся в 404/409.
- **Тест-инвариант ToolRouter**: approve/reject/list НЕ входят в `MEMORY_TOOLS` (только
  `search_memory`/`get_recent_memory`/`store_memory`) — чтобы LLM чата не мог утверждать proposals.
- `assistant-ui/ARCHITECTURE.md`: в Data flows добавить страницу `/memory` и `/api/memory/*`.

---

## ПЛАН 2 — Правила и хуки plan mode (slug: `plan-mode-rules-hooks`)

Независим от ПЛАНА 1. Принцип: не плодить параллельные механизмы — встраиваем в существующие точки
(`context_gate.py`, общий `hooks/lib`), новый хук только там, где нет существующего триггера.

### 2A. Правило в глобальном AGENTS.md
Файл-мастер `G:\Мой диск\…\Codex\AGENTS.md` (через `C:\Users\Zaxva\.codex\AGENTS.md`; правим только
мастер, его читают и Claude, и Codex). Точечной правкой (Edit, не перезапись) добавить в Section A
подраздел «Plan Mode» рядом с пунктами про Plan files / Engineering rules:
- создание нового или изменение существующего плана выполняется в plan mode;
- на этапе планирования обязательна сверка с `D:\GitHub\_engineering_rules\tech-stack-choices.md`
  и применение релевантных skills (`central-plan-workflow`, `ai-memory-recall`, `impeccable` и т.д.)
  и hooks;
- после утверждения плана — проанализировать план на новые повторяемые инструменты: кирпичики в
  tech-stack-choices.md (новые/улучшение существующих) и новые хуки под повторяющиеся действия;
  предлагать правки с подтверждением пользователя.
- Примечание: Claude wiring (`settings.json`) — отдельно; для Codex не полагаться на Claude settings,
  базовый контракт — AGENTS.md + skill (если у Codex своя проводка shared hooks — учесть отдельно).

### 2B. Подсказка на входе в планирование (расширить существующий хук)
- `D:\GitHub\_engineering_rules\hooks\lib\context.py`: добавить консервативные `PLAN_PATTERNS`
  (создать/изменить/обновить план, «распланируй/составь план», «draft/update plan», «roadmap»…) и
  функцию `plan_directive(prompt) -> str | None` с текстом-подсказкой (войти в plan mode; свериться с
  tech-stack-choices.md; применить central-plan-workflow и релевантные skills/hooks).
- `D:\GitHub\_engineering_rules\hooks\context_gate.py`: вычислять и `memory_directive`, и
  `plan_directive`, объединять непустые в один `user_prompt_context(...)` (сейчас отдаётся только
  memory-директива).

### 2C. Пост-анализ утверждённого плана
Надёжный механизм (работает и в Claude, и в Codex) — **шаг в `central-plan-workflow` SKILL.md**:
после утверждения плана проанализировать его на (1) повторяющиеся действия → кандидаты в новые hooks;
(2) новые технологические решения → кирпичики в `tech-stack-choices.md` (новые/улучшение); предложить
конкретные правки с подтверждением (по `hook-authoring`). Это закрывает требование детерминированно.

Дополнительно (best-effort, Claude-side) — `hooks/plan_tooling_analysis.py`, событие `PostToolUse`,
matcher `ExitPlanMode`: при срабатывании отдаёт тот же `additionalContext` (хелпер PostToolUse в
`lib/claude.py`; при отсутствии — добавить `post_tool_context()`). Идемпотентен, без сайд-эффектов.
**Проверить эмпирически**, что matcher `ExitPlanMode` вообще вызывает PostToolUse (в доках не
задокументирован); если не вызывает — убрать matcher из settings и оставить только шаг в skill.

### 2D. Проводка, тесты, документация хуков
- `C:\Users\Zaxva\.claude\settings.json`: блок `PostToolUse` с `"matcher": "ExitPlanMode"` →
  `plan_tooling_analysis.py` добавляем только если эмпирическая проверка (2C) подтвердит срабатывание;
  иначе settings не трогаем. (Для 2B правок settings не нужно — `context_gate` уже в `UserPromptSubmit`.)
- Тесты (stdlib-only) в `hooks/tests/`: классификация `plan_directive`, объединённый вывод
  `context_gate`, вывод `plan_tooling_analysis`. Прогон:
  `cd D:\GitHub\_engineering_rules\hooks; python -m unittest discover -s tests`.
- Обновить `D:\GitHub\_engineering_rules\hooks\README.md` (список хуков) и при необходимости
  `skills/central-plan-workflow/SKILL.md` (шаг пост-анализа после утверждения).

---

## Переиспользуемые точки (не писать заново)
- AI-memory: `ProposalRepository` (proposals.py), `promote_proposal` (promote.py), `_build_collection_response`,
  `_register_tool` (mcp_app.py).
- Assistant UI: `MCPClients.call_memory`/convenience-методы (mcp_client.py), `run_in_threadpool`,
  паттерн `budget_page` + `/api/budget/*`, `FakeMCPClients` (tests), confirm-dialog хелпер из
  `_assistant_scripts.html`, классы таблиц/вкладок Design-system.
- Hooks: `lib/claude.py` (`read_event`, `user_prompt_context`, `allow`), `lib/context.py`
  (`classify_prompt`, `memory_directive`), `session_start.py`/`context_gate.py` как образцы контракта.

## Verification (end-to-end)
ПЛАН 1:
1. `ai-memory`: `uv run ruff check .`, `uv run pytest`, контрактная проверка (тест на `EXPECTED_TOOLS`/
   `scripts/check_stdio_contract.py`) — новые тулзы в контракте. Дымовой
   `uv run python -m memory.cli proposals-list`.
2. `assistant-ui`: `uv sync` (если менялись зависимости), `uv run ruff check .`, `uv run pytest`
   (включая тест-инвариант ToolRouter).
3. Браузер `http://127.0.0.1:8000/memory`: «Записи» — debounced-поиск/фильтры/пагинация/раскрытие
   деталей; «Кандидаты» — «Утвердить» создаёт запись в AI-memory (status `approved`, видна в «Записи»),
   «Утвердить» disabled для suspicious/без clean stage2a, «Отменить» → status `rejected`.
   desktop+mobile, без ошибок в консоли (frontend-verification).

ПЛАН 2:
4. Хуки: `cd D:\GitHub\_engineering_rules\hooks; python -m unittest discover -s tests`; ручной прогон
   в plan-mode сессии — на входе видна подсказка плана; пост-анализ приходит после утверждения (через
   skill-шаг гарантированно, через хук — если matcher подтверждён). Проверить, что ничего не ломает
   отказ от плана.

## Последовательность и задачи PM-MCP
Два независимых плана → две PM-MCP задачи и **отдельные коммиты в `main`** соответствующих
репозиториев (ветки/PR — только по явному запросу пользователя, по AGENTS.md). Задачи создавать по
`pm-mcp-task-flow`; enum-значения (`status`/`priority`/`domain`/`type`) не выдумывать — подтвердить по
схеме/существующим задачам.
- **Задача 1 (ПЛАН 1) — «Вкладка Память в Assistant UI»**: коммиты в репозитории AI-Assistant
  (`ai-memory` + `assistant-ui`); ассет `assistant-memory.js`(+CSS при необходимости) — отдельный
  коммит в репозитории Design-system (сначала Design-system, потом ссылка из Assistant UI).
- **Задача 2 (ПЛАН 2) — «Правила и хуки plan mode»**: точечная правка мастер-AGENTS.md (Google Drive,
  не git) + коммит в `_engineering_rules` (хуки/тесты/README/skill) + правка `~/.claude/settings.json`
  (условно, см. 2C).

Перед правками в каждом репозитории — перевести соответствующую задачу PM-MCP в `В работе`; после
проверок закрыть в `Готово`; ключевые решения зафиксировать в AI-memory (`ai-memory-capture`).

## Риски/оговорки
- Хук не может принудительно включить plan mode — только подсказать (ограничение харнесса).
- `matcher: ExitPlanMode` для `PostToolUse` официально не задокументирован — пост-анализ держим
  в `central-plan-workflow` (детерминированно), хук добавляем условно после эмпирической проверки.
- Новые AI-memory тулзы — read/admin для локального доверенного stdio; намеренно НЕ попадают в
  toolset чата (tool_router), чтобы LLM не утверждал proposals сам.
- Claude wiring (settings.json) — отдельно; для Codex не полагаться на Claude settings, базовый
  контракт plan mode — AGENTS.md + skill (если у Codex своя проводка hooks — учесть отдельно).
- Cross-repo Design-system неатомарен; «Записи» показывают только активную память (архив вне scope).
