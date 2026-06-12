# Google Calendar read-only sync для /roadmap

PM-MCP: #818 T1 · #819 T2 · #820 T3 · #821 T4 · #822 T5 (pm-mcp-server) · #823 T6 (pm-mcp-server) · #824 T7 (gateway) · #825 T8 (assistant-ui). Зависимости проставлены.

## Контекст

Карте целей `/roadmap` (план `ai-assistant-happy-coral`, time-anchored intent
canvas до 2040) нужны реальные временные якоря из Google Calendar. Сейчас в
`pm-mcp-server` уже есть **каркас** календарной интеграции, но он полностью
не подключён:

- `app/adapters/calendar.py` — чистый adapter с dependency-injected
  `search_events` / `create_event`. Без инжекта клиента `calendar_context()`
  возвращает пусто, а `create_calendar_event` бросает
  `calendar_write_not_configured`.
- `app/config.py` — env `PM_MCP_CALENDAR_IDS`, `PM_MCP_CALENDAR_LOOKAHEAD_DAYS`.
- `server.py` — `WorkItemDomain` включает `"calendar"`, `_policy_calendar_context`
  (server.py:3266) подмешивает занятость в scoring задач (server.py:3324),
  tools `create/update/delete_calendar_event` (server.py:3905+).
- Реального Google-клиента, OAuth и хранилища нет нигде.

Цель: подключить **read-only** чтение всех календарей одного Google-аккаунта,
персистить события локально, инкрементально синхронизировать и отдавать их
в `/roadmap` как временные якоря. Любое изменение в Google Calendar должно
**автоматически** доходить до ассистента.

Решения пользователя (2026-06-05):
- **Один Google-аккаунт.** `calendarList.list` под одним аккаунтом возвращает
  все календари: primary, вторичные, подписанные и shared. Одна OAuth-авторизация.
- **Near-instant (push).** Нужен `events.watch` (webhook), не только поллинг.

## Цель и не-цели

**Цель (этот план):**
- Read-only чтение всех календарей одного аккаунта (scope `calendar.readonly`).
- Двухуровневый sync: `calendarList.list` + per-calendar `events.list` с `syncToken`.
- Локальное хранилище событий + автоматическое распространение изменений в `/roadmap`.
- Push через `events.watch` → webhook (входит через `gateway`) → инкрементальный sync.
- Поллинг как надёжный fallback и для renew watch-каналов.

**Не-цели (явно вне scope):**
- Создание/редактирование/удаление событий из ассистента. Существующие
  `create/update/delete_calendar_event` остаются **не подключёнными** (dormant),
  write-client не инжектится. Read-only scope.
- Несколько Google-аккаунтов (схема готовится account-aware, но реализуем один).
- Двусторонний sync.

## Архитектурные решения

1. **Дом интеграции — `pm-mcp-server`, без нового daemon.** Adapter, config,
   `WorkItemDomain="calendar"`, scoring-хуки, lifespan FastAPI
   (`app/http_transport.py:39`), SSE event bus и зависимость `keyring` уже там.
   Новый публичный daemon запрещён (ADR-0001: «нельзя регистрировать новый
   публичный daemon мимо gateway»). Это **осознанное отклонение от совета
   ChatGPT** про отдельный `calendar-sync` сервис — он противоречит
   ADR-0001 D-8 (минимизация daemon) и brick #12.

2. **Push поверх incremental sync, не вместо.** Google push не несёт тело
   события — webhook лишь триггерит `events.list` по `syncToken`. Поэтому ядро —
   incremental sync; push снижает задержку; поллинг — fallback + renew каналов.

3. **Webhook входит через `gateway` (ADR-0001: единственный внешний ingress).**
   Google зовёт публичный HTTPS gateway (Tailscale Funnel, валидный cert),
   gateway валидирует `X-Goog-Channel-Token` (наш секрет) + resource id, затем
   loopback-вызовом триггерит sync в `pm-mcp-server`. Это новый класс ingress
   (не OAuth/scope, а channel-token) → фиксируется в **ADR-0015**.

