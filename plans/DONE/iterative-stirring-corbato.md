# План: общий слой hooks + карта ответственности AGENTS.md / Skills / Hooks / AI-memory / PM-MCP

> Пятая редакция (учтены рецензии Codex round 1–3; утверждён round 4 на полный объём).
> По решению пользователя план не переименовывается; текущий файл
> `iterative-stirring-corbato.md` остается каноническим.
>
> **Документационный scope:** полная карта ответственности + аудит skills.
> **Execution scope:** все фазы плана, staged delivery — волнами (Wave 1–4), каждая
> волна отдельными PM-MCP задачами с dependency links. Phase 1 = Wave 1 (первая волна),
> но не финальный объём.
>
> **Codex approval (round 4):** расширить execution scope на все идеи плана, не только
> Phase 1. Сохранить staged delivery: каждая волна — отдельные PM-MCP задачи с
> dependency links. Phase 1 остаётся первой волной; план утверждаем на полный набор
> доработок: hooks core, skills, AGENTS cleanup, live-refresh/server-side enum
> follow-ups и все listed follow-up hooks.

## Контекст

Экосистема использует несколько агентов и интерфейсов (Claude, Codex, ChatGPT,
Hermes, локальные LLM) поверх единых: глобального `AGENTS.md`, каталога skills
(`D:\GitHub\_engineering_rules\skills\`), AI-memory (MCP) и PM-MCP. Недавно заведён
пустой каталог `D:\GitHub\_engineering_rules\hooks\` — слой процедурной автоматизации.

Проблема, выявленная в обсуждении: глобальный `AGENTS.md` смешивает три уровня —
(1) принципы/инварианты, (2) процессы, (3) детали реализации (конкретные вызовы
вроде `get_recent_memory(...)`). Процедуры уже частично вынесены в skills, но
**дублирующая проза в `AGENTS.md` не удалена**. По принципу `migration-discipline`
это незавершённая миграция: новый путь (skills) построен, старый (проза) не убран.

Цель плана:
1. Зафиксировать **формальную карту ответственности** между AGENTS.md / Skills /
   Hooks / AI-memory / PM-MCP и провести аудит существующих skills.
2. Построить **общий слой hooks** (контракт + структура + проводка по агентам).
3. Реализовать **три хука**, запрошенных пользователем:
   - для подходящих типов запросов — сначала прочитать AI-memory;
   - перед доработками — задача PM-MCP в статус «В работе»;
   - после доработок — напоминание о переводе в «Готово».

Ограничение: архитектура общая для всех агентов, не привязана к одному.

---

## Состояние на момент планирования (snapshot, проверено)

**Глобальные правила:**
- Мастер: `G:\Мой диск\Бэкапы, инструкции, настройки, синхронизации\Codex\AGENTS.md`
  (Google Drive). `~/.codex/AGENTS.md` — symlink на мастер; `~/.claude/CLAUDE.md` —
  symlink на `G:\...\Claude\CLAUDE.md` со строкой `@C:\Users\Zaxva\.codex\AGENTS.md`.
- Секции: **A** (universal), **B** (out-of-project), **C** (AI-memory protocol),
  **D** (PM-MCP / Assistant-UI task management).

**Skills (15)** в `D:\GitHub\_engineering_rules\skills\`:
- Процессные: `ai-memory-capture`, `project-onboarding`, `central-plan-workflow`,
  `migration-discipline`, `pm-mcp-task-flow`, `repo-automation-audit`.
- Техника/домен: `doc`, `pdf`, `impeccable`, `frontend-verification`,
  `design-system-integration`, `spreadsheet-to-db-migration`,
  `js-ts-destructuring-refactor`, `agent-session-report`, `repo-release-pr`.

**Слой hooks:** `D:\GitHub\_engineering_rules\hooks\` — **пуст** (миграция не начата).

**Runtime state:** `.gitignore` в `_engineering_rules` игнорирует корневой `.state/`
(стр. 22) и `reports/`; там уже лежит `repo-automation-audit.json`. Поэтому state
хуков кладём в `D:\GitHub\_engineering_rules\.state\hooks\`, **не** внутрь `hooks/`.

**Транспорты и порты (netstat + конфиги, проверено):**
- `ai-memory` → loopback HTTP MCP `http://127.0.0.1:8765/mcp`.
- `budget` → `http://127.0.0.1:8767/mcp`.
- `pm-mcp-server`: у Codex настроен **stdio** (`…\pm-mcp-server\server.py`); **HTTP на
  `http://127.0.0.1:8766`** (`/health`→`{"ok":true}`, `/mcp/get_work_item` работает).
  Для live-refresh использовать только `:8766`. Порты **8770** (401, похоже AI-memory
  proposals) и **8780** (gateway) — **не** PM-MCP.
- Ollama → `http://127.0.0.1:11434` (в Phase 1 **не** используется; см. Р-1).

**Факты pm-mcp (проверено в коде):**
- `update_task_status()` — storage-level (`app/task_store.py:457`) — возвращает только
  `{old_status, new_status}` (стр. 494). MCP-tool в `server.py:4351` оборачивает её и
  добавляет `task_id`, но **`project_path` не возвращает ни один слой**. → state-хук
  берёт `task_id`/`project_path` из **`tool_input`**, результат — для проверки успеха.
- `project_key = Path(project_path).name` (`app/task_store.py:61`): для
  `project_path = D:\GitHub\_engineering_rules` → ключ `_engineering_rules`.
- Статус валидируется server-side (`invalid_status`/`allowed_statuses`); остальные enum
  (`priority`/`domain`/`type`/`kind`) + клиентские схемы — доведение (Later phase 8).
- HTTP pm-mcp на `:8766` → live-refresh `get_work_item` возможен в Later phase 9.

**Возможности хуков по агентам:**
- **Claude Code**: полноценная система хуков; в `~/.claude/settings.json` блока
  `hooks` сейчас **нет** (чистый старт). (Док: https://code.claude.com/docs/en/hooks ,
  https://code.claude.com/docs/en/hooks-guide )
- **Codex**: `approval_policy="never"`, `sandbox_mode="danger-full-access"`; **нет
  generic PreToolUse/PostToolUse/Stop-хуков**, `notify` не покрывает гард правок
  (Р-7). → поведения держим как правило + skill.
- **ChatGPT / Hermes / прочие**: хуков и ФС нет → только проза или принуждение на
  стороне MCP-сервера.

**Статусы PM-MCP** (AI-Assistant/AGENTS.md, K.3): `Бэклог`, `К выполнению`,
`На согласовании`, `В работе`, `Готово`, `Не актуально`. Переходы:
`К выполнению → В работе` (одна активная задача на агента), `В работе → Готово`.
Инструменты: `start_task`, `complete_task`, `update_task_status`, `move_task`, `link_task_dependency`.
Проект `_engineering_rules` зарегистрирован (`list_projects` его возвращает; есть
задачи #603/#652/#655/#669).

---

## Граница: rule / skill / hook (литмус-тест)

| Формулировка | Слой |
|---|---|
| «Учитывать накопленный контекст перед работой» (что верно) | **Rule** (AGENTS.md) |
| «Как искать в памяти хорошо: запрос, project-scope, конфликт с репо» | **Skill** |
| «Когда включать память + факт срабатывания» (детерминированный триггер) | **Hook** |
| `search_memory(query=…, project=…)` (конкретный вызов) | **Реализация** (тело hook/skill) |

**Тест:** уберёшь — меняется *что верно* → **Rule**; отвечает *как* делать класс
работы (нужно суждение) → **Skill**; детерминированный триггер без суждения →
**Hook**; конкретный endpoint/параметр → **деталь реализации**.

---

## Карта ответственности (формальная)

| Слой | Вопрос | Что держит | Чего НЕ держит | Кросс-агентность |
|---|---|---|---|---|
| **AGENTS.md** | Что и почему | Принципы, инварианты, приоритеты, исполнимые правила, *указатели* на skills/hooks | Пошаговые процедуры; вызовы; триггеры | Все агенты (для ChatGPT/Hermes — единственный слой) |
| **Skills** | Как правильно делать класс работы | Процедуры с суждением | Правила-«почему»; триггеры; данные | Claude/Codex и любой loader skills; ChatGPT — вручную |
| **Hooks** | Как выполнить автоматически | Детерминированные триггеры и проверки | Логику с суждением; «почему» | **Скрипты общие, рантайм — нет** (Claude — да; Codex — нет; ChatGPT — нет) |
| **AI-memory (MCP)** | Что решено/узнано раньше | `decision/fact/note/change/task_context`, схема, валидация | Процесс записи (skill); правила (AGENTS.md) | Все агенты через MCP |
| **PM-MCP** | Каково состояние работы | Задачи, статусы, зависимости, workflow, audit | Engineering-правила; память | Все агенты через MCP |

**Уровни принуждения (надёжность = где принуждается):**
1. **Server-side (MCP)** — у всех агентов, обойти нельзя. Лучше для data-integrity.
2. **Hooks (рантайм агента, общие скрипты)** — Claude; Codex ограниченно.
3. **Проза (AGENTS.md/skills)** — у всех, включая ChatGPT; зависит от послушания.

Правило выбора: *прежде чем писать hook, проверь — нельзя ли принудить на стороне
MCP-сервера (покрывает всех агентов сразу)?*

---

## Аудит существующих skills

| Skill | Категория | Дублирует в AGENTS.md | Действие (Later phase) |
|---|---|---|---|
| `ai-memory-capture` | process | Раздел C (почти дословно) | Оставить как «how»; C свести к правилу+указатель |
| `project-onboarding` | process | A «Before you start», git/precedence | Свести к правилу+указатель; триггер «вход в репо» объединить с `repo-automation-audit` |
| `central-plan-workflow` | process | A «Plan files» | Свести к правилу+указатель |
| `migration-discipline` | process | A migration/doc discipline | Свести к правилу+указатель |
| `pm-mcp-task-flow` | process | Раздел D | D свести к правилу+указатель; **добавить шаг start/complete + .state** |
| `repo-automation-audit` | process | A automation-audit (триггеры) | Триггеры → session-start hook; описание → правило+указатель |
| `doc`, `pdf`, `impeccable`, `frontend-verification`, `design-system-integration`, `spreadsheet-to-db-migration`, `js-ts-destructuring-refactor`, `agent-session-report`, `repo-release-pr` | техника/домен | нет | Оставить как есть |

**Важно (кросс-агент):** для ChatGPT/Hermes `AGENTS.md` — единственный слой. При
сведении C/D и пунктов A в `AGENTS.md` остаётся **минимально исполнимое правило**
(самодостаточное без loader skills), а skill держит подробный «how». **Не** сводить к
«см. skill».

**Пересечения:** `project-onboarding` ∩ `repo-automation-audit` (оба «при входе в
репозиторий») → свести триггер в один session-start hook.

**Недостающие skills:** `ai-memory-recall` (как искать память в Типе C),
`hook-authoring` (meta-skill: контракт + проводка по агентам).

---

## Hook runtime contract (общий для всех скриптов)

- **Только Python stdlib** на Phase 1 (никаких внешних зависимостей в hot-path).
- **Вход:** JSON из stdin (по событию хоста). **Выход:** код возврата + stdout/JSON
  согласно контракту события (ниже) + при необходимости atomic-запись в `.state\hooks\`.
- **Atomic write state:** во временный файл + `os.replace()` — без частично записанных
  JSON при гонке Claude/Codex.
- **State-файл** `…\.state\hooks\active-task.<agent>.json`, поля:
  `session_id`, `cwd`, `agent`, `task_id`, `project_path`, `started_at`.
- **TTL (Phase 1, без вызова PM-MCP):** гард опирается **только** на state + TTL по
  `started_at` (напр. >8 ч → протух) + очистку через PostToolUse при
  `complete_task`/смене статуса. Никаких сетевых вызовов из hot-path хука на Phase 1.
- **Live refresh (Later phase 9):** опц. дешёвая `get_work_item(task_id)` через
  `http://127.0.0.1:8766/mcp/get_work_item`; если задача уже не «В работе»/не найдена —
  state очищается.
- **Тесты:** на каждый скрипт — sample JSON fixtures (stdin) + unit-тесты. Хуки не
  уходят в проводку без тестов. `_engineering_rules` не uv-проект → запуск системным
  Python: `python -m unittest discover -s D:\GitHub\_engineering_rules\hooks\tests`.
- **Общая логика** в `hooks/lib/`; агент-специфична только проводка (событие→скрипт).

**Claude output contract по событиям** (по официальным докам):

| Событие | Как влияет | Используем для |
|---|---|---|
| `SessionStart` | stdout → контекст сессии (свежо при каждом старте) | **volatile**: branch/status, дайджест памяти (фаза 2) |
| `UserPromptSubmit` | stdout/`additionalContext` → контекст; можно `decision:"block"` | короткая **директива** (не volatile — риск resume) |
| `PreToolUse` | `permissionDecision:"deny"`+reason / `exit 2` / `"ask"` | гард правок (Hook 2), guard-хуки follow-up |
| `PostToolUse` | feedback после инструмента | запись/очистка `.state` (Hook 2/3) |
| `Stop` | `exit 0`=разрешить; `decision:"block"`+reason=продолжить; защита `stop_hook_active` | напоминание (Hook 3), reminder-хуки follow-up |

---

## Три хука (целевой дизайн — Phase 1)

### Hook 1 — context-gate (память для подходящих типов запросов)

**Правило (в AGENTS.md, заменяет безусловное «первым действием читать память»):**
> Контекст загружается, когда задача зависит от прошлых решений / непрерывности
> проекта (Тип B/C); для самодостаточных одноразовых запросов (Тип A) память не читается.
> **A** (математика, перевод, справка, разовый тех-вопрос) — пропустить.
> **B** (рекомендации, сравнение, проектирование) — лёгкий поиск при необходимости.
> **C** (продолжение проекта, следующий шаг, PM-MCP/AI-memory, ранее принятые
> решения, анализ инфраструктуры) — память обязательна.

**Механизм (Claude) — разнесён по двум событиям из-за риска stale-context:**
- `UserPromptSubmit` → `hooks/context_gate.py`: **только дешёвые эвристики**
  (regex/слова) на Phase 1 (Ollama НЕ в hot-path, Р-1). Для B/C впрыскивает **короткую
  неизменяемую директиву**: «Тип C: вызови `get_recent_memory`/`search_memory` перед
  ответом». Никаких volatile-данных (текст оседает в transcript → stale при `--resume`).
- `SessionStart` → (фаза 2, Р-2) свежий **дайджест памяти**/branch/status — грузится
  заново каждую сессию, не протухает.
- **Codex**: правило A/B/C в AGENTS.md (хука нет). **ChatGPT/local**: проза/MCP.

**Дефолт:** только директива на эвристиках (фаза 1). Дайджест из AI-memory через
SessionStart — фаза 2.

### Hook 2 — задача PM-MCP → «В работе» (гард + напоминание)

Правило задаёт «когда», агент сам делает MCP-вызов, хук **принуждает и поддерживает
состояние** (не делает вызов за агента). **Phase 1 — TTL/state, без сетевых вызовов.**

**Механизм (Claude):**
- `PostToolUse` matcher `mcp__PM-MCP-server__start_task` (и `update_task_status`=
  «В работе») → `hooks/task_state.py`: `task_id`/`project_path` берёт из **`tool_input`**,
  результат — только для проверки успеха; atomic-запись `active-task.<agent>.json`.
- `PreToolUse` matcher `Edit|Write|MultiEdit` → если активной задачи нет (TTL/state) —
  `permissionDecision:"deny"` с подсказкой «создай/выбери задачу и вызови `start_task`».
- `PreToolUse` matcher `Bash` → классификатор команды по спискам Р-6 (правку можно
  сделать в обход Edit/Write): мутирующая без активной задачи → `deny`; read-only →
  пропуск; неоднозначная → `ask`.
- Исключение: explanation-only запросы (анализ/диагностика) задачу не создают (K.2) —
  гард не трогает read-only действия.

**Codex/прочие:** правило + шаг start_task в `pm-mcp-task-flow`.

### Hook 3 — задача PM-MCP → «Готово» (напоминание/подтверждение)

«Готово» автоматически не ставится (один ход ≠ задача завершена).

**Механизм (Claude):**
- `Stop` → `hooks/remind_complete.py`: если `active-task.<agent>.json` есть — **только
  мягкое напоминание** на Phase 1 (Р-5) «активная задача в «В работе»; если доработки
  завершены — вызови `complete_task`». Жёсткий `decision:"block"` не включаем до обкатки
  (если позже — только с защитой `stop_hook_active`).
- `PostToolUse` matcher `mcp__PM-MCP-server__complete_task` (и `update_task_status`=
  «Готово») → очищает `active-task.<agent>.json`.

**Codex/прочие:** правило + шаг complete_task в `pm-mcp-task-flow` + пункт pre-completion checklist.

---

## Структура `hooks/`

```
D:\GitHub\_engineering_rules\hooks\
  README.md                 # контракт + проводка по агентам
  context_gate.py           # Hook 1
  task_state.py             # Hook 2/3: запись/очистка активной задачи
  require_active_task.py    # Hook 2: гард правок (Edit/Write/MultiEdit + Bash)
  remind_complete.py        # Hook 3: напоминание
  lib/                      # stdin-парсинг, чтение .state, эвристики, Bash-классификатор
  tests/                    # fixtures + unit-тесты на каждый скрипт
# runtime НЕ здесь:
D:\GitHub\_engineering_rules\.state\hooks\active-task.<agent>.json
```

**Проводка:** Claude — блок `hooks` в `~/.claude/settings.json`
(`SessionStart`/`UserPromptSubmit`/`PreToolUse`/`PostToolUse`/`Stop`) с командами
`python … hooks\<script>.py`. Codex — правило+skill. ChatGPT/local — правило/MCP.
**State per-agent** (Claude и Codex могут работать параллельно; K.3 — одна активная
задача *на агента*).

---

## Порядок исполнения (staged delivery, волнами)

Каждая волна — отдельные PM-MCP задачи с dependency links. Проект по умолчанию —
`D:\GitHub\_engineering_rules` (hooks, skills, AGENTS, central plan). **Исключение:**
работы, затрагивающие сервер (Wave 3 server-side), — отдельная PM-MCP задача для
`D:\GitHub\AI-Assistant\pm-mcp-server`.

**Wave 1 — hooks core (= Phase 1):**
1. `hook-authoring` skill + `hooks/README.md` + `hooks/lib/` + `tests/` каркас.
2. **context-gate**: правило A/B/C в AGENTS.md (урезанное) + `context_gate.py`
   (`UserPromptSubmit`, только эвристики/директива) + проводка Claude. Codex — правило.
3. **active-task guard**: `task_state.py` + `require_active_task.py` (Edit/Write/
   MultiEdit + **Bash-классификатор Р-6**, TTL-only) + проводка `PostToolUse`/`PreToolUse`
   Claude; шаг start_task в `pm-mcp-task-flow`.
4. **complete reminder**: `remind_complete.py` (`Stop`, мягкое) + очистка в
   `PostToolUse`; шаг complete_task в `pm-mcp-task-flow`.

**Wave 2 — skills + AGENTS cleanup (после обкатки Wave 1; порядок важен):**
5. **Сначала** skill `ai-memory-recall` (как искать память в Типе C) — до тримминга
   memory-прозы (migration-discipline: новый путь до удаления старого).
6. Чистка дублей в AGENTS.md: C/D и пункты A → минимально исполнимое правило
   (самодостаточно для ChatGPT/Hermes) + указатель на skill. Не «см. skill».
7. Session-start hook: свести триггеры `project-onboarding`/`repo-automation-audit`;
   (опц.) `SessionStart`-дайджест памяти для Hook 1 (фаза 2).

**Wave 3 — live-refresh + server-side (server-side → отдельный проект `pm-mcp-server`):**
8. Live-refresh для Hook 2: `get_work_item` через `:8766/mcp/get_work_item` (код хука —
   в `_engineering_rules`; при изменениях сервера — задача в `pm-mcp-server`).
9. Server-side enum: добить `priority`/`domain`/`type`/`kind` + клиентские схемы
   (задача для `D:\GitHub\AI-Assistant\pm-mcp-server`).

**Wave 4 — follow-up hooks (по приоритету):**
10. `admin_guard.py` → `python_env_guard.py` → `memory_write_guard.py` →
    `plan_lifecycle_guard.py` → `frontend_verification_reminder.py` → `migration_doc_guard.py`.

Все шаги Wave 1 обратимы (новые файлы + точечные правки). Шаг 6 (правка глобальных
правил) — только после проверки Wave 1 в реальной работе.

---

## Follow-up hooks (НЕ в первом scope; идеи для Later phases)

- `admin_guard.py` (`PreToolUse`, Bash/PowerShell): блок `Start-Process -Verb RunAs`,
  `sudo/runas`, `sc.exe`, `nssm install/remove`, firewall/hosts/registry/system
  scheduled task. Ложится на глобальное правило «не самоэскалироваться».
- `python_env_guard.py` (`PreToolUse`): блок/предупреждение про `python -m venv`,
  bare `pip install`, bare `pytest/python` внутри AI-Assistant subsystems; подсказывать
  `uv sync` / `uv run pytest` / `uv run python`.
- `memory_write_guard.py` (`PreToolUse` на `store_memory`/`store_memory_batch`/
  `propose_memory`): проверять `kind`, `metadata.source/files/tags`, подозрительные
  PII/health/finance. Сервер уже ловит secrets/tool-call leaks — это семантический слой.
- `plan_lifecycle_guard.py`: для правок `C:\Users\Zaxva\.claude\plans\*.md` —
  проверять slug, секции «Контекст/Цель/Этапы/Риски/Критерии приёмки», ссылку на
  `tech-stack-choices.md` при выборе технологии, напоминать про архивацию в DONE\ после закрытия задачи.
- `frontend_verification_reminder.py` (`Stop`, reminder): если за ход менялись
  `templates/*.html`, CSS/JS или UI-routes — напомнить про `frontend-verification`/скриншоты.
- `migration_doc_guard.py` (`Stop`, reminder): если были удаления/переименования/замена
  API/config — напомнить про `migration-discipline` (убрать старый путь, docs, без shim без ADR).

**Приоритет реализации follow-up:** `admin_guard.py` → `python_env_guard.py` →
`memory_write_guard.py`; остальные позже.

---

## Решения по открытым вопросам (закрыто round 3 с Codex)

- **Р-1. Hook 1 классификатор:** Phase 1 — **только эвристики**. Ollama не включать в
  hot-path первого релиза; вернуться к нему после обкатки при заметных false pos/neg.
- **Р-2. Hook 1 фаза 2:** дайджест памяти и любой volatile context — через
  `SessionStart`, не `UserPromptSubmit` (mid-session инъекция оседает в transcript и
  протухает при resume).
- **Р-3. pm-mcp HTTP:** PM-MCP HTTP на `127.0.0.1:8766` (`/health`→`{"ok":true}`,
  `/mcp/get_work_item` работает). 8770/8780 для PM-MCP live-refresh не использовать.
  Phase 1 остаётся TTL-only.
- **Р-4. PM-MCP `project_path`:** `D:\GitHub\_engineering_rules` (зарегистрирован;
  `list_projects` возвращает; задачи #603/#652/#655/#669; project_key `_engineering_rules`).
- **Р-5. Hook 3:** Phase 1 — мягкое напоминание. Жёсткий `decision:"block"` не включать
  до обкатки; если позже понадобится — только с проверкой `stop_hook_active`.
- **Р-6. Bash-классификатор Hook 2:**
  - **deny** без active task: явная запись/удаление/перемещение файлов, редиректы
    `>`/`>>`, `Set-Content`, `Out-File`, `New-Item`, `Remove-Item`, `Move-Item`,
    `Copy-Item`, `git add`/`commit`, dependency-changing (`uv add/remove`,
    `pip install/uninstall`, `npm|pnpm|yarn add/remove/install`).
  - **allow:** `rg`, `Get-Content`, `Get-ChildItem`, `git status/diff/show/log/branch
    --show-current`, проверки (`pytest`, `ruff check`, `node --check`, `tsc --noEmit`).
  - **ask:** составные команды с `;`, `&&`, `|`, `cmd /c`, `powershell -Command`,
    произвольные `python …*.py`, генераторы и всё, где мутация неочевидна.
- **Р-7. Codex hook surface:** generic PreToolUse/PostToolUse/Stop-хуков нет, `notify`
  не эквивалент Claude hooks и не покрывает гард правок. Для Codex — rule + skill.

---

## Верификация (Phase 1)

**Hook 1:** Тип A («переведи X») → инъекции нет, токены не растут. Тип C («продолжи
задачу #NNN») → директива впрыснута, агент читает память. Сравнить токены A vs C.

**Hook 2:** `Edit`/мутирующий `Bash` (по Р-6) без активной задачи → `deny` с подсказкой
(неоднозначный `Bash` → `ask`). `start_task` → `active-task.<agent>.json` создан
(atomic, `task_id`/`project_path` из `tool_input`) → `Edit` проходит. Протухший по TTL
state → очищается, не блокирует ложно. Read-only запрос → гард молчит.

**Hook 3:** завершение хода при активной задаче → мягкое напоминание в `Stop`.
`complete_task` → state очищен; повторный `Stop` молчит.

**Контракт/тесты:** на каждый скрипт — fixtures + unit-тесты; запись state атомарна.
Запуск: `python -m unittest discover -s D:\GitHub\_engineering_rules\hooks\tests`.

**Проводка/кросс-агент:** `~/.claude/settings.json` с блоком `hooks` валиден
(`claude --debug`/тест хука). Codex-путь: правило A/B/C и шаги start/complete есть в
AGENTS.md и `pm-mcp-task-flow` (у Codex хук не сработает by design).

**Итог в AI-memory:** после утверждения — `decision` (карта ответственности + дизайн
хуков) через `ai-memory-capture`.

---

## Процедурные заметки (central-plan-workflow)

- Файл создан plan-mode под именем `iterative-stirring-corbato.md` и по решению
  пользователя остается под этим именем; имя файла не меняется (rename по task-id отменён).
- План живёт прямо в `C:\Users\Zaxva\.claude\plans\` (это и есть `plansDirectory`);
  отдельный harness-указатель не нужен.
- Архивировать план (перенос в `C:\Users\Zaxva\.claude\plans\DONE\`, не удаление) только
  при: задача закрыта, outcome в AI-memory, архитектурные причины в ADR/docs, нет
  уникальных acceptance criteria только в плане.

## Источники

- Claude Code Hooks reference: https://code.claude.com/docs/en/hooks
- Claude Code Hooks guide: https://code.claude.com/docs/en/hooks-guide
