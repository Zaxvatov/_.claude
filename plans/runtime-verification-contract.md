# Runtime Verification Contract — изолированный прогон UI и обменов

Статус: **Phase 1 — conditional greenlight (пилот); Track B — согласованное направление, реализация отложена.**
Дата создания: 2026-06-02. Ревизия 2026-06-02: учтены замечания Codex (orphan-check, task split, scenario-таблица, helper-тест, canonical data-path).
PM-MCP: **#779** (assistant-ui, Phase 1). Track B / Phase 3 — отдельные задачи при запуске (K.4).

## Контекст

Повторяется один и тот же класс сбоев при поднятии окружения для проверки задачи —
причём по обе стороны системы:

- **UI-сторона.** В сессии Codex по `/budget` фоновый `uvicorn` не смог занять `8000`
  (его уже держал постоянный NSSM-сервис `AI-Assistant-Assistant-UI`), и проверка
  молча пошла против **живого сервиса**, а не против правок. Пронесло только потому,
  что фикс был в Design-system-ассетах (отдаются статикой live); будь он в Python/шаблоне —
  verification проверял бы старый код. Сопутствующее: ненадёжный выбор порта
  (`Get-NetTCPConnection` не вернул владельца), сломанный quoting inline-PowerShell,
  ручной cleanup, tmp-скрипт в корне.
