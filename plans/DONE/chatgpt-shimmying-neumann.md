# ChatGPT: чтение операций бюджета + запись новой операции (Потребности per-op)

## Context

ChatGPT работает через **gateway** (OAuth, единственный внешний ingress, 8780).
Сейчас он видит только агрегаты бюджета (`budget_summary`, `budget_monthly_report`,
`balances`, `categories`, `accounts`, `available_months`, `rules`) и **не может**:

1. прочитать отдельные операции журнала («какая операция была 2 июня» — виден факт
   дня `62.90 CHF`, но не сама проводка);
2. записать новую операцию с полем **Потребности**.

Почему так (подтверждено по коду):

- **Read.** Scope `budget.read` намеренно исключает сырые строки транзакций
  (ADR-0011 §External scopes). Scope `budget.read.transactions` **зарезервирован,
  но не проброшен**: он есть в `gateway/auth.py` SUPPORTED_SCOPES, в `openapi.yaml`,
  в OAuth-metadata, но **отсутствует** в `scope_policy.ALLOWLIST`, `backends.ROUTES`,
  `MCP_TO_ROUTE`, профилях и `_mcp_tools()`. Бэкендный MCP-tool `list_transactions`
  существует в budget-демоне и просто никуда не замаплен наружу.
- **Write.** `budget_propose_transaction` уже существует
  (`budget/budget/mcp_app.py:145`) и уже проброшен через scope `budget.propose`
  (план #800, профиль `/mcp/budget/write`). Дефолтинг **движения** и **типа** из
  статьи уже работает (`_payload_with_category_defaults`,
  `budget/budget/services/transactions.py:307`). Но **Потребности (need_kind)** —
  атрибут только статьи: его нет ни в наборе полей `TransactionIn`, ни в схеме
  транзакции; в журнал он попадает лишь как join `category_need_kind`.

**Цель:** дать ChatGPT (владельцу данных) полноценный read бюджета (операции,
остатки, планы, счётчики, анализ затрат) и привести запись операции к набору полей
формы «Новая операция», сделав **Потребности переопределяемым на операции** полем
(дефолт из статьи).

### Решения пользователя (2026-06-04)

- **Потребности → per-op override.** Дефолт берётся из статьи (Продукты→Базовые),
  но на конкретной операции можно изменить (Продукты/Блага). В **форме «Новая
  операция»**: при выборе/смене статьи Потребность всегда сбрасывается на типовую
  из справочника статьи (перетирая ручной выбор); затем пользователь может
  переопределить. То же — для агента (propose).
- **Объём read = полный.** ChatGPT должен отвечать на любые вопросы по бюджету.
  Проекция операций — полная (`list_transactions` как есть: суммы, итоги, FX,
  остатки). Это **чувствительный row-level ledger** (комментарии, счета, FX,
  балансы), поэтому он остаётся за отдельным explicit-opt-in scope
  `budget.read.transactions`, отдельным профилем, с аудитом и серверным clamp.
- **Коннекторы = два (least-privilege).** Read и propose не смешиваются в одном
  ChatGPT-коннекторе:
  - **Budget Read** → endpoint `/mcp/budget/transactions`,
    scopes `budget.read budget.read.transactions` (агрегаты + строки операций);
  - **Budget Propose** → endpoint `/mcp/budget/write`, scope `budget.propose`.

Связанный контур: план `rosy-cuddling-lemon.md`, ADR-0011, задачи #798/#799/#800.
Ревью Codex: AI-memory `#1577`.

---

## Current state (verified, file:line)

- `gateway/gateway/scope_policy.py:16-34` — `ALLOWLIST`: `budget.read` (7 routes),
  `budget.propose` (3 routes). `budget.read.transactions` и `/budget/transactions`
  отсутствуют. `ensure_allowed` (59-74) режет вызов по scope.
- `gateway/gateway/backends.py:37-46` — `ROUTES` маппит gateway-route → budget-tool;
  `_tool_arguments` (78-79) прокидывает аргументы как есть. `list_transactions`
  принимает `(limit, offset, filters)`, поэтому top-level `month/op_date/...` без
  трансформера до него не дойдут.
- `gateway/gateway/app.py`: `DEFAULT_DCR_SCOPES` (39-47, без budget.* — explicit
  opt-in), `MCP_TO_ROUTE` (48-65), `MCP_PROFILES` (66-89: `budget_read`=7,
  `budget_write`=3). `tools/list` фильтрует **только по профилю** (336-338, 1057),
  вызовы — по scope (394). Routing endpoint→profile (577-585). Готовый hook-поинт
  для нормализации body — `_body_with_gateway_provenance` (417-432), уже
  спец-кейсит `scope == "budget.propose"`.
- `budget/budget/mcp_app.py:83-107` — `list_transactions(limit, offset, filters)`;
  `filters` = `op_type|category_id|month|op_date|comment_query|sort_desc`.
- `budget/budget/services/transactions.py:56-209` — `TransactionRow` + `SELECT_SQL`
  (join `category_need_kind`, суммы, FX, остатки) + сервис `list_transactions`.
  `307-321` — `_payload_with_category_defaults` (private): фолбэк `op_type/movement`
  из статьи; ранний guard выходит, если заданы оба.
- `budget/budget/schemas.py` — `NeedKind` literal (:14), `TransactionIn` (:117-145,
  **без** need_kind), `CategoryIn.need_kind` (:49), `default_movement/op_type` (:50-51).
- `budget/budget/services/proposals.py` — `propose_transaction` хранит
  `TransactionIn.model_dump()`; apply зовёт `create_transaction_in_connection`
  (наследует дефолтинг); `CATEGORY_MUTABLE_FIELDS` (need_kind/default_*).
- **assistant-ui (форма «Новая операция»)** — логика в
  `assistant-ui/app/static/budget.js` (ADR-0008: shared Design-system asset
  `assistant-budget.js` удалён; Glob подтверждает отсутствие). Текущее поведение:
  - `:391` select Потребности с `name="category_need_kind"`, `state.category_need_kind`;
  - `:460-467` смена статьи уже проставляет need/op_type/movement из статьи (нужное поведение);
  - `:468-471` смена Потребности **очищает статью** (старое поведение — удалить);
  - `:530-554` `buildTransactionData` **не отправляет** need_kind;
  - `app/main.py:~1376-1405` `POST/PUT /api/budget/transactions` зовут budget-сервис.
- ADR-0011 (`docs/adrs/0011-budget-write-proposal-contour.md:46-47,56-57`):
  `budget.read.transactions` reserved до политики минимизации; «для list-like
  внешних tools gateway **обязан clamp'ить** limit и применять response cap».

---

## Deliverables

### A. budget — `need_kind` как поле операции + публичный резолвер дефолтов  (PM-MCP: #801)

1. **Схема/миграция** (`budget/budget/db.py`, идемпотентный `_apply_migrations` —
   guard'ы через `_table_columns`, не numeric user_version):
   - `TRANSACTIONS_TABLE_SQL` (CREATE TABLE transactions, :151): добавить колонку
     `need_kind TEXT CHECK(need_kind IN ('Базовые','Блага','Развлечения (эмоции)') OR need_kind IS NULL)`
     — fresh-схема и rebuild-цель получают колонку;
   - обновить `_rebuild_transactions_without_movement_lookup` (:193-217): добавить
     `need_kind` в список колонок INSERT...SELECT, иначе legacy-rebuild теряет колонку;
   - **idempotent migration** для существующих БД: при отсутствии колонки
     (`"need_kind" not in _table_columns(conn, "transactions")`) — `ALTER TABLE transactions
     ADD COLUMN need_kind TEXT` (CHECK через ADD COLUMN SQLite не вешает — инвариант держит
     app-level `NeedKind` + sanitation статей);
   - **backfill — строго после** существующей sanitation `categories.need_kind` (:247-254,
     она уже NULL'ит невалидные значения, напр. Фриланс `0`) и после rebuild, в той же
     транзакции; перед прогоном — backup runtime `data/budget.db`:
     `UPDATE transactions SET need_kind = (SELECT need_kind FROM categories c WHERE c.id = transactions.category_id) WHERE category_id IS NOT NULL AND need_kind IS NULL`;
   - bump `SCHEMA_VERSION` 5 → 6 (:11);
   - **post-check**: count строк `category_id IS NOT NULL AND need_kind IS NULL` (ожидается
     только там, где у статьи need_kind пуст), 0 невалидных need_kind в transactions, +
     `balances.verify_invariant`.
2. **Схема ввода** (`schemas.py`): `need_kind: NeedKind | None = None` в `TransactionIn`.
   Итоговый набор полей «Новой операции»:
   `op_date, accounts(inc/exp), category_id, op_type, movement, need_kind, comment`.
3. **Публичный резолвер** (`transactions.py`): вынести
   `resolve_transaction_defaults_in_connection(conn, payload) -> (resolved_payload, resolved_meta)`,
   который фолбэчит `op_type/movement/need_kind` из статьи. **Переработать guard**:
   подтягивать статью, если не задано любое из `op_type/movement/need_kind` при наличии
   `category_id`. `_payload_with_category_defaults` стать тонкой обёрткой над публичным
   резолвером (или заменить вызовы). `resolved_meta` = `{op_type, movement, need_kind,
   category_item}` для echo.
4. **Чтение** (`TransactionRow`/`SELECT_SQL`/`list_transactions`): добавить
   `t.need_kind AS need_kind`; в проекции возвращать `need_kind` операции (fallback на
   `category_need_kind`, если NULL). `category_need_kind` оставить как «текущий дефолт
   статьи».
5. **Propose** (`proposals.py` `propose_transaction`, create/update): вызвать публичный
   резолвер (через connection-aware слой) **до** сохранения payload — предложение несёт
   разрешённые движение/тип/Потребности (паритет с формой). В ответ tool добавить
   `resolved` (из `resolved_meta`). Apply-time дефолтинг оставить идемпотентной страховкой.
   **Не вызывать** private `_payload_with_category_defaults` из proposals.
6. **Docs/tests**: `budget/docs/MCP_TOOLS.md` (need_kind в payload и в list_transactions);
   тесты: backfill миграции (+ idempotency повторного запуска), per-op override vs дефолт
   из статьи, propose-time резолв+echo, list_transactions возвращает need_kind.

### B. gateway — активировать `budget.read.transactions` + clamp + профиль  (PM-MCP: #803, dep A, D)

1. `scope_policy.py:16-34` — добавить
   `"budget.read.transactions": frozenset({"/budget/transactions"})`.
2. `backends.py:37-46` — `"/budget/transactions": BackendRoute("list_transactions", "budget_mcp")`.
3. **Серверный clamp + transformer** (ADR-0011 §Privacy): по аналогии с
   `_body_with_gateway_provenance` добавить нормализацию body для
   `route == "/budget/transactions"` (вызывать в `handle()` после `ensure_allowed`,
   до backend):
   - whitelist filter-ключей: `month, op_date, category_id, op_type, comment_query`
     (неизвестные — отбросить); собрать их в `filters`;
   - валидация: `month` `^\d{4}-\d{2}$`; `op_date` ISO `^\d{4}-\d{2}-\d{2}$`;
     `category_id` int ≥ 1; `op_type` str ≤ 64; `comment_query` str ≤ 100;
   - `limit`: запретить `None`, default 100, max 500; `offset`: int ≥ 0, max 5000
     (числовые пороги клампятся **молча** — нормализация, не ошибка);
   - **error contract**: малформированное значение фильтра (`month/op_date/category_id/
     op_type/comment_query` не прошло валидацию) → `GatewayResponse(400, {"ok": False,
     "error": "invalid_budget_filter:<field>"})` + аудит `_audit(..., "invalid_request",
     scope=decision.scope, error=...)`; слой `tools/call` (app.py:363-369) отдаёт это
     клиенту как JSON-RPC error;
   - на выходе body = `{limit, offset, filters, tool_name}` (env-tunable пороги,
     по образцу `BUDGET_PROPOSAL_*` — опционально).
4. `app.py` — `MCP_TO_ROUTE += {"budget_transactions": "/budget/transactions"}`;
   новый профиль **`budget_read_transactions` = budget_read ∪ {budget_transactions}**;
   endpoint `/mcp/budget/transactions` (добавить в набор routes :577 и mapping :583-585);
   дескриптор в `_mcp_tools()` c `required_scope="budget.read.transactions"`,
   `read_only=True`, input-schema (`month|op_date|category_id|op_type|comment_query|limit|offset`).
   `/mcp/budget/read` и `/mcp/budget/write` **не менять**. `DEFAULT_DCR_SCOPES` не трогать.
5. Доки: `openapi.yaml`/`README.md`/`gateway/ARCHITECTURE.md` — снять «reserved; not
   routed», описать новый endpoint/scope и разницу `budget.read` (агрегаты) vs
   `budget.read.transactions` (row-level ledger, sensitive).
6. Tests (`gateway/tests/`): `budget.read.transactions` открывает `/budget/transactions`,
   `budget.read` в одиночку — нет; clamp (limit=None/`>max`, offset>max, мусорные filter-ключи
   отброшены, невалидный month отвергнут); профиль `/mcp/budget/transactions` показывает
   агрегаты + транзакции; обновить tool-count/profile-ассерты.

### C. assistant-ui — Потребности в форме «Новая операция» (migration old→new)  (PM-MCP: #804, dep A)

Только `assistant-ui` (shared primitives не трогаем).

1. `assistant-ui/app/static/budget.js`:
   - `:391` переименовать `name="category_need_kind"` → `name="need_kind"`;
     preset-state читать `state.need_kind` (после A.4 list_transactions отдаёт need_kind);
   - **удалить** старый хендлер `:468-471` (смена Потребности больше **не очищает** статью —
     Продукты/Блага остаётся Продукты);
   - **сохранить** `:460-467` (смена статьи перезаписывает need/op_type/movement дефолтами статьи);
   - `buildTransactionData` `:530-554` — добавить `data.need_kind = rowValues.need_kind || null;`.
2. `assistant-ui/app/templates/budget.html` — если budget-context не содержит список
   need_kinds, добавить (для опций select); метку оставить «Потребности».
3. `assistant-ui/app/main.py` (`POST/PUT /api/budget/transactions`) — пробросить
   `need_kind` в budget-сервис (сервис сам дефолтит из статьи по A.3).
4. Frontend verification: `scripts/verify_frontend.py --page budget` (desktop+mobile,
   изолированно, brick #17), `node --check app/static/budget.js`.

### D. ADR-0012 + кросс-доковая правка (gate для B)  (PM-MCP: #802)

- **ADR-0012** (следующий свободный номер в `docs/adrs/`): «Budget external
  transaction reads + per-operation need_kind». Зафиксировать:
  - `budget.read.transactions` раскрывает **row-level ledger** (комментарии, счета,
    FX, балансы) — чувствительный слой; контроль: explicit opt-in scope, отдельный
    профиль `/mcp/budget/transactions`, audit hash-chain, серверный clamp+response cap,
    понятное имя коннектора **Budget Read**, разделение read/propose по коннекторам;
  - доменное изменение: `need_kind` на операции (per-op override, дефолт из статьи,
    backfill из `categories.need_kind`).
- Обновить `budget/ARCHITECTURE.md` (need_kind, набор полей операции), `budget/AGENTS.md`
  при необходимости.

---

## Verification

- **budget**: `uv run ruff check .`; `uv run pytest` (A.6); миграция: backup →
  migrate → backfill → `balances.verify_invariant(sample_every=25)` → post-check count;
  smoke `GET /healthz`.
- **gateway**: `uv run ruff check .`; `uv run python -m unittest discover -s tests` (B.6).
- **assistant-ui**: `uv run ruff check .`; `uv run pytest`;
  `uv run python scripts/verify_frontend.py --page budget`.
- **E2E (изолированно, brick #17 — без записи в живую БД)** — развести direct vs gateway:
  - **direct budget MCP**: `list_transactions(filters={op_date:"2026-06-02"})` → проводка
    дня со статьёй, типом, движением, **need_kind**, суммами, комментарием (проверяет
    доменный слой/проекцию);
  - **external Gateway MCP**: `budget_transactions(op_date="2026-06-02")` через
    `/mcp/budget/transactions`, токен `budget.read budget.read.transactions` → тот же
    результат (проверяет transformer top-level→`filters`, профиль и scope-gating); тот же
    вызов с токеном только `budget.read` → 403/denied;
  - **propose**: `budget_propose_transaction(action="create", transaction={op_date,
    category_id=Продукты, exp_*})` → ответ `resolved`: movement=`-`, op_type,
    need_kind=Базовые; override `need_kind="Блага"` сохраняется; apply через Assistant-UI
    `/budget/proposals`;
  - **clamp/error**: `limit=null`/огромный limit/offset — молча нормализуются; мусорный
    filter-ключ отброшен; невалидный `month` → JSON-RPC error.
- **Операционный шаг (разблокирует ChatGPT)**:
  1. **Revoke / re-auth** текущий ChatGPT OAuth client (он на `budget.read` →
     `/mcp/budget/read`); токены НЕ копировать в чат/transcript.
  2. Зарегистрировать **два** коннектора (budget.* — explicit opt-in DCR):
     - **Budget Read** → `/mcp/budget/transactions`, scope `budget.read budget.read.transactions`;
     - **Budget Propose** → `/mcp/budget/write`, scope `budget.propose`.
  3. Повторить: «какая операция была 2 июня»; «проведи анализ затрат, где экономить»;
     запись новой операции (Продукты/Блага) → проверить в `/budget/proposals`.

---

## PM-MCP tasks (созданы 2026-06-04; выполняет Codex)

- `#801` budget: need_kind per-op + публичный резолвер + propose echo.
- `#802` docs/ADR-0012 (**gate** перед exposure).
- `#803` gateway: активировать budget.read.transactions + clamp + профиль (dep A, D).
- `#804` assistant-ui: Потребности в форме, миграция old→new (dep A).

Порядок: A → D → B; C параллельно после A. (Skill `migration-discipline` для C и для
backfill в A.)

## Post-approval follow-ups (propose-only, с подтверждением)

- Кандидат в `tech-stack-choices.md` (#5 scope tree): «row-level external read
  требует owner approval, explicit scope, **profile isolation**, серверный clamp +
  response cap, audit и ADR — недостаточно того, что агрегаты уже экспонированы».
- Новых hook-напоминаний не требуется; backfill/удаление старого UI-поведения
  покрываются skill `migration-discipline`, новый scope — gateway-тестами.
