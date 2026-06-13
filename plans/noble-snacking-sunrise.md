# Agent Orchestrator — архитектурный анализ и проектное предложение

PM-MCP: — (задача создаётся после утверждения документа)
Статус: draft v1 — анализ и проектирование, БЕЗ реализации (по требованию пользователя)
Дата: 2026-06-12
Scope: локальная оркестрация Claude Code / Codex для автономного выполнения многошаговых планов с переживанием лимитов и перезапусков.

---

## 0. TL;DR — главные выводы

1. **Свой «оркестратор-платформу» писать не нужно. Нужен тонкий execution-слой.**
   Control plane уже существует — PM-MCP (workflow runs, batch/portfolio очереди, handoff v1
   с `recovery_action`/`stop_reason`/`autonomy_level`, approval gates,
   `auto_resume_paused_workflows`, audit, SSE events). Драйвинг агентов дают официальные
   механизмы: **Claude Agent SDK** (headless Claude Code, resume по session_id, структурный
   `RateLimitInfo`) и **`codex exec` / `codex exec resume`** (+ `--output-schema`).
   Недостающее — **Agent Runner**: адаптеры двух агентов, quota-менеджер, durable-петля
   выполнения. Это единицы тысяч строк, а не платформа.

2. **Критический gap в существующей базе:** workflow runs PM-MCP живут в памяти процесса
   (`pm-mcp-server/server.py:262` — `WORKFLOW_RUNS: dict = {}`). Требование «переживать
   перезапуск» означает: персистировать workflow state в SQLite PM-MCP (brick #3) —
   это расширение PM-MCP, а не новая система. Иначе любой ребут обнуляет процесс.

3. **Claude Desktop исключить из контура.** У него нет API/CLI для автоматизации;
   та же подписка покрывает Claude Code (headless + Agent SDK). «Codex chat N» — это
   `codex exec` сессии с resume, не чаты приложения.

4. **Самый недооцененный риск — не лимиты, а права и качество без человека.**
   Лимиты решаются ожиданием. Автономная серия шагов с `danger-full-access` (текущий
   конфиг Codex) противоречит вашему же контуру #853 / ADR-0018/0019 / brick #24
   (write-proposal + approval). Дизайн ставит approval gates и бюджеты по умолчанию.

5. **Рекомендуемая топология:** новая подсистема `agent-runner/` в монорепо AI-Assistant,
   запуск как **Layer B** (logon scheduled task, brick #20) — потому что креды обоих CLI
   per-user. Runner **stateless между тиками**: вся durable-правда в PM-MCP + артефакты
   в файлах; перезапуск = перечитать состояние и продолжить.

---

## 1. Контекст

### 1.1 Текущий ручной процесс (фактический, по AI-memory)

Процесс уже устоялся и выглядит так (memory id 1655, 1661, 1667–1669, 1674–1676):

1. Claude (plan mode) пишет план в `C:\Users\Zaxva\.claude\plans\<slug>.md`.
2. Codex ревьюит план (2–5 раундов), замечания уходят в AI-memory (notes/proposals).
3. Claude вносит правки, план утверждается пользователем.
4. План декомпозируется в задачи PM-MCP (#NNN) с зависимостями (`link_task_dependency`).
5. Codex реализует задачи, статусы двигаются через PM-MCP, итоги — в AI-memory.

Узкое место — ручной перенос между агентами и ожидание сброса лимитов (5-часовые и
недельные окна у обоих провайдеров): процесс может стоять часами/днями, потом
возобновляется вручную.

### 1.2 Окружение (проверено в этой сессии)

| Компонент | Факт | Evidence |
|---|---|---|
| Claude Code | 2.1.96, нативно на Windows, на PATH | `claude --version` |
| Codex | Desktop-приложение 26.609.30741 + CLI `codex.exe` в `%LOCALAPPDATA%\OpenAI\Codex\bin\<hash>\` (не на PATH sandbox-шелла); sessions в `~/.codex/sessions`, есть `automations/` | `~/.codex/config.toml` (`CODEX_CLI_PATH`) |
| Codex config | `approval_policy="never"`, `sandbox_mode="danger-full-access"`, model gpt-5.5; MCP: ai-memory (8765), budget (8767), pm-mcp (stdio); общие hooks проведены | `~/.codex/config.toml` |
| PM-MCP | Задачи/цели/workflow/PSM/audit; workflow runs **in-memory**; SQLite только tasks/deps/history/process/audit/calendar | `pm-mcp-server/server.py:262-267`, `app/tasks_db.py` |
| AI-memory | Долговременная память, proposals-контур, staleness/lineage (ADR-0020/0021) | `ai-memory/` |
| assistant-ui | Dashboard (в т.ч. «paused workflows»), chat-runtime над LLM API (Ollama/cloud) — НЕ драйвер coding-агентов | `assistant-ui/ARCHITECTURE.md` |
| Hooks-слой | Общие хуки для Claude и Codex (`_engineering_rules/hooks`): context gate, active-task guard, plan lifecycle и т.д. | `~/.codex/config.toml [hooks]` |
| Gateway | Внешний ingress, OAuth/PKCE; в локальном контуре не участвует, но задаёт S2S-identity направление (ADR-0019) | `docs/adrs/0019` |

### 1.3 Возможности агентских CLI (сверено с актуальной документацией, июнь 2026)

**Claude Code / Claude Agent SDK (Python):**
- headless: `claude -p` (`--print`) + permission-флаги; `--output-format json/stream-json`;
- сессии: `session_id` из JSON-вывода, `--resume <session-id>` / `resume` в
  `ClaudeAgentOptions`; транскрипты на диске (`~/.claude/projects/...`);
- **`RateLimitInfo`**: status `allowed | allowed_warning | rejected`, reset-времена,
  типы окон `five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`, `overage`,
  utilization — структурный сигнал лимитов прямо в SDK (фикс парсинга
  `rate_limit_event` — SDK ≥ 0.1.40);
- hooks, allowedTools/permission modes, MCP-клиент.

**Codex CLI (0.41+):**
- `codex exec` — одношотный non-interactive запуск с exit code, под cron/CI;
- `codex exec resume <session>` — продолжение с сохранением транскрипта, plan history
  и approvals; `--output-schema` — финальный ответ по JSON Schema;
- `/status` показывает лимиты и reset-времена; в JSONL-событиях есть снапшоты rate limits;
- встроенные Automations (scheduled background runs) — но только для Codex.

Источники: [Codex non-interactive](https://developers.openai.com/codex/noninteractive),
[Codex CLI reference](https://developers.openai.com/codex/cli/reference),
[Agent SDK Python](https://platform.claude.com/docs/en/agent-sdk/python),
[issue #50518 (RateLimitInfo)](https://github.com/anthropics/claude-code/issues/50518).

---

## 2. Критический разбор идеи (что в постановке ошибочно)

### 2.1 Claude Desktop как управляемый агент — нереализуемо и не нужно
У Claude Desktop нет API, CLI или сессионного протокола; автоматизация возможна только
хрупким UI-кликаньем (computer-use), что для ночного робота неприемлемо. Та же подписка
Max покрывает Claude Code. **Решение:** роль «планировщика» исполняет Claude Code
(plan mode — уже ваш процесс), Claude Desktop остаётся ручным инструментом человека.

### 2.2 «Чат» — неправильная единица процесса
В постановке шаги привязаны к «чатам» («Codex, чат 1»). У CLI-агентов единица — это
**сессия с resume**, но многодневная сессия — плохой носитель истины: контекст растёт,
срабатывает compaction, качество дрейфует, а формат session storage — внутренняя деталь
вендора. **Решение:** единица процесса — **артефакт** (план.md, review-notes.md, diff,
отчёт по фиксированной схеме) + handoff payload PM-MCP (ADR-0002). Сессия — оптимизация
(сохранить контекст между соседними шагами одного агента), а не хранилище состояния.
План ссылается на «именованные сессии» (`codex:review-1`), но каждый шаг обязан быть
выполним и с чистой сессией — из артефактов.

### 2.3 «Оркестратор передаёт контекст между агентами» — не строками, а файлами
Перетаскивать выводы агента A в промпт агента B как текст — воспроизводит ручной
copy-paste со всеми его потерями. У вас уже есть общий носитель: **репозиторий +
файлы планов + PM-MCP задача + AI-memory**. Шаг читает вход из файлов/PM-MCP и пишет
выход в файлы/PM-MCP; runner передаёт только **пути и ссылки**, не содержимое.
Это же решает проблему «контекст не влезает в промпт».

### 2.4 Линейный список шагов не переживёт первое ревью
Пример плана (11 шагов) — линейный, но реальный процесс итеративен: «Codex ревьюит →
Claude правит → Codex ревьюит снова» до сходимости (у вас бывало 5 раундов). Формат
плана обязан поддерживать **циклы с условием выхода** (gate: `approved | needs_revision`,
`max_rounds`) — иначе оркестратор остановится на первом «есть замечания».

### 2.5 Лимиты — не главное препятствие
Ожидание сброса автоматизируется тривиально. Настоящие препятствия автономности:
(а) права на запись/команды без человека, (б) качество результата без ревью-гейтов,
(в) расход общих с человеком лимитов. Дизайн ниже посвящает этому больше места, чем
самим лимитам.

### 2.6 «Оркестратор хранит состояние» — да, но не в новом сторе
Отдельная БД оркестратора воспроизводит проблему, из-за которой был удалён PM-Agent
(ADR-0001, альтернатива B): три источника правды (PM-MCP, runtime оркестратора,
AI-memory handoff). **Решение:** durable-состояние процесса — в PM-MCP (расширить),
у runner — только эфемерное (PID живого процесса, буферы вывода).

---

## 3. Недооцененные риски

| # | Риск | Почему серьёзно | Митигдация (в дизайне) |
|---|---|---|---|
| R1 | **Unattended-права.** Серия шагов без человека с текущим Codex-конфигом (`approval_policy=never`, `danger-full-access`) и/или `--dangerously-skip-permissions` у Claude | Прямо противоречит вашему контуру #853, ADR-0018/0019, brick #24 (propose + approval). Ночной робот с полным доступом — максимальный blast radius | Per-run tool policy: deny-by-default; для Codex — `--sandbox workspace-write`, отдельный профиль; для Claude — allowedTools allowlist; запреты: push, удаления вне workspace, сеть кроме allowlist. Approval gates на необратимое |
| R2 | **Prompt injection из репо/веба.** Агент читает файлы и интернет; внушённая инструкция распространяется по цепочке шагов без человека | Ваш же аудит зафиксировал F-5 (agent auto-execute) как находку | Гейты между шагами; выходные артефакты по фикс. схеме; запрет «сырого» переноса инструкций из выходов в промпты; lint выходов (secret-scan уже есть в proposals intake) |
| R3 | **Выжигание лимитов самим оркестратором.** Retry-шторм или плохо поставленный шаг сжигает 5h/недельное окно за ночь; утром у человека пустая подписка | Недельное окно общее у человека и робота | Бюджеты: max_attempts/step, max_steps/run; **quota reserve** (не работать, если utilization недельного окна > N%, конфиг); рабочие часы робота; backoff с jitter; kill-switch (стоп-файл/кнопка в UI) |
| R4 | **Качество без человека.** Ошибка шага 2 компаундируется к шагу 11; «готово» без тестов | Сейчас качество держится на ваших ручных ревью | Верификационные шаги обязательны в плане (ruff/tests per AGENTS.md M.2/M.3); кросс-ревью Claude↔Codex как штатный примитив плана; human-гейты перед merge-точками |
| R5 | **Windows-специфика.** (а) Креды обоих CLI per-user → служба session 0 не аутентифицируется; (б) сон/гибернация останавливают всё; (в) `codex.exe` лежит в каталоге с хэшем — путь нестабилен | Инциденты session-0 уже были (WinError 5, brick #20) | Runner — Layer B (logon task, brick #20); опция wake timers (`powercfg`) — отдельное решение пользователя; путь Codex резолвить динамически (PATH user-сессии / скан `%LOCALAPPDATA%\OpenAI\Codex\bin`) |
| R6 | **Конкуренция за working tree.** Два агента в одном репо = гонки; правило J.1 запрещает branches/worktrees без явного запроса | Параллельность «из коробки» ломает репо | v1: lock per `project_path`, параллельность только между проектами. Worktrees внутри проекта — отдельное решение пользователя (ревизия J.1) |
| R7 | **Хрупкость limit-сигналов.** Форматы ошибок/событий CLI меняются с релизами | Тихая поломка → run висит вечно или долбит API | Contract-тесты адаптеров (запись реальных сэмплов); деградация: если reset неизвестен → консервативный backoff (15м → 1ч → 4ч cap) + alert |
| R8 | **Один компьютер.** Ноут выключен/унесён — всё стоит | Ограничение принимается осознанно | Зафиксировать как non-goal v1; future: Codex cloud / Claude background tasks как off-machine исполнители |
| R9 | **Длинные сессии дороги и хрупки.** Resume через дни = холодный кеш, гигантский контекст | Скрытая стоимость «сессионной» модели | Шаги проектировать короткими вокруг артефактов; сессии переиспользовать только в пределах одного дня/окна |
| R10 | **Подписочная серая зона.** Интенсивная автономная фоновая эксплуатация подписочных лимитов обоих вендоров — допустимая, но «жадная» стратегия | Возможное ужесточение политик вендоров | Quota reserve + рабочие окна по умолчанию; не строить процесс, требующий 100% утилизации окон |

---

## 4. Готовые решения: ландшафт и почему их недостаточно

| Решение | Что даёт | Чего не хватает для задачи |
|---|---|---|
| **Claude Code сам по себе** (headless, subagents, hooks, background tasks, `/loop`, cloud Tasks) | Однo-агентную автономию, очереди, ресюм | Нет Codex; нет cross-agent FSM; нет «ждать днями и проснуться»; cloud — не локально |
| **Claude Agent SDK (Python)** | Программный драйвер Claude Code, RateLimitInfo, resume | Это **библиотека адаптера**, не оркестратор — петлю/state/Codex всё равно писать |
| **Anthropic Managed Agents** (cloud, beta) | Серверные сессии, outcome-loop с грейдером, webhooks | API-биллинг (не подписка Max), облако (не локальная машина), нет Codex |
| **Codex Automations / Codex cloud** | Scheduled background runs, окружения | Только Codex; нет cross-agent контекста; cloud-задачи не видят локальную машину |
| **GitHub Actions (self-hosted runner)** | Триггеры, ретраи, матрицы | Ожидание днями — анти-паттерн (timeout джобов); подписочные креды на раннере; state всё равно вне Actions; Windows interactive session для кред — боль |
| **Temporal** | Лучшая в классе durable execution, sleep на дни | Тяжёлая инфраструктура (сервер+БД+воркеры) на single-user машине; адаптеры и квоты всё равно ваши; против brick-минимализма |
| **Prefect / Dagster** | Flows, retries, schedules | Ориентация на data-пайплайны; daemon-сервер; state-модель не ваша (PM-MCP дублируется) |
| **LangGraph + SQLite checkpointer** | Durable граф с interrupt/resume, человеческие гейты | Рассчитан на API-агентов в процессе, а не внешние подписочные CLI; PM-MCP остаётся вторым SoT; польза ~= ничтожна при готовом PM-MCP |
| **n8n локально** | Low-code workflow, ожидания, веб-UI | Ещё один daemon+UI; интеграция с PM-MCP/правилами/хуками хуже; логика адаптеров всё равно кастомная в нодах |
| **claude-flow / claude-squad / community-оркестраторы** | Мульти-агентные эксперименты | Claude-центричны, незрелы, не Windows-first, не знают про Codex-квоты и ваш PM-контур |
| **PM-MCP (ваш)** | Workflow/batch/portfolio FSM, handoff v1, approval, audit, recommendations | Не запускает агентов (исполнял человек), run state in-memory, нет quota-модели |

**Вывод.** Уникальная комбинация требований — *подписочные CLI двух вендоров + паузы на
часы/дни + Windows single-user + существующий PM-контур и правила* — целиком не
закрывается ни одним продуктом. Покупаемое ядро (Temporal/LangGraph) экономит ~10%
работы (петля и persist), но навязывает второй SoT и инфраструктуру. Дешевле дописать
недостающее к PM-MCP.

### 4.1 Вердикт: что пишем и что НЕ пишем

**НЕ пишем:** state machine с нуля (есть PM-MCP workflow + handoff v1), scheduler-фреймворк
(хватает tick-петли + Task Scheduler), UI (assistant-ui уже показывает paused workflows
и approve-кнопки), memory-слой (AI-memory), очередь задач (PM-MCP batch runs),
UI-автоматизацию приложений (отказ от Claude Desktop).

**Пишем (новое):**
1. Персист workflow runs в SQLite PM-MCP + статусы `waiting_limit` (расширение PM-MCP).
2. Подсистему `agent-runner/`: durable-петля, адаптеры Claude/Codex, quota manager,
   plan loader, kill-switch, notifier.
3. Формат execution-плана (`*.run.yaml`) + загрузчик в PM-MCP.
4. Contract-тесты адаптеров и spike-проверки limit-детекции.

---

## 5. Целевая архитектура

### 5.1 Общая схема

```
                ┌────────────────────────────────────────────────┐
                │  Assistant-UI (наблюдение, approve, kill)      │
                │  dashboard: runs, paused, approvals, quotas    │
                └──────────────▲────────────────▲────────────────┘
                               │ SSE /events    │ MCP tools
                               │                │
┌──────────────────────────────┴────────────────┴───────────────────┐
│  PM-MCP-server  ·  CONTROL PLANE / source of truth                │
│  tasks · goals · workflow_runs(persist!) · steps · gates ·        │
│  agent_sessions · quota_windows · audit_log · events              │
└──────────────▲────────────────────────────────────────────────────┘
               │ MCP (loopback, service bearer ADR-0019)
               │ claim_next_step / report_step_result / set_waiting
┌──────────────┴────────────────────────────────────────────────────┐
│  agent-runner  ·  EXECUTION PLANE (новая подсистема, Layer B)     │
│  tick-петля (stateless между тиками) · Quota Manager ·            │
│  Gate Enforcer · Notifier · kill-switch                           │
│        │                                                          │
│        ├── ClaudeCodeAdapter ── Claude Agent SDK ── claude.exe    │
│        └── CodexAdapter ────── subprocess ───────── codex.exe     │
│                       (exec / exec resume, --output-schema)       │
└───────────┬───────────────────────────────────────────────────────┘
            │ cwd = project repo
   ┌────────▼──────────┐      ┌──────────────┐     ┌──────────────┐
   │ git-репозитории   │      │ run workspace │     │  AI-memory   │
   │ (артефакты кода)  │      │ (артефакты    │     │ (durable     │
   │ commit per AGENTS │      │  процесса)    │     │  контекст)   │
   └───────────────────┘      └──────────────┘     └──────────────┘
```

Принципы (наследуют ADR-0001):
- **Один SoT по вертикали:** процесс/статусы — PM-MCP; код — git; контекст — AI-memory;
  транскрипты — нативные session-сторы агентов (PM-MCP хранит только указатели).
- **Runner stateless:** перезапуск runner = перечитал PM-MCP, восстановил картину,
  продолжил. Никакой второй БД с durable-правдой.
- **Каждый шаг идемпотентен на уровне намерения**: промпт шага формулируется как
  «проверь текущее состояние и доведи до результата X», а не «сделай действие Y».

### 5.2 Компоненты

| Компонент | Ответственность |
|---|---|
| **Runner Core** | Tick-петля: взять у PM-MCP готовые шаги (`claim_next_step`), проверить квоты/гейты/локи, запустить адаптер, дождаться результата, записать (`report_step_result`). Один процесс, asyncio; sleep до ближайшего события (resume_at квоты / поллинг новых runs) |
| **Agent Adapter** (contract) | `start(step, ctx) -> Attempt`, `resume(session_ref, prompt) -> Attempt`, `classify_failure(raw) -> {limit, fatal, retryable}`, `probe_quota() -> QuotaSnapshot?`. Никакой бизнес-логики |
| **Quota Manager** | Состояние окон per provider (`five_hour`, `weekly`, …): `status`, `resets_at`, `utilization`, `observed_at`. Источники: RateLimitInfo (Claude), JSONL-события/`classify_failure` (Codex). Выдаёт `allow / defer(until)` + применяет quota reserve и рабочие часы |
| **Gate Enforcer** | Шаги-гейты: `auto` (по схеме выхода), `agent_review` (кросс-ревью другим агентом), `human` (run → `waiting_approval`, видно в assistant-ui; resume по approve) |
| **Context Broker** | Конвенции артефактов: материализует входы шага (пути), валидирует выходы по контракту (фикс. секции отчёта — переиспользовать `report` contract ADR-0002) |
| **Project Lock** | Один активный run на `project_path` (файловый/БД-лок) |
| **Notifier** | События в PM-MCP audit + SSE (`workflow.step_*`); опционально push (канал — решение пользователя) |
| **CLI** | `agent-runner submit <plan.run.yaml>`, `status`, `pause/resume/kill <run>` — тонкие обёртки над MCP tools |

### 5.3 Размещение и запуск

- Подсистема `agent-runner/` в монорепо AI-Assistant (свой `.venv`, uv, brick #7).
- Запуск — **Layer B**: logon scheduled task (brick #20), т.к. креды Claude/Codex
  per-user (аналог инцидента G:\ session 0). NSSM-fallback бессмыслен — в session 0
  агенты не аутентифицируются.
- Process State: регистрация в PSM (`register_process`), Idle между тиками (brick #18);
  EcoQoS НЕ применять к живым агентским подпроцессам (heavy compute правило).
- Singleton-guard (brick #8) — lock-file, порт не нужен (runner без HTTP; управление
  через PM-MCP tools и CLI).

---

## 6. Модель данных (расширение PM-MCP SQLite)

Новые таблицы в `pm-mcp-server` (через миграцию `schema_migrations`; все — с `tenant_id`
forward-колонкой по brick #3):

```
execution_runs            -- durable замена in-memory WORKFLOW_RUNS для agent-runs
  id, plan_slug, project_path, title,
  status: draft|ready|running|waiting_limit|waiting_approval|blocked|done|failed|aborted
  current_step_id, created_at, updated_at, resume_at (nullable),
  budget_json (max_steps, max_attempts_per_step, quota_reserve_pct, work_hours),
  policy_json (tool allowlist/sandbox per agent), source_plan_path, metadata

execution_steps
  id, run_id, ordinal, name,
  agent: claude|codex|<plugin>,
  session_handle (nullable, напр. "codex:review-1"),
  prompt_ref (путь к шаблону/тексту в run workspace),
  inputs_json (artifact refs), outputs_json (contract + факт. пути),
  gate: auto|agent_review|human,
  loop_json (nullable: max_rounds, until),
  depends_on_json,
  status: pending|ready|running|waiting_limit|waiting_approval|done|failed|skipped,
  attempts_count, last_error_json, started_at, finished_at

step_attempts             -- журнал попыток (recovery + диагностика)
  id, step_id, n, agent_session_id (nullable), pid (nullable),
  started_at, finished_at,
  outcome: ok|limit|error|crash|killed,
  usage_json (tokens/cost если доступно), error_excerpt, transcript_ref

agent_sessions            -- указатели на нативные сессии агентов
  id, run_id, handle, agent, provider_session_id,
  transcript_path, created_at, last_used_at, status: live|stale|closed

quota_windows
  id, provider: anthropic|openai, window_kind: five_hour|weekly|weekly_model|...,
  status: ok|warning|exhausted, utilization_pct, resets_at, observed_at, source
```

Существующий `audit_log` получает события `execution_run.*` / `execution_step.*`
(паттерн ADR-0002 уже таков). Транскрипты и stdout **не** пишутся в БД — только
`transcript_ref` (путь к нативному session-файлу агента / лог-файлу в run workspace).

Runner-локально durable-данных нет; только `data/agent-runner/` под run workspace
(артефакты процесса) и lock-файлы.

**Отдельное решение (кандидат ADR):** судьба существующих in-memory
`WORKFLOW_RUNS`/`BATCH_*`/`PORTFOLIO_*` — мигрировать на те же таблицы (предпочтительно,
migration-discipline) или оставить как есть для интерактивных сценариев и считать
`execution_runs` отдельным контуром. Рекомендация: единый персист, два потребителя.

---

## 7. Формат плана выполнения

Два уровня (соответствует текущему процессу):

1. **Human-план** — как сейчас, `~/.claude/plans/<slug>.md` (central-plan-workflow).
2. **Execution-план** — `<slug>.run.yaml` рядом с планом; загружается командой
   `agent-runner submit` → создаёт `execution_run` в PM-MCP. После загрузки SoT — PM-MCP
   (YAML дальше не читается — никакого двойного источника).

Пример (сценарий пользователя, переписанный с циклом ревью):

```yaml
run:
  title: "Доработка X по плану <slug>"
  project_path: 'D:\GitHub\AI-Assistant'
  plan: 'C:\Users\Zaxva\.claude\plans\<slug>.md'
  budget: { max_steps: 40, max_attempts_per_step: 3, quota_reserve_pct: 25 }
  policy:
    codex:  { sandbox: workspace-write, approval: never-but-gated }
    claude: { permission_mode: acceptEdits, allowed_tools: [Read, Edit, Write, Bash(uv run *), Grep, Glob] }

steps:
  - id: plan-draft
    agent: claude
    session: claude:planning
    prompt: prompts/plan-draft.md          # вход: human-план; выход: draft в plans dir
    outputs: [plan_draft]

  - id: plan-review-loop                   # цикл «ревью → правки» до сходимости
    loop: { max_rounds: 4, until: "review.verdict == approved" }
    body:
      - id: codex-review
        agent: codex
        session: codex:plan-review
        prompt: prompts/review-plan.md     # вход: plan_draft; выход: review.md (schema: verdict+findings)
        output_schema: schemas/review.json # codex exec --output-schema
      - id: claude-revise
        agent: claude
        session: claude:planning
        when: "review.verdict != approved"
        prompt: prompts/revise-plan.md

  - id: human-approve-plan
    gate: human                            # run → waiting_approval, кнопка в assistant-ui

  - id: decompose
    agent: claude
    prompt: prompts/decompose-to-pm.md     # создаёт задачи PM-MCP, link_task_dependency

  - id: implement-queue                    # для каждой задачи пула
    foreach: pm_tasks(umbrella from decompose)
    body:
      - { id: codex-impl,  agent: codex,  session: "codex:task-{task_id}", prompt: prompts/implement.md }
      - { id: verify,      agent: claude, prompt: prompts/verify.md }   # ruff/tests/AGENTS.md M
      - { id: review-gate, gate: agent_review, reviewer: claude }

  - id: final-acceptance
    gate: human
```

Семантика:
- `session:` — именованный handle; адаптер делает resume, если сессия жива, иначе
  стартует новую (шаг обязан быть выполним с нуля из артефактов — §2.2);
- `prompt:` — шаблон с подстановкой **путей** к артефактам, не содержимого;
- `output_schema` — структурный выход (у Codex нативно `--output-schema`, у Claude —
  инструкция формата + валидация Context Broker'ом, при невалидном — один retry);
- линейный список из постановки (шаги 1–11) выражается тривиально: каждый шаг с
  `agent:` + `session:` — формат покрывает исходный пример как частный случай.

---

## 8. Механизм передачи контекста между агентами

Иерархия носителей (от основного к вспомогательному):

1. **Git-репозиторий** — код, диффы, коммиты (по правилам AGENTS.md разд. J/N — коммиты
   делает агент в рамках шага; push — см. §11.3).
2. **Run workspace** (`AI-Assistant/data/agent-runner/runs/<run_id>/`) — артефакты
   процесса: промпты шагов, отчёты, review.md, логи. Вне git (J.3), retention по конфигу.
3. **PM-MCP handoff payload v1** (ADR-0002: `payload`, `recovery_action`, `stop_reason`,
   `autonomy_level`) — формальная передача «кто следующий и что делать».
4. **AI-memory** — durable-выводы на гейтах (как сейчас вручную): `task_context` при
   утверждении плана, `change` после реализации. Пишет агент в рамках шага по протоколу
   Section D, runner сам в память не пишет.
5. **Нативные сессии агентов** — оптимизация контекста соседних шагов одного агента.

Контракт выхода шага: отчёт фикс. структуры (verdict / findings / artifacts / next) —
расширение существующего `report` contract PM-MCP. Вход следующего шага — ссылки на
эти файлы. Прямой межагентный «телефон» (выход A дословно в промпт B) запрещён —
анти-инъекционная мера (R2) и защита от разбухания контекста.

---

## 9. Механизм восстановления после лимитов

### 9.1 Детекция

| Агент | Основной сигнал | Fallback |
|---|---|---|
| Claude | `RateLimitInfo` из Agent SDK: status (`allowed_warning`/`rejected`), `resets_at`, тип окна | classify по тексту ошибки CLI; консервативный backoff |
| Codex | JSONL-события rate limits в `exec --json`; non-zero exit + текст «usage limit»; снапшот окон из status | консервативный backoff (15м → 1ч → 4ч cap) + alert |

Каждый сигнал записывается в `quota_windows` (observed_at, resets_at, utilization).

### 9.2 Поведение

1. Перед запуском шага Quota Manager: `allow | defer(until)`. Defer, если: окно
   `exhausted`; недельная utilization > (100 − quota_reserve_pct); вне рабочих часов
   робота (конфиг, по умолчанию ночь = можно, день = только с резервом).
2. Если попытка упала с `outcome=limit`: attempt журналируется, шаг →
   `waiting_limit`, run → `waiting_limit`, `resume_at = resets_at + jitter(1–5м)`.
3. Тик-петля спит до ближайшего `resume_at`. После пробуждения — повторная попытка
   **той же** попыткой-семантикой (resume сессии, если жива; иначе новая с артефактами).
4. Если reset неизвестен — экспоненциальный backoff с cap 4ч и нотификацией после
   второго цикла.
5. Окна обоих провайдеров независимы: шаги Claude могут идти, пока Codex ждёт (если
   позволяет DAG-зависимость).

### 9.3 Сожительство с человеком (ключевая политика)
Робот и пользователь делят одни подписки. По умолчанию: quota_reserve_pct=25 (робот
не доводит недельные окна выше 75%), и опция «робот работает только в окне 22:00–08:00 +
выходные» — оба параметра в конфиге, требуют выбора пользователя.

---

## 10. Восстановление после перезапуска системы

1. **Автостарт:** logon scheduled task (brick #20, как pm-mcp/assistant-ui);
   pwsh 7 (brick #19); ожидание доступности PM-MCP loopback с retry.
2. **Reconciliation при старте runner:**
   - прочитать из PM-MCP все runs в `running|waiting_limit|waiting_approval`;
   - для steps в `running`: живого процесса нет (PID из последнего attempt мёртв) →
     attempt закрывается `outcome=crash`; верификация фактического состояния:
     (а) выходные артефакты валидны → шаг считается `done`;
     (б) артефактов нет/частичны → новая попытка с промптом-ресюмом («состояние могло
     быть частично применено: проверь X, доведи до Y») + resume сессии, если жива;
   - для `waiting_limit`: пересчитать `resume_at` (окно могло сброситься за время
     даунтайма) → возможно, немедленный старт;
   - для `waiting_approval`: ничего (ждём человека).
3. **Сон/гибернация:** по умолчанию принять «машина спит → run продолжится после
   пробуждения» (resume_at в прошлом обрабатывается немедленно). Опция wake timers
   (`powercfg`/Task Scheduler wake) — отдельное решение пользователя (влияет на
   энергопотребление ноутбука).
4. **Крэш самого runner:** systemd-аналога нет — Task Scheduler restart-policy логон-таски
   (уже паттерн `run-user-session.ps1`); singleton-guard от двойного запуска.

---

## 11. Параллельность, PM-MCP, AI-memory, GitHub

### 11.1 Несколько задач одновременно
- v1: **параллельность на уровне проектов** — несколько активных runs, но max один run
  на `project_path` (Project Lock) и max один живой подпроцесс на провайдера
  (`max_concurrent_per_provider=1`): параллельность внутри провайдера не экономит
  лимиты (окна общие), а риски гонок добавляет.
- Внутрипроектная параллельность (worktrees) — **заблокирована правилом J.1**; вынести
  на решение пользователя как v2 (потребует ревизии AGENTS.md и отдельного ADR).

### 11.2 Интеграция с PM-MCP
- Новые MCP tools (расширение, по паттерну composite tools ADR-0002):
  `create_execution_run`, `get_execution_run`, `claim_next_step`, `report_step_result`,
  `set_step_waiting(kind, resume_at)`, `approve_execution_gate`, `abort_execution_run`,
  `list_resumable_runs`, `record_quota_observation`.
- Runner — MCP-клиент loopback c service bearer (ADR-0019 / план #853 WP-4: runner
  входит в S2S-матрицу как новый principal).
- События `execution_run.*` в SSE `/events` → dashboard assistant-ui обновляется
  существующим механизмом (`domain_event_v1`).
- Статусы задач: шаги implement-цикла двигают PM-MCP задачи теми же tools
  (`start_task`/`complete_task`) — поведение идентично ручному Codex, hooks
  (`task_state.py`) продолжают работать.

### 11.3 Интеграция с GitHub
- Коммиты — внутри шагов агентами по AGENTS.md (N.2: отдельный коммит на логическую
  единицу, по-русски). Runner git сам не трогает (кроме read-only проверок чистоты
  дерева до/после шага).
- **Push/PR — только через human-гейт** (J.2/N.1 «не пушить без явного подтверждения» —
  правило сохраняется; гейт и есть подтверждение). `gh` доступен агентам для PR по
  явному шагу плана.
- GitHub Actions/webhooks в v1 не используются (один компьютер, локальный контур).

### 11.4 Интеграция с AI-memory
- На гейтах «план утверждён» и «run завершён» план предписывает агенту записать
  `task_context`/`change` (протокол Section D, как сейчас вручную) — runner только
  проверяет, что шаг это сделал (запись появилась).
- Recall в начале содержательных шагов — через существующие правила AGENTS.md/hooks
  (агент сам вызывает `get_recent_memory`) — менять ничего не нужно.
- Runner не получает собственного write-доступа в память (граница ADR-0001: operational
  события — в PM-MCP audit, не в память).

---

## 12. Адаптеры агентов

### 12.1 Claude Code
- Через **Claude Agent SDK (Python)** в venv runner'а: `ClaudeSDKClient` /
  `query(prompt, options)`; options: `cwd` (репо), `resume=<session_id>`,
  `permission_mode` + allowedTools из policy, `max_turns`, `output stream-json`.
- Session_id сохраняется в `agent_sessions`; transcript path — указатель.
- Лимиты: подписка на rate-limit события SDK (`RateLimitInfo`); статусы
  `allowed_warning` → записать utilization; `rejected` → outcome=limit + resets_at.
- Hooks глобального слоя действуют и в headless (важно: active-task guard может
  блокировать Edit без start_task — план шага обязан включать start_task, как у
  ручного процесса; это фича, не баг).

### 12.2 Codex
- Subprocess: `codex.exe exec [resume <session_id>] --cd <repo> --json
  [--output-schema <schema.json>] "<prompt>"`; путь к exe резолвится при старте
  (PATH user-сессии → скан `%LOCALAPPDATA%\OpenAI\Codex\bin\*\codex.exe`, выбор по mtime).
- Policy: **не** наследовать глобальный `danger-full-access` — передавать override
  флагами (`--sandbox workspace-write`, approvals по политике run); проверить на spike,
  что флаги перекрывают config.toml.
- Session id — из JSONL-событий запуска / session_index; resume тем же id.
- Лимиты: события rate limits в JSONL; non-zero exit + классификатор текста; снапшоты
  окон писать при каждом запуске.

### 12.3 Добавление новых агентов без изменения ядра
- `AgentAdapter` — Protocol (python): `name`, `start()`, `resume()`,
  `classify_failure()`, `probe_quota()?`, `capabilities` (supports_resume,
  supports_schema, …).
- Регистрация: entry point группа `agent_runner.adapters` либо конфиг-секция
  (`[adapters.gemini] module=...`) — ядро перебирает реестр, план ссылается по `agent:`.
- Требования к кандидату: non-interactive режим с exit code; параметризуемый cwd;
  желателен session resume; различимая ошибка лимита. (Gemini CLI, opencode и т.п.
  проходят по этим критериям.)

---

## 13. MVP и production-версия

### 13.1 MVP (минимальный полезный контур)

Фазы (последовательные, каждая с критерием выхода):

- **Φ0 — Spike адаптеров (де-риск, до любого кода ядра):**
  (а) Claude: headless run + resume + получение RateLimitInfo на этой машине;
  (б) Codex: `exec` + `exec resume` из non-interactive шелла, override sandbox-флагов,
  фиксация формата limit-событий (записать сэмплы для contract-тестов);
  (в) поведение обоих при реально исчерпанном 5h-окне.
  *Выход: задокументированные контракты сигналов в `agent-runner/docs/ADAPTERS.md`.*
- **Φ1 — Персист execution runs в PM-MCP:** таблицы §6, tools §11.2, события SSE;
  миграция/сосуществование с in-memory runs — мини-ADR.
- **Φ2 — Runner core:** tick-петля, sequential-исполнение (без foreach/DAG), 2 адаптера,
  waiting_limit + resume_at, reconciliation при старте, kill-switch, logon task,
  PSM-регистрация.
- **Φ3 — План-формат:** загрузчик `*.run.yaml` (линейные шаги + loop-конструкция +
  human-гейт), Context Broker конвенции, отчёт-контракт.

В MVP **не входит:** foreach по задачам PM-MCP, DAG-зависимости, параллельные runs,
worktrees, UI-редактор планов, авто-push, cost-аналитика, новые агенты, wake timers.

**Acceptance MVP:** реальный сценарий «draft плана (Claude) → ревью-цикл (Codex, до
2 раундов) → human-гейт» проходит end-to-end при: (а) принудительном лимит-стопе
посреди (искусственно/реально) с автопродолжением после сброса; (б) перезагрузке
машины между шагами; (в) kill-switch останавливает за ≤1 тик. Ruff/тесты подсистем
зелёные (M.2/M.3).

### 13.2 Production-версия (после эксплуатации MVP)

- `foreach` по пулу задач PM-MCP + DAG-зависимости шагов; параллельность по проектам.
- Dashboard в assistant-ui: таймлайн run'ов, ссылки на транскрипты, approve/abort,
  квоты-виджет (расширение существующего Tokens & Limits).
- Quota-стратегии: резервирование, прогноз «успеет ли run до конца недельного окна»,
  приоритеты runs.
- Политики прав как декларативный слой (per-project policy в репо, ревизия R1).
- Retry-политики и классификатор ошибок по статистике attempts; отчёты из audit.
- Реестр адаптеров (Gemini CLI и др.), opt-in worktrees (после ревизии J.1 + ADR).
- Off-machine исполнители (Codex cloud / Claude background) как особый тип адаптера —
  когда понадобится «ноут выключен, а работа идёт».

---

## 14. Решения, требуемые от пользователя (перед реализацией)

1. **Топология:** подтвердить `agent-runner/` как новую подсистему AI-Assistant
   (рекомендация) vs модуль внутри pm-mcp-server.
2. **Политика прав unattended** (R1): согласиться на deny-by-default + гейты; отдельно —
   допустим ли `acceptEdits` для Claude и `workspace-write` для Codex без подтверждений.
3. **Quota reserve и рабочие часы робота:** значения по умолчанию (предложено 25% / ночь+выходные).
4. **Worktrees** для внутрипроектной параллельности: пересматривать ли J.1 (v2).
5. **Канал нотификаций:** только assistant-ui/SSE или + push (Telegram/email).
6. **Wake timers:** будить ли ноутбук ради runs.
7. **Судьба in-memory workflow runs** PM-MCP: единый персист (рекомендация) или два контура.

## 15. Кандидаты ADR / bricks / skills (ретроспектива осей — после утверждения)

| Ось | Кандидат | Вердикт |
|---|---|---|
| ADR | «Agent execution plane: PM-MCP control plane + stateless agent-runner» (фиксирует не-повторение PM-Agent) | needs-user-confirmation |
| ADR | «Персист workflow/execution runs в SQLite PM-MCP» | needs-user-confirmation |
| tech-stack brick | «Драйвинг coding-агентов: официальные headless-интерфейсы (Agent SDK / codex exec), не UI-автоматизация» — оформлять **после** реализации (evidence-правило каталога) | follow-up-task |
| skill | `agent-run-plan-authoring` (как писать `*.run.yaml`) — после стабилизации формата | follow-up-task |
| hooks | нет нового детерминированного триггера на этом этапе | no-change |

## 16. Верификация дизайна

- Φ0-spike — главная проверка гипотез (limit-сигналы, resume, override política Codex).
- Contract-тесты адаптеров на записанных сэмплах вывода CLI (без сети) — `uv run pytest`
  в `agent-runner/`.
- Runtime-verification по brick #17: tier-2 ephemeral instance PM-MCP с временной БД для
  тестов новых tools; никаких писем в живые данные.
- Конец-в-конец прогон acceptance-сценария MVP (см. §13.1) на реальных подписках в
  ночном окне.

---

*Документ подготовлен по запросу «глубокий архитектурный анализ + проектное предложение,
без кода». Все утверждения о существующем коде проверены чтением репозитория в сессии
2026-06-12; утверждения о CLI сверены с официальной документацией (ссылки в §1.3).*