- **Exchange-сторона.** Инциденты 2026-05-22 (ai-memory: 4 параллельных python-процесса,
  порт `8765` не слушался) и 2026-06-01 / `#762` (сервис в Running, `8766` отказывает,
  висящие `python.exe`, stale bootstrap удалял текущие сервисы). Тот же корень: неясно,
  где живой daemon, а где наш прогон; singleton-guard (#8) не даёт поднять второй instance
  на том же порту; за собой остаются мусорные процессы.

Вывод, к которому пришли в обсуждении: **это не две задачи, а одна дисциплина**
runtime-verification с двумя реализациями (UI и обмены). Значительная часть UI-контура
уже существует и её не надо строить заново:

- skill `frontend-verification` (`D:\GitHub\_engineering_rules\skills\frontend-verification\SKILL.md`)
  — процедура проверки + HTTP-smoke fallback + требование честно писать, что не проверено;
- hook `frontend_verification_reminder.py` — уже реагирует на `templates/static/assets`
  и **только напоминает**, ничего не «чинит» (это и есть верное поведение hook'а);
- де-факто практика изолированных портов (`8013–8023` в прошлых сессиях) — но не закреплена,
  поэтому Codex от неё отклонился.

Existing exchange-активы: `runtime_contract.py` + `EXPECTED_TOOLS` (brick #5), singleton-guard
(brick #8), health-эндпоинты (`/api/health/mcp`, `/health`), namespaced env vars (brick #4),
hybrid consumption direct+MCP (brick #14).

## Цель

Единый контракт runtime-verification и конкретные per-subsystem helper'ы, которые:

1. поднимают **изолированный** instance проверяемого компонента;
2. прогоняют его через **детерминированный primary path** (Playwright для UI, ASGI/MCP-roundtrip для обменов);
3. **никогда** не трогают и не убивают живой NSSM-сервис и его данные;
4. в `finally` гасят **только то, что сами подняли**, и складывают артефакты в предсказуемую ignored-папку;
5. честно сообщают, что не удалось проверить.

## Общий принцип (инвариант обоих треков)

- **Никогда не использовать канонический порт** живого сервиса для проверочного instance
  (8000 / 8765 / 8766 / 8767 / 8770 / 8780). Только ephemeral `port=0` либо изолированный порт.
- **Никогда не писать в живые данные.** Любой write-path verification идёт в временную БД /
  временный data-dir через namespaced env (brick #4). Helper отказывается стартовать, если
  указан канонический data-dir.
- **Детерминированный primary path**, а не браузер-плагин / ручной curl как единственный механизм.
- **Teardown только своего PID** (или своего in-process сервера). Чужой процесс не трогаем —
  ни kill, ни restart.
- **Артефакты предсказуемы и ignored** (`logs/verify/...` — уже покрыто `.gitignore`).
- **Honest reporting**: что прошло и что не проверено — явно в финале.

## Track A — UI verification (пилот в assistant-ui, обобщаемо на любой UI-проект)

Согласованный дизайн:

- Команда: `uv run python scripts/verify_frontend.py` (публичный стандарт — `uv run`);
  внутри для дочернего `uvicorn` — `sys.executable` (наследует `.venv`, без хардкода пути → не нарушает E.3).
- Порт: **ephemeral `port=0`**. Архитектура — **subprocess** `uvicorn app.main:app --port 0`,
  порт читаем из строки `Uvicorn running on http://127.0.0.1:<port>` в выводе процесса.
  (Гонки нет — uvicorn держит сокет всё время. In-process вариант рассмотрен и отклонён:
  чище по cleanup, но расходится с реальным boot и тащит app в процесс проверки.)
- Изолируем **только web-процесс** Assistant-UI. Бэкенды (PM-MCP `8766`, budget `8767`,
  AI-memory) helper не поднимает — переиспользует живые loopback-сервисы, как и `8000`.
  Проверка зависимостей — **по сценарию**, а не общим списком (см. таблицу): иначе ложное
  «бэкенд недоступен» там, где страница рендерится direct-import'ом (brick #14).

| Сценарий | Источник данных primary-path | Обязательный live-бэкенд |
|----------|------------------------------|--------------------------|
| `/budget`, `/settings` | direct import `budget.*` (brick #14) → `budget.db` | budget MCP `8767` **не** требуется |
| `/dashboard` | PM-MCP proxy (`get_dashboard_snapshot`, work-items) | PM-MCP `8766` |
| `/overview` | PM-MCP `get_portfolio_overview` | PM-MCP `8766` |
| `/memory` | AI-memory (active memory, proposals) | AI-memory (stdio child / `8765`) — тянет cold-start эмбеддингов |
| `/console` | render-only smoke | нет (глубокий чат с LLM — вне smoke) |

  Helper для каждого сценария проверяет, что данные реально пришли (непустой ключевой
  контейнер), а не только HTTP 200.
- PID / URL / логи / скриншоты → `logs/verify/<scenario>/` (ignored).
- Сценарии — постоянные в `scripts/`, не tmp-файлы в корне. Старт: `/budget`, `/dashboard`
  (самые активные); структура — чтобы `/overview`, `/console`, `/memory`, `/settings`
  добавлялись одной функцией каждый.
- Hook — без изменений (существующий reminder достаточен). AGENTS.md — расширить «Final validation».
  Skill `frontend-verification` — одна строка («для Assistant-UI — project helper, Playwright primary,
  не ad-hoc сервер») — **в Phase 3** (репозиторий `_engineering_rules`, отдельная задача по K.4), не в пилоте.

## Track B — Exchange verification (MCP-транспорты / gateway / cross-subsystem)

Новый контур. Ключевое отличие от Track A: **обмены пишут**, поэтому к изоляции порта
добавляется изоляция данных. Три уровня (от дешёвого к дорогому):

1. **In-process / ASGI (primary, без порта).** Тесты контракта и схемы tools гоняются через
   `httpx.ASGITransport` / Starlette TestClient без `bind` — нет коллизии портов, нет
   singleton-guard, нет orphan-процессов, нет cleanup. Сверяем `EXPECTED_TOOLS` из
   `runtime_contract.py`, форму request/response. Во многом это уже текущий `uv run pytest`.
2. **Ephemeral-port instance с изолированными данными (transport smoke).** Когда нужен
   настоящий транспорт (native MCP streamable HTTP handshake, client↔server roundtrip,
   gateway↔backend) — поднимаем instance на `port=0` с временной БД / временным data-dir
   через namespaced env. Singleton-guard (#8) не срабатывает, т.к. порт и данные другие.
3. **Read-only probe против ЖИВОГО сервиса (drift/health).** Только health + сверка
   `runtime_contract` (порт/путь/набор tools) с тем, что реально отвечает. **Без writes.**
   Это единственный разрешённый контакт с живым daemon.

Жёсткие правила Track B:

- Канонические порты (8765/8766/8767/8770/8780) — только для tier-3 read-only probe.
  Для tier-2 — ephemeral.
- Write-tools (`create_task`, `store_memory`, `set_process_state`, `delete_*`) — **только**
  против временных данных tier-2.
- **Canonical data-path detection.** Эффективный data-path резолвится из config-модуля /
  `runtime_contract` / namespaced env (brick #4) каждого subsystem (pm-mcp-server — task-SQLite;
  ai-memory — `data/memory.db`; budget — `budget.db`). Helper сравнивает эффективный путь с
  каноническим и **отказывается стартовать при совпадении**. Точные env-ключи перечисляются
  в спеке Phase 2. Проверка — **до** старта instance.
- Teardown только своего: остановить свой PID, дождаться своих child-PID, проверить, что
  освободился **свой** порт. Чужие python-процессы не сканируем и не трогаем.
- Первый helper — `pm-mcp-server/scripts/verify_exchange.py` (самый активный обмен после
  native-MCP миграции). Остальные (ai-memory, gateway, budget) — по образцу, позже.

## Раскладка по слоям (по вашей же сепарации)

| Что | Куда |
|-----|------|
| Жёсткие инварианты (порт, данные, teardown, артефакты) | `Final validation` в local `AGENTS.md` каждого затронутого subsystem |
| Helper'ы | `<subsystem>/scripts/verify_*.py` + обязательный тест по обычным правилам runtime-кода (root AGENTS.md M.2/M.3: ruff + pytest). `hook-authoring` — только если появится новый reminder-hook |
| Процедура UI | существующий skill `frontend-verification` (+1 строка — в Phase 3, `_engineering_rules`) |
| Процедура обменов | **кандидат**: новый skill `exchange-verification` _или_ обобщить во `runtime-verification` (открытый вопрос) |
| Reminder при изменении транспорта/контракта | **кандидат**: sibling reminder-hook (только напоминание) — не делать без отдельного подтверждения |
| Общий принцип как повторяемый выбор | **кандидат**: новый brick в `tech-stack-choices.md` (#17 «Runtime verification») — предлагать только после пилота |

## Этапы

- **Phase 1 — пилот Track A (assistant-ui).** Одна PM-MCP задача `assistant-ui`: `verify_frontend.py`
  + тест + правка `Final validation` в `assistant-ui/AGENTS.md`. Строку в skill `frontend-verification`
  в Phase 1 **не трогаем** — это отдельный репозиторий `_engineering_rules` (отдельная задача по K.4);
  переносим её в Phase 3 вместе с правками skill под Track B.
- **Phase 2 — reference Track B (pm-mcp-server).** `verify_exchange.py` tier-1 + tier-2 с изоляцией
  данных; правка `Final validation` в `pm-mcp-server/AGENTS.md`. Отдельная PM-MCP задача `pm-mcp-server`.
  **Реализацию не начинаем, пока не расписан отдельный спек** `verify_exchange.py` (временная БД +
  проверка write-tools + canonical-path guard).
- **Phase 3 — обобщение.** Перенести на ai-memory / gateway / budget; решить судьбу skill
  (`exchange-verification` vs `runtime-verification`) и tech-stack brick #17. По задаче на subsystem (K.4).

## Риски

- **Порча живых данных** (главный риск Track B). Митигация: обязательные временные пути +
  refuse-on-canonical-path guard в helper'е.
- **Cold-start ai-memory** (sentence-transformers ~45 c, brick #2/#8) делает tier-2 для памяти
  медленным. Митигация: `deploy-service` runtime-profile / стаб эмбеддингов для smoke
  (паттерн уже есть — memory `#072`).
- **Over-engineering.** Соблазн построить параметризованный фреймворк сразу. Митигация:
  один helper на трек, обобщение после третьего случая.
- **Singleton-guard.** Instance на каноническом порту был бы отклонён (#8). Снимается тем,
  что tier-2 всегда ephemeral + изолированные данные — guard не срабатывает.

## Критерии приёмки

- **Track A (Phase 1).** Helper существует; ephemeral-порт; `finally`-teardown; артефакты ignored;
  контракт в `assistant-ui/AGENTS.md`; тест на helper. Повторный прогон при живом `8000`
  **не** коллизит и **не** проверяет `8000`. (Строка в skill `frontend-verification` — критерий Phase 3.)
- **Track B (pm-mcp-server).** Helper существует; **до** старта instance проверено, что data-path
  временный (не канонический); tier-2 поднимает instance с временной БД; сверяет `EXPECTED_TOOLS`
  + реальный streamable-HTTP roundtrip; после прогона **свой** порт свободен и **свои** child-процессы
  завершены (чужие не проверяем); живая task-БД не изменилась.

## Открытые вопросы (для обсуждения)

1. **Очерёдность.** Сначала пилот Track A (полностью спроектирован), потом Track B — или оба параллельно?
2. **Изоляция данных Track B.** Временная БД на каждый прогон (чисто, но cold-start) vs выделенный
   постоянный `verify`-data-dir (быстрее, но нужен reset)?
3. **Tier-3.** Разрешаем read-only probe против живого сервиса — или вообще запрещаем любой контакт
   с живыми daemon'ами в verification?
4. **Skill.** Новый `exchange-verification` — или обобщить `frontend-verification` в зонтичный
   `runtime-verification`?
5. **tech-stack brick #17.** Фиксировать общий принцип как кирпичик сейчас — или после пилота?
6. **ai-memory cold-start.** Для smoke стабить эмбеддинги / использовать deploy-service profile?

## Не входит в план (anti-scope)

- Не строим параметризованный verification-фреймворк на старте.
- Не добавляем новые hooks без отдельного подтверждения (существующий reminder остаётся reminder'ом).
- Не трогаем CI: на single-user машине CI нет, всё локально через `uv run`.
- Не выносим helper'ы в `_engineering_rules` раньше времени — пока остаются per-subsystem.
