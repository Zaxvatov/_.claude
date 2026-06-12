# План: plan-mode гейт + пост-плановая ретроспектива (кирпичики / Design-system / skills)

PM-MCP: #816

## Контекст

Пользователь хочет три автоматизации вокруг workflow планов:
1. Любая работа с планами (создание/редактирование) — только в **Plan mode**.
2. По завершении работ — анализ и доработка `tech-stack-choices.md` (кирпичики).
3. По завершении работ — **реальный анализ** (не напоминание) кирпичиков, дополнений в
   Design-system и кандидатов в skills, с доработкой при необходимости.

Уточнение пользователя по триггеру (важное): **«не напоминание, а реальный анализ и, при
необходимости, доработка; выполнять после завершения всех работ по плану или задач по
доработке кода».** → объединяем запросы 2 и 3 в одну ретроспективу на **завершении**, а не на
апруве.

Значительная часть инфраструктуры уже есть (память #1499, #1513, #1480) — задача
**wire-up + extend + migrate**, не greenfield. Не плодим параллельные хуки (фрагментация уже
была отмечена проблемой в #1480).

## Подтверждено по офиц. докам Claude Code hooks + ревью Codex (#1589)

- `permission_mode` есть в **PreToolUse input** (значения вкл. `plan`); как common-поле может
  отсутствовать у части событий → в гейте **fallback: нет поля ⇒ ask** (fail-closed). Главное:
  plan mode детектируется в PreToolUse → Ask 1 можно сделать настоящим гейтом (в #1499 не знали).
- **Output-contract (критично).** Решения PreToolUse должны идти как
  `hookSpecificOutput.permissionDecision` (+ `hookEventName:"PreToolUse"`). Текущие `ask/deny` в
  `lib/claude.py` пишут **top-level `permissionDecision`**, который контрактом PreToolUse **не
  распознаётся** → deny/ask, вероятно, молча игнорируются (косвенно — #708 «Edit прошёл после
  упавшего create_task»). Значит весь guard-слой (admin/python_env/require_active_task/
  plan_lifecycle) сейчас может НЕ принуждать. Правится в **Фазе 0** и включает гейты по-настоящему.
- matcher матчит `tool_name`. `complete_task`/`close_task` уже доказанно срабатывают
  (`plan_archive_reminder`) → используем их и **обходим непроверенный `ExitPlanMode`**.
- `.codex/AGENTS.md` — **symlink** на master `G:\Мой диск\…\Codex\AGENTS.md`; правим master,
  symlink только как entry point.

## Что переиспользуем (already in place)

- `hooks/plan_lifecycle_guard.py` — уже проведён PreToolUse(Write) + PostToolUse(Edit|Write|…).
- `hooks/plan_archive_reminder.py` — уже на complete/close_task; останется как есть (логистика архива).
- `hooks/lib/files.py:is_plan_path` — детектор plan-пути; переиспользуем.
- `hooks/lib/claude.py` — хелперы `allow/post_tool_context` (+ Фаза 0: `pretool_ask`/`pretool_deny`, `permission_mode`).
- `skills/central-plan-workflow/SKILL.md` — раздел «После утверждения» (перенесём в «После реализации»).
- `~/.claude/hooks/enforce-plan-edit.py` — уже удалён (консолидация #709), чистить нечего.

## Дельта

### Фаза 0 — Output-contract fix (фундамент, делать первым)
- `hooks/lib/claude.py`: ввести **PreToolUse-специфичные** `pretool_ask(reason)`/`pretool_deny(reason)`
  → `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask"|"deny",
  "permissionDecisionReason":…}}`. Мигрировать всех текущих вызывающих (admin/python_env/memory_write/
  require_active_task/plan_lifecycle) на них; старые top-level `ask`/`deny` удалить (footgun: будущий
  PostToolUse/Stop hook не должен получить неверный контракт). `allow()` (`{}`) оставляем.
- Обновить тесты, ассертящие `result["permissionDecision"]` →
  `result["hookSpecificOutput"]["permissionDecision"]` (`test_followup_guards.py` и др.).
- **Сайд-эффект (предупредить пользователя):** после фикса deny/ask начнут реально срабатывать у
  ВСЕХ guard-хуков (admin/python_env/memory_write/require_active_task/plan_lifecycle) — ожидаемо,
  но это смена поведения.

### Ask 1 — Реальный гейт «планы создаются/правятся только в plan mode»
- `hooks/lib/claude.py`: + `permission_mode(event) -> str` (нет поля ⇒ "" ⇒ != "plan" ⇒ ask).
- `hooks/plan_lifecycle_guard.py`: ветвление по `hook_event_name`:
  - **PreToolUse** (Edit|Write|MultiEdit): plan-путь (`is_plan_path`) при `permission_mode != "plan"`
    → `pretool_ask` («войди в plan mode, central-plan-workflow»); в plan mode → `allow`. Не-plan `.md` +
    Write поверх существующего → `pretool_ask` (migration-guard). Иначе `allow`.
  - **PostToolUse** (Edit|Write|MultiEdit|NotebookEdit): plan-путь → `post_tool_context(...)`
    (фикс п.3: сейчас `session_context` шлёт `hookEventName:"SessionStart"` на PostToolUse — баг).
- `~/.claude/settings.json`: matcher PreToolUse-инстанса `plan_lifecycle_guard` `Write` →
  `Edit|Write|MultiEdit`.
- Тесты (`tests/test_followup_guards.py`): plan+`"default"`→ask; plan+`"plan"`→allow; Edit+plan вне
  plan mode→ask; **plan без `permission_mode`→ask (fallback)**; PostToolUse Edit plan-пути →
  additionalContext c `hookEventName:"PostToolUse"`.
- Severity = `ask`: легитимный пост-апрувный стэмп `PM-MCP: #id` вне plan mode = один быстрый confirm.

### Ask 2 + 3 — Реальная ретроспектива по завершении работ (4 оси)
Развязка конфликта с active-task guard (ревью Codex п.2): после `complete_task` state очищается и
`require_active_task` (deny) заблокирует реальные правки. Поэтому два слоя:
- **Основной (надёжный) — pre-close в skill/AGENTS:** ретроспектива — ПОСЛЕДНИЙ шаг работ ДО
  `complete_task`, под ещё активной задачей. Производит **артефакт-таблицу** по 4 осям с вердиктом
  на каждую: `no-change` | `needs-user-confirmation` | `follow-up-task`. Правки общих правил — только
  после подтверждения; что не делается сразу → отдельная follow-up задача (per-project, K.4). Потом
  `complete_task`. Оси:
  1. `tech-stack-choices.md` — новые техрешения/паттерны → brick/уточнение;
  2. Design-system (`D:/GitHub/Design-system`) — общие примитивы/токены/компоненты;
  3. Skills — повторившийся паттерн поведения → новый/уточнённый skill (skill-creator);
  4. Hooks — новые детерминированные триггеры → hook-authoring.
- **Страховочный — хук:** NEW `hooks/plan_retrospective.py` (PostToolUse, matcher
  `mcp__pm_mcp_server__complete_task|mcp__pm_mcp_server__close_task|complete_task|close_task`).
  Хук состояния ретроспективы не знает (нет маркера) → директива агенту: **сверь, проводилась ли
  pre-close ретроспектива; если нет — оформи follow-up task** (правки теперь требуют НОВОЙ активной
  задачи, require_active_task). Хук триггерит — сверку/анализ делает агент (карта ответственности).
- `~/.claude/settings.json`: `plan_retrospective.py` в существующий PostToolUse-блок complete/close_task
  (перед `plan_archive_reminder`: анализ → потом архив).
- RETIRE `hooks/plan_tooling_analysis.py` + `tests/test_plan_tooling_analysis.py` — неподключённый
  approval-reminder, заменяется ретроспективой (migration-discipline: без мёртвого параллельного пути).
- NEW `tests/test_plan_retrospective.py` (образец `test_plan_archive_reminder.py`; кейс namespaced
  `mcp__pm_mcp_server__complete_task`).
- `hooks/README.md`: Scripts — убрать `plan_tooling_analysis`, добавить `plan_retrospective`.

### Reliable layer (чтобы анализ реально выполнялся, в т.ч. Codex без хуков)
Хук триггерит, но судит/делает анализ агент по процедуре — «реальность» обеспечивают skill + AGENTS.md:
- `skills/central-plan-workflow/SKILL.md`: перенести анализ из «После утверждения» в раздел
  **«Перед закрытием задачи (ретроспектива)»**: 4 оси + артефакт-таблица + доработка/follow-up под
  активной задачей; правки общих правил — после подтверждения; архивация — следующим шагом.
- **master AGENTS.md = `G:\Мой диск\Бэкапы, инструкции, настройки, синхронизации\Codex\AGENTS.md`**
  (`.codex/AGENTS.md` — symlink/entry point, как файл НЕ править). Section A, строки 77-80:
  «After user approval, analyze … tech-stack bricks and hook changes» → pre-close ретроспектива с
  4 осями. Строка 62 «plans are created and edited in plan mode» уже есть (Ask 1 = хук-энфорс).

## Процесс реализации (дисциплина системы)
1. `require_active_task.py` делает `deny` без активной задачи → создать work item в проекте
   `_engineering_rules` (pm-mcp-task-flow), `start_task` (это запишет state через task_state.py),
   затем правки.
2. Записать полученный PM-MCP id строкой `PM-MCP: #<id>` в этот план (имя файла не менять).
3. По окончании — `ai-memory-capture` (decision/durable): дизайн, pre-close ретроспектива +
   страховочный complete/close hook, ретайр plan_tooling_analysis, связка с задачей.

## Файлы
- `…\hooks\lib\claude.py` (M: **Фаза 0** ask/deny → hookSpecificOutput; + `permission_mode`)
- `…\hooks\plan_lifecycle_guard.py` (M: plan-mode гейт + фикс PostToolUse-контекста)
- `…\hooks\plan_retrospective.py` (A)
- `…\hooks\plan_tooling_analysis.py` (D)
- `…\hooks\tests\test_followup_guards.py` (M: nested-контракт + кейсы гейта/fallback)
- `…\hooks\tests\*` прочие, ассертящие `permissionDecision` (M: nested-контракт)
- `…\hooks\tests\test_plan_retrospective.py` (A)
- `…\hooks\tests\test_plan_tooling_analysis.py` (D)
- `…\hooks\README.md` (M)
- `…\skills\central-plan-workflow\SKILL.md` (M)
- master `G:\Мой диск\Бэкапы, инструкции, настройки, синхронизации\Codex\AGENTS.md` (M: строки 77-80;
  symlink `.codex/AGENTS.md` как файл НЕ править)
- `C:\Users\Zaxva\.claude\settings.json` (M: 2 matcher-правки)

## Верификация
- `cd D:\GitHub\_engineering_rules\hooks; python -m unittest discover -s tests` → всё зелёное.
- `python -m json.tool "C:\Users\Zaxva\.claude\settings.json"` → валидный JSON.
- Контракт: emitted JSON deny/ask = `hookSpecificOutput.permissionDecision` (юнит) + **live-проверка**,
  что один guard (напр. plan_lifecycle на plan-пути вне plan mode) реально просит подтверждение после Фазы 0.
- Smoke тронутых хуков (`echo '<json>' | python <hook>.py`): `permission_mode:"default"`+plan→ask;
  `"plan"`→`{}`; без поля→ask; `complete_task`→`plan_retrospective` даёт 4-осевой контекст.
- `rg plan_tooling_analysis` по `hooks\` и `settings.json` → пусто (ретайр полный).
- Live-тест matcher `ExitPlanMode` **не нужен** — ретроспектива на доказанном complete/close_task.

## Риски / границы
- **Фаза 0 включает реальное принуждение** у всех guard-хуков (раньше top-level decision, вероятно,
  игнорировался). Ожидаемо, но это смена поведения — фиксируем и сообщаем пользователю.
- Ретроспектива — pre-close (под активной задачей); хук — страховка. Авто-«доработки» без
  подтверждения нет (стандартное правило + active-task guard).
- Хук не принимает архитектурных решений (карта ответственности): надёжность = хук (Claude) +
  skill + AGENTS.md (Claude и Codex).
- Гейт Ask 1 = `ask`: легитимные пост-апрувные правки плана → быстрый confirm (не блок).
- AGENTS.md-правка минимальна; при желании — Claude-only (skill+hook) без AGENTS, но тогда Codex
  ретроспективу не делает.