4. **Хранилище — выделенный `calendar.db` (brick #3 SQLite/WAL),** отдельно от
   tasks DB, по образцу `app/tasks_db.py` (`SCHEMA_VERSION`, `CREATE TABLE IF
   NOT EXISTS`, индексы, путь из `app/config.py`). События НЕ пишутся в таблицу
   `tasks` — это не actionable-задачи, а внешние якоря.

5. **Распространение изменений** — `publish_event("calendar.events_changed",
   payload, domain="calendar")` (`app/events.py`) → SSE `/events?domain=calendar`
   → `/roadmap` рефетчит. Полная цепочка «автоматически»:
   poller/webhook → sync → store → EVENT_BUS → SSE → UI.

6. **Sync — raw events, exact incremental (не `singleEvents` + window).**
   Full sync: `events.list` с `singleEvents=false`, `showDeleted=true`,
   замороженным `timeMin`-якорем (напр. сегодня−90д), БЕЗ `timeMax`/`orderBy`/`q`/
   `updatedMin`; пагинация до последней страницы (`nextSyncToken` только там).
   **Incremental несёт те же stable-параметры**, что и full sync — `singleEvents=false`,
   `showDeleted=true`, `maxResults`, `timeZone`, опц. `eventTypes` — плюс `syncToken`/
   `pageToken`; запрещены `timeMin/timeMax/orderBy/q/updatedMin` (иначе 400).
   `410 Gone` → очистить `sync_token` этого календаря + заново full sync.
   **Recurring masters**: при `singleEvents=false` master приходит, если его
   эффективный конец (последний instance / unbounded) ≥ `timeMin`, т.е. серии с
   будущими instances попадают; **spike перед T4** это подтверждает, иначе fallback —
   masters-pass без `timeMin` либо проекция `events.instances()`. Recurrence
   разворачиваем на READ-time в окне `/roadmap` (`recurring-ical-events`/`dateutil`)
   с cap (см. read-tool), не храним бесконечные instances; periodic re-baseline
   двигает `timeMin` и чистит старьё. Отклонён `singleEvents=true`+window: syncToken
   замораживает окно (не тянет до 2040) и раздувает instances для recurring/public.

7. **Несколько доверенных + несколько публичных календарей (один аккаунт).**
   Источники = `calendarList.list` (фильтр `PM_MCP_CALENDAR_SELECTED_IDS`) ∪ явные
   public IDs `PM_MCP_CALENDAR_PUBLIC_IDS` (публичные календари не в подписке).
   Старый `PM_MCP_CALENDAR_IDS` мигрируется на эти два (T1). `calendar_sources`
   несёт `access_role` (owner/writer/reader/freeBusyReader) и `trust_tier`
   (trusted|public; default: owner/writer→trusted, иначе public). `freeBusyReader`
   → только занятость без деталей. Tier и `selected` управляют включением и уровнем
   детализации в `/roadmap` (см. privacy в Phase C).

## Хранилище (`calendar.db`)

- `calendar_sources` — id, google_calendar_id (UNIQUE), summary, access_role
  (owner/writer/reader/freeBusyReader), trust_tier (trusted|public), primary flag,
  selected, color, time_zone, synced_at.
- `calendar_events` — **raw** events (без client-side expansion): id, source_id
  (FK), google_event_id, ical_uid, status (confirmed/tentative/cancelled), summary,
  location, description, start, end, all_day, recurrence (RRULE/RDATE),
  recurring_event_id, original_start_time, updated, html_link, etag, visibility,
  synced_at. Cancelled → **tombstone** (status=cancelled), не удаляем молча —
  иначе incremental рассинхрон. Recurrence разворачивается на read-time.
- `calendar_sync_state` — source_id (FK, UNIQUE), sync_token, baseline_time_min,
  last_full_sync_at, last_incremental_sync_at, last_status, failure_count.
- `calendar_watch_channels` (Phase B) — channel_id (UNIQUE), resource_id, source_id
  (FK), **token_hash**, token_hint, expiration, last_message_number, last_seen_at,
  created_at, stopped_at, failure_count. Сам секрет НЕ хранится открыто — нужен
  только при создании канала и не сохраняется.

Индексы: `(source_id, start)`, `(start, end)` для range-запросов `/roadmap`;
`(status)` для фильтра tombstone. Retention/cleanup: prune events с `end` раньше
`baseline_time_min` на re-baseline.

## Этапы

### Phase A — Sync core + store + read-client (pm-mcp-server, полностью локально)

Самодостаточный и shippable без push: поллинг уже даёт «автоматически».

- **Зависимости:** `uv add google-api-python-client google-auth google-auth-oauthlib recurring-ical-events`.
- **`app/calendar_client.py`** — read-only Google клиент. OAuth installed-app
  flow (`google-auth-oauthlib` `InstalledAppFlow`, loopback redirect, разовое
  согласие), refresh-token в keyring (по образцу `app/secrets.py`), scope
  `https://www.googleapis.com/auth/calendar.readonly`. API: `list_calendar_list()`;
  два явных режима — `list_events_full(calendar_id, time_min, *, page_token=None)`
  (full sync: `timeMin` без `timeMax` + stable params) и
  `list_events_incremental(calendar_id, sync_token, *, page_token=None)`
  (только `syncToken` + те же stable params).
- **`app/calendar_store.py`** — SQLite store по паттерну `tasks_db.py`
  (`SCHEMA_VERSION`, схема выше, upsert по google_event_id, range-выборка).
- **`app/calendar_sync.py`** — движок (точный контракт, см. решение 6):
  - `sync_calendar_list()` → upsert `calendar_sources` из `calendarList.list`,
    отфильтрованных `PM_MCP_CALENDAR_SELECTED_IDS`, ∪ `PM_MCP_CALENDAR_PUBLIC_IDS`;
    проставить `access_role`/`trust_tier`.
  - `full_sync(source)` — `events.list` с `singleEvents=false`, `showDeleted=true`,
    `timeMin=baseline_anchor`, без `timeMax`/`orderBy`/`q`; пагинация до конца;
    `nextSyncToken` только с последней страницы; upsert raw events (+ masters).
  - `incremental_sync(source)` — те же stable-параметры + `syncToken`(+`pageToken`);
    `410` → очистить токен и `full_sync`. cancelled → tombstone.
  - `sync_all()` / re-baseline (advance `timeMin`, prune old); при изменениях —
    `publish_event("calendar.events_changed", ...)`.
  - `expand_occurrences(events, window, cap)` — read-time разворот recurrence
    (`recurring-ical-events`/`dateutil`) в окне `/roadmap`, с cap per-source.
- **Поллер** — fire-and-forget `asyncio.create_task` в lifespan
  `app/http_transport.py` (`except BaseException`, урок hotfix #758 / memory
  1529), интервал `PM_MCP_CALENDAR_POLL_SECONDS` (default 600). brick #12.
- **Read-tool** — `list_calendar_events(time_min, time_max, calendar_ids=None)`:
  нормализованные, **recurrence-expanded** и **privacy-shaped** события для
  `/roadmap` (см. Phase C). **Cap** per-source max occurrences/events + diagnostics
  `truncated=true` при усечении (горизонт до 2040). Имя — в `EXPECTED_TOOLS`
  (`app/runtime_contract.py`) и в `PM_MCP_TOOL_NAMES` fallback Assistant-UI
  (`assistant-ui/app/mcp_client.py`, brick #5).
- **Подключить `calendar_context()`** — инжектить store-backed `search_events`
  в `_policy_calendar_context` (server.py:3266). Бонус: оживают busy-blocks /
  next-event / available-window в scoring задач (server.py:3324) без отдельной работы.
- **Разовый auth CLI** — `python -m app.calendar_auth` (consent + сохранение токена).

### Phase B — Push (events.watch + gateway webhook), near-instant

Зависит от Phase A (движок sync + store + таблица каналов).

- **pm-mcp-server `app/calendar_watch.py`** — start/stop/renew `events.watch`
  на каждый календарь; генерим per-channel секрет, храним только `token_hash` +
  `token_hint`; **stop старого канала при renew** (`channels.stop`); renew до
  expiry внутри поллера. Loopback internal route (напр. `POST /internal/calendar/
  notify`, БЕЗ публичной выдачи) — **здесь** авторитетная валидация триплета
  `X-Goog-Channel-Id` + `X-Goog-Resource-Id` + `X-Goog-Channel-Token` (constant-time
  против `token_hash` в своём store), обработка `X-Goog-Resource-State`
  (`sync`/`exists`/`not_exists`), idempotency по `X-Goog-Message-Number`
  (`last_message_number`), затем incremental sync для resource.
- **gateway** — `POST /webhooks/google-calendar` = **тонкий edge**: rate-limit,
  audit, базовая проверка формы заголовков, форвард заголовков на PM-MCP internal
  route. `200` возвращается **только после** того как PM-MCP принял/поставил
  уведомление в очередь; если PM-MCP недоступен → `503` (Google сделает retry,
  push-сигнал не теряется). **Gateway НЕ читает `calendar.db`** и не валидирует
  токен сам (subsystem boundary — `token_hash` живёт в PM-MCP store). Тело Google
  пустое → наружу никаких данных. Новый класс ingress (не OAuth/scope) → ADR-0015.
- **Tailscale Funnel** — публичный HTTPS-путь webhook gateway. **Preflight**:
  `tailscale funnel status --json`, проверка публичного URL, MagicDNS и
  node-attribute/policy; задокументировать включение и **rollback/off** snippet
  в `gateway/README.md` + `tools/`. Поллер — fallback при недоступности Funnel.

### Phase C — Якоря в /roadmap (assistant-ui)

Зависит от Phase A read-tool (T5).

- **`assistant-ui/app/roadmap.py`** — новый node kind `"event"` (time anchor):
  `{kind:"event", date:start, end, title, calendar, all_day, status, tier}`.
  Отдельная дорожка/стиль; НЕ участвует в goal/task scoring.
- **Privacy (acceptance):** по умолчанию только title (или «Занято», если summary
  скрыт/`visibility:private`/freeBusy). НЕ отдавать наружу `description`,
  `location`, `html_link` без отдельного решения. Public-tier — title-or-busy.
  Privacy-shaping делает read-tool на стороне pm-mcp-server (T5), UI не получает лишнего.
- **`assistant-ui/app/main.py`** `/api/roadmap` — дотягивать события через
  `mcp_client` (`list_calendar_events` в диапазоне `range`) и мержить в view model.
- **frontend (React Flow island)** — read-only маркеры, визуально отличные от
  goals/tasks; tier влияет на плотность/детализацию.
- **Live update** — SSE `/events?domain=calendar`; на `calendar.events_changed`
  — рефетч `/api/roadmap`.

**Порядок:** A → (B ∥ C). Push (B) пользователь подтвердил как обязательный.

## PM-MCP задачи (создать после approval; K.4 — отдельный work item на подсистему)

Phase A (pm-mcp-server) — T1 разбит, чтобы не мешать ADR/OAuth/БД/sync в одном коммите:
- **T1:** ADR-0015 + config/env (`PM_MCP_CALENDAR_SELECTED_IDS` /
  `PM_MCP_CALENDAR_PUBLIC_IDS` + миграция legacy `PM_MCP_CALENDAR_IDS`, poll
  interval, baseline anchor, cap) + credential/token contract (keyring vs
  restricted-file под учётку NSSM). Блокер для всего.
- **T2:** `calendar_store.py` + migrations (schema, индексы, tombstone, retention/
  re-baseline). Зависит T1.
- **T3:** `calendar_client.py` (read-only) + auth CLI (InstalledAppFlow, token по T1).
  Зависит T1.
- **T4:** `calendar_sync.py` (full/incremental/410/tombstone/re-baseline) +
  recurrence expansion (read-time, cap) + поллер (lifespan) + `publish_event` +
  тесты, включая **spike**: recurring series до baseline → future instance виден.
  Зависит T2, T3.
- **T5:** read-tool `list_calendar_events` (recurrence-expanded, privacy-shaped) +
  EXPECTED_TOOLS + store-backed `calendar_context()`. Зависит T4.

Phase B:
- **T6 (pm-mcp-server):** `calendar_watch.py` (start/stop/renew, `channels.stop`
  старого, `token_hash`) + loopback internal route с авторитетной валидацией
  триплета/states/idempotency → incremental sync. Зависит T4.
- **T7 (gateway):** тонкий webhook-edge (rate-limit, audit, форма заголовков,
  форвард на PM-MCP internal route; НЕ читает store, НЕ валидирует токен) + Funnel
  preflight/rollback. Зависит T6.

Phase C:
- **T8 (assistant-ui):** event-node в roadmap + `/api/roadmap` merge + frontend +
  SSE live + privacy-shape. Зависит T5.

Линковать зависимости `link_task_dependency` (cross-subsystem — с
`dependency_project_path`).

## Переиспользуем (не изобретать)

- `app/adapters/calendar.py` — `read_calendar_work_items`, `calendar_context`,
  `get_today_busy_blocks`, `get_available_window`. Кормим store-backed `search_events`.
- `app/tasks_db.py` — паттерн SQLite store (схема/версии/индексы/путь из config).
- `app/events.py` — `publish_event` / SSE `/events` (домен из префикса `calendar.`).
- `app/secrets.py`, dep `keyring` — хранение refresh-токена.
- `app/http_transport.py` lifespan — точка запуска поллера (brick #12).
- `assistant-ui/app/roadmap.py` `roadmap_view_model` — точка добавления event-нод.

## Безопасность

- Scope строго `calendar.readonly`; write-tools не подключаем.
- Refresh-token: keyring **или** restricted-file (ADR-0001). **Риск:** NSSM-сервис
  и интерактивный пользователь могут иметь разный keyring-контур (см. Риски) —
  выбрать хранилище под учётку сервиса; по умолчанию restricted-file fallback.
- Webhook: Google не шлёт тело события → наружу никаких данных. Авторитетная
  валидация триплета channel_id+resource_id+token (**constant-time** против
  `token_hash`, открытый токен не храним) + idempotency по message-number — **в
  PM-MCP** (его store). Gateway — тонкий edge (rate-limit/audit/форма заголовков),
  НЕ читает `calendar.db` и не валидирует токен (subsystem boundary). Channel-token
  не несёт чувствительных данных.
- Privacy: private/freeBusy события — только занятость/якорь; UI не получает
  `description`/`location`/`html_link` без отдельного решения.
- OAuth client secrets — вне кода, через env/keyring (E.3: без хардкода путей).

## Tech-stack brick кандидат + ADR (предлагать ТОЛЬКО после approval)

- **Brick кандидат:** «External Google API: `google-api-python-client` +
  `google-auth-oauthlib`, installed-app OAuth, read-only scope, refresh-token в
  keyring/restricted-file». Нет в `tech-stack-choices.md`. Evidence — новые
  модули. (Паттерн incremental-sync `syncToken` + poll + push-webhook —
  возможный будущий brick, обобщать после 3-го кейса, не сейчас.)
- **ADR-0015** (следующий свободный; 0014 — последний): Google Calendar
  read-only sync — `calendarList.list` + `events.list` syncToken + `events.watch`
  через gateway webhook. Фиксирует webhook как новый класс ingress.

## Риски

- **Watch-каналы истекают** (макс ~7 дней, часто меньше) → renew в поллере;
  без renew push молча умирает (ловит fallback-поллинг).
- **`syncToken` инвалидация (410)** → очистить токен календаря + full sync; не
  терять состояние других календарей.
- **Recurring / volume** — храним raw events, разворачиваем на read-time в окне
  (не бесконечные instances); public/holiday recurring могут быть объёмными —
  cap per-source + diagnostics `truncated` + re-baseline cleanup.
- **Recurring master до baseline** — серия, начатая до `timeMin`, но с будущими
  instances, должна попасть в full sync; проверяется spike (T4); fallback —
  masters-pass без `timeMin` или `events.instances()` projection.
- **all-day / TZ** — start/end с tz; default `Europe/Zurich` (как в adapter).
- **Cancelled tombstones** — incremental отдаёт `status=cancelled`; трактовать как
  удаление-якоря, иначе stale.
- **keyring под NSSM** — токен доступен учётке сервиса; разовый auth под той же
  учёткой либо restricted-file. Решить в T1.
- **Tailscale Funnel / cert** — при недоступности push не работает; поллинг держит свежесть.

## Критерии приёмки

- `sync_all()` наполняет `calendar_sources` всеми календарями (trusted + public) и
  `calendar_events` raw-событиями; повторный запуск — exact incremental по
  `syncToken` (incremental-запрос без запрещённых `timeMin/timeMax/orderBy/q/
  updatedMin`, но со stable params из initial sync; `nextSyncToken` с последней
  страницы; `410` → clear+full resync). Изменения не теряются.
- Изменение/удаление события в Google → отражается в `calendar.db` (push: секунды;
  fallback-поллинг: ≤ интервала; удаление → tombstone) и публикует
  `calendar.events_changed`.
- `/roadmap` показывает события как временные якоря (отличные от goals/tasks),
  recurrence развёрнут в окне с cap (`truncated=true` при усечении), обновляется
  live по SSE. Recurring-серия, начатая до baseline, показывает будущий instance.
- **Privacy:** private/freeBusy/public-tier события — без `description`/`location`/
  `html_link`; скрытый summary → «Занято».
- `calendar_context()` отдаёт реальные busy-blocks.
- Write-tools остаются dormant; scope только readonly.
- ADR-0015 оформлен; доки подсистем обновлены (L.1).

## Верификация

- **Unit (pm-mcp-server, mock Google, isolated temp `calendar.db`):** full sync
  (raw, последняя страница даёт `nextSyncToken`); incremental с теми же stable-
  параметрами + `syncToken`; `410` → clear+full resync; cancelled tombstone;
  recurrence expansion + cap/`truncated`; **spike** recurring-до-baseline → future
  instance; несколько календарей + tiers; поллер-тик; re-baseline cleanup.
  `uv run ruff check .`, `uv run pytest`.
- **B (mock):** валидация триплета/states/idempotency **в PM-MCP** internal route
  (валидный → sync, невалидный/старый message-number → отклонён); gateway-edge
  форвардит и НЕ трогает store; renew со `stop` старого канала.
- **C:** unit на `roadmap_view_model` (event-ноды, privacy-shape); `assistant-ui/
  scripts/verify_frontend.py --page roadmap` (desktop+mobile).
- **Live smoke (безопасно, brick #17):** только read-only/diagnostic ИЛИ против
  **отдельного тестового календаря**; isolated temp DB; НЕ писать в живой
  `calendar.db`, НЕ занимать канонические порты (ephemeral `port=0`).

## Предпосылки от пользователя (manual, вне кода)

1. Google Cloud project → включить Calendar API → создать OAuth 2.0 Client
   (Desktop app), настроить consent screen (test user = твой аккаунт),
   выдать client_id/secret коду через env/keyring.
2. Разово выполнить auth CLI (согласие в браузере → refresh-token сохранён).
3. (Phase B) Настроить Tailscale Funnel на публичный путь webhook gateway по HTTPS.

## Ревью

Codex-ревью редакции (2026-06-05): подтвердил направление; точечные правки —
syncToken-контракт, raw-events vs window, webhook/token lifecycle, split T1,
Funnel preflight, isolated verification, roadmap privacy — внесены выше. Заметка
ревью в AI-memory `#1600`. Второй раунд (`#1601`): recurring-master vs baseline +
spike, gateway не читает store (валидация токена в PM-MCP), точные incremental
stable-параметры, split env (`SELECTED`/`PUBLIC`), cap/`truncated` для /roadmap —
внесены.

## Pre-close retrospective (2026-06-05)

- tech-stack: `no-change` — использованы существующие bricks: SQLite/WAL store, FastMCP loopback, namespaced env, FastAPI background task, Gateway ingress, React/Vite island, runtime verification. Новый Google API pattern зафиксирован в ADR-0016, но не выносится в общий brick без повторного кейса.
- Design-system: `no-change` — Assistant-UI изменения используют app-local React island CSS на `var(--md-sys-color-*)`, typography/shape tokens и не требуют правок `D:\GitHub\Design-system`.
- skills: `no-change` — применены `central-plan-workflow`, `pm-mcp-task-flow`, `impeccable`, `frontend-verification`, `ai-memory-recall/capture`; отдельный skill не нужен.
- hooks: `follow-up-task` — существующий #708 остаётся общим follow-up по hook discipline; дополнительных hook-изменений в этом плане нет.
- ADR note: фактический Calendar ADR — `docs/adrs/0016-google-calendar-read-sync.md`, потому что `0015` занял Obsidian read scope.
- Completion: PM-MCP #818-#825 закрыты в `Готово`; AI-memory записи сохранены по затронутым подсистемам; unique acceptance criteria из плана перенесены в код, ADR и subsystem docs.
