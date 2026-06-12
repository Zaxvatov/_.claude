# Бюджет: контур анализа и управления для агентов (Level 1–3)

PM-MCP (создать после утверждения; ADR — не отдельная подсистема, вложен в budget):
- budget (project_path=budget): **#798** — ADR-0010 + threat model/privacy/retention
  (Фаза 0), rules + `get_budget_rules` (Фаза 1), `budget_proposals` + propose-tools +
  connection-aware apply + retention (Фаза 2), budget docs + правка корневого AGENTS.md.
- assistant-ui (project_path=assistant-ui): **#799** — review/apply routes (auth+CSRF) +
  панель ревью + локальный E2E (Фаза 3). Зависит от #798 (budget).
- gateway (project_path=gateway): **#800** — budget.read exposure (Фаза 1) + budget.propose
  (Фаза 4, после E2E). Зависит от #798 (budget, read) и #799 (assistant-ui, propose).

## Контекст

Задача: дать агентам (ChatGPT через gateway + локальные Claude Code/Codex)
возможность **анализировать и управлять** личным бюджетом. ChatGPT-набросок
описывает 3 уровня (анализ → анализ+правила → управление через proposal-модель)
и предполагает, что budget-инструментов ещё нет.

Реальность после изучения репозитория: значительная часть уже построена.

**Что уже есть (НЕ переделывать, не переименовывать — E.2):**
- Подсистема `budget/` (задачи #664–665): SQLite-домен, сервисы, Excel-импорт,
  FastMCP-daemon на `127.0.0.1:8767`, singleton-lock, `/healthz`.
- **10 read-only MCP tools** покрывают весь Level 1 ChatGPT (наброску нужны
  3 — все есть под другими именами):
  - `budget_summary` → [`get_analytics_snapshot`](D:/GitHub/AI-Assistant/budget/budget/mcp_app.py:67)
  - `budget_transactions` → [`list_transactions`](D:/GitHub/AI-Assistant/budget/budget/mcp_app.py:81)
  - `budget_monthly_report` → [`get_monthly_product`](D:/GitHub/AI-Assistant/budget/budget/mcp_app.py:72)
  - плюс `get_balances`, `list_categories`, `list_accounts`, `list_settings`,
    `list_available_months`, `list_operation_types`, `list_need_kinds`.
- **Write-логика транзакций уже есть** внутри домена и используется Assistant-UI
  напрямую (НЕ как MCP-tools): [`create_transaction`/`update_transaction`/`replace_transactions_for_date`/`delete_transaction`](D:/GitHub/AI-Assistant/budget/budget/services/transactions.py:563).
- Ключ-значение настройки `app_settings` + типизированные геттеры — основа для
  Level 2 правил уже есть. Категории несут `kind` (Постоянная/Переменная) и
  `need_kind`; транзакции — отдельное поле `exp_children_chf`.

**Главные пробелы (фактический объём работы):**
1. **Budget не выведен наружу.** В [gateway](D:/GitHub/AI-Assistant/gateway/gateway/scope_policy.py:10)
   только `memory.*` и `pm.*` scope. ChatGPT сейчас **не может** дотянуться до
   бюджета; локальные агенты — могут уже сегодня через loopback.
2. **Privacy не закрыт.** [`gateway.redaction`](D:/GitHub/AI-Assistant/gateway/gateway/redaction.py:6)
   маскирует только токены/секреты — НЕ суммы, комментарии, названия счетов и
   категорий. Сырой вывод бюджета наружу недопустим без data-minimization.
3. **Нет структурированных «бюджетных правил»** (целевые остатки/накопления/долги).
4. **Нет контура записи.** `budget/AGENTS.md` и `ARCHITECTURE.md` прямо фиксируют
   «MCP read-only в v1; write требует отдельной задачи и guardrail-решения».

**Эталон proposal-модели уже существует** — AI-memory write-контур
(`propose_memory` → approval в UI). Tech-stack brick #5: расширение write-семантики
для external клиента → **новый ADR + ревизия scope tree**, не lazy-route.

## Решения (из обсуждения + ревью Codex)

- **Аудитория:** ChatGPT (через gateway, OAuth/PKCE) + локальные агенты.
- **Объём:** весь контур Level 1–3.
- **Модель записи:** единый контур `propose → approve в Assistant-UI → apply`
  для ВСЕХ клиентов. Прямых write-MCP-tools для агентов нет.
- **Approve/reject — только UI.** Это НЕ loopback MCP-tools: budget MCP loopback
  без auth, и approve-tool в `EXPECTED_TOOLS` позволил бы локальному агенту обойти
  UI-подтверждение. В MCP остаются только `budget_propose_*`; `list/approve/reject/
  apply` — direct-import сервис для Assistant-UI routes c auth + CSRF.
- **Data-minimization наружу.** Раздельные scope: `budget.read` (агрегаты/правила/
  балансы/справочники, без сырых строк) и `budget.read.transactions` (минимизированные
  строки, отдельный opt-in). Сырые комментарии по умолчанию не отдаются; field
  allowlist + hard clamp `limit` (запрет `limit=None` наружу).
- **Provenance ставит сервер, не payload.** `source`/`client_fingerprint`/subject/
  scope/ip/request_id инжектит gateway; пользовательские reserved-поля отбрасываются.
- **Порядок безопасный:** ADR → внешнее чтение (без сырых транзакций) → proposals
  локально → UI-подтверждение → и только потом внешний propose-контур.

## Целевая архитектура контура

```
ChatGPT ──OAuth/PKCE──► gateway (8780) ──┬─ budget.read          → read tools (loopback 8767)
                                         ├─ budget.read.transactions → minimized tx (loopback 8767)
                                         └─ budget.propose        → propose tools (loopback 8767)
Claude/Codex ─loopback─► budget MCP (8767): read + propose  (approve/reject/list — НЕ здесь)

Assistant-UI (direct import, auth+CSRF)
   ├─ панель ревью budget proposals (list)
   └─ approve → atomic apply: budget.services.* (create/update/...) ; reject
```

Принципы:
- **Один daemon.** Отдельный hardened proposals-daemon (как AI-memory:8770 с
  собственным OAuth/2-stage review) НЕ строим — то против недоверенного внешнего
  контента; здесь данные личные, внешний контур закрыт gateway (brick #12/#1,
  ADR-0001 D-8). Выделение в отдельный daemon — путь отхода, если понадобится анти-абуз.
- **Propose не мутирует**; apply — только после approve в UI; apply переиспользует
  существующие сервисы и транзакционный путь (lock-date, пересчёт балансов).
- **Наружу — только read + propose.** `list/approve/reject` не существуют как MCP-tools.

---

## Фаза 0 — ADR + threat model + privacy policy (только документ)

`docs/adrs/0010-budget-write-proposal-contour.md`:
- Решение о write-контуре бюджета, propose-only наружу, единый daemon, отмена
  «budget MCP read-only v1».
- **Scope-дерево:** `budget.read`, `budget.read.transactions`, `budget.propose`.
- **Data-minimization policy:** что отдаём в `budget.read` (агрегаты, балансы,
  справочники, правила) vs что прячем; transactions — отдельный scope, field
  allowlist, комментарии off by default, clamp limit.
- **Имена счетов/категорий наружу:** решить — реальные (Raiff, Mono) или стабильные
  псевдонимы/labels; рекомендация — alias-map для external `budget.read` (balances
  раскрывает имена счетов).
- **Proposal retention (ADR-0005-стиль), конкретные сроки (env-конфигурируемо):**
  `pending` → `expires_at` = created+14д, по истечении статус `expired`;
  `rejected`/`expired` → полный payload purge через 30д после смены статуса;
  `applied` → `payload_json` санитизируется через 90д после `applied_at` (остаются
  `payload_sha256`/`status`/result summary), строка удаляется через 365д.
- **`budget.read.transactions` — reserved / phase-later.** Зафиксировать в ADR как
  зарезервированный scope; НЕ вносить в `ALLOWLIST`/`SUPPORTED_SCOPES`/metadata, пока
  не утверждена политика минимизации построчных данных.
- **Trust boundary / provenance:** gateway-verified subject для external;
  loopback = trust-by-default (свои агенты); provenance — серверная.
- **DCR opt-in:** budget-scopes НЕ входят в default DCR scopes (явный запрос).

## Фаза 1 — Level 2 правила + минимальное внешнее чтение (без сырых транзакций)

Правила: производное вычисляем, целевое храним.
- Производные (не дублировать): `monthly_income` из доходов; `fixed_expenses` из
  категорий `kind='Постоянная'`; `children_expenses` из `exp_children_chf`.
- Хранимые цели в `app_settings`, ключи `rule.*`: `rule.minimum_balance`,
  `rule.savings_goal`, `rule.debt_payments`, `rule.social_support_limits`
  (+ опц. override `rule.monthly_income`).

1. `budget/budget/services/settings.py` — `get_budget_rules()`: хранимые `rule.*`
   + производные сигналы (fixed-total, children-total, доход/месяц, текущий остаток
   из `balances_service`).
2. `budget/budget/mcp_app.py` + `runtime_contract.py` — read-tool `get_budget_rules`
   (+ в `EXPECTED_TOOLS`).
3. **Gateway budget.read exposure** (полный список точек — иначе scope-drift):
   - [`scope_policy.py`](D:/GitHub/AI-Assistant/gateway/gateway/scope_policy.py:10) — `budget.read` + маршруты
     `/budget/summary`→`get_analytics_snapshot`, `/budget/monthly_report`→`get_monthly_product`,
     `/budget/balances`→`get_balances`, `/budget/categories`→`list_categories`,
     `/budget/accounts`→`list_accounts`, `/budget/available_months`→`list_available_months`,
     `/budget/rules`→`get_budget_rules`.
   - [`auth.SUPPORTED_SCOPES`](D:/GitHub/AI-Assistant/gateway/gateway/auth.py:11) — добавить budget-scopes.
   - [`app.DEFAULT_DCR_SCOPES`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:39) — переопределить ЯВНО,
     исключив `budget.*` (сейчас `frozenset(ALLOWLIST)` авто-включил бы их).
   - OAuth metadata (оба эндпоинта: [auth-server](D:/GitHub/AI-Assistant/gateway/gateway/app.py:241),
     [protected-resource](D:/GitHub/AI-Assistant/gateway/gateway/app.py:258)) — добавить scopes.
   - [`MCP_TO_ROUTE`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:40) + [`_mcp_tools`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:828) — дескрипторы budget read-tools с securitySchemes.
   - **Отдельный profile endpoint для бюджета.** Существующие `/mcp/read` и `/mcp/write`
     НЕ трогаем — иначе ChatGPT read-коннектор увидит budget-tools в `tools/list` и
     начнёт запрашивать budget scopes. Добавить профиль `budget_read` в
     [`MCP_PROFILES`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:48) и endpoint
     `/mcp/budget/read` в [`_do_POST`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:500)
     (сейчас роутятся только `/mcp`,`/mcp/read`,`/mcp/write`); либо отдельный connector URL.
   - [`upstream_health_lines`](D:/GitHub/AI-Assistant/gateway/gateway/app.py:684) — budget upstream; `config.budget_mcp_url`; клиент в [`create_backend_handlers`](D:/GitHub/AI-Assistant/gateway/gateway/backends.py:40).
   - `gateway/openapi.yaml`, gateway README, gateway tests.
4. **Hard clamp** на любых list-returning внешних tool: max limit, запрет
   `limit=None`; тест на response cap.

NB: сырые транзакции наружу в этой фазе НЕ выводим.

## Фаза 2 — `budget_proposals` + propose-tools (локально, без approve в MCP)

1. **Схема** `budget/budget/db.py` (`SCHEMA_VERSION` 4→5, идемпотентно): таблица
   `budget_proposals` с полями (учтён ревью): `id`, `created_at`, `op_kind`
   (`transaction`/`category_change`/`plan_change`), `payload_json`, `payload_version`,
   `payload_sha256`, `target_ref`, `target_updated_at` (precondition),
   `target_snapshot_json` (precondition для update/delete/category/settings),
   `reason`, `source`, `created_by_subject`, `client_fingerprint`,
   `gateway_request_id`, `idempotency_key` (`UNIQUE(created_by_subject, idempotency_key)`), `status`
   (`pending`/`applied`/`rejected`/`expired`), `expires_at`, `reviewed_by`,
   `reviewed_at`, `applied_at`, `result_json`, `error_json`, `review_reason`.
2. **Сервис** `budget/budget/services/proposals.py` (паттерн
   [`ProposalRepository`](D:/GitHub/AI-Assistant/ai-memory/memory/proposals/proposals.py:57)):
   insert, list, get, `mark_applied`, `mark_rejected`, retention
   (`purge_expired`/`sanitize_applied`). Pydantic-схемы предложений в `schemas.py`
   поверх `TransactionIn`/`CategoryIn`/`AppSettingIn`.
   - **Idempotency:** key принимается от gateway/локального MCP; если отсутствует —
     сервер строит deterministic hash по `normalized(payload)+op_kind+target_ref+subject`.
     Уникальность — `(created_by_subject, idempotency_key)`; повторный propose
     возвращает существующий `proposal_id`, дубль не создаётся.
3. **Propose-tools** (пишут pending, без мутации; provenance серверная, reserved
   user-поля отбрасываются) в `mcp_app.py` + `EXPECTED_TOOLS`:
   - `budget_propose_transaction` — перевод между счетами = транзакция с парными
     `inc_*`/`exp_*`, отдельной transfer-сущности нет (AI-memory #778);
   - `budget_propose_category_change` — только разрешённые mutable-поля категории
     (item/kind/need_kind/planning/sort/active); создание/удаление категорий и любые
     изменения счетов — вне scope;
   - `budget_propose_plan_change` — **только ключи `rule.*`**; служебные настройки
     (lock-date и будущие системные ключи) предлагать нельзя.
4. **Connection-aware apply-слой (предусловие атомарности).** Текущие сервисы сами
   открывают connection и commit (напр. [`create_transaction`](D:/GitHub/AI-Assistant/budget/budget/services/transactions.py:563)),
   поэтому status proposal и мутация бюджета НЕ будут атомарны. Ввести
   `*_in_connection(conn, ...)` варианты (или переиспользовать нижний слой
   `_insert_raw`/`_update_raw`/`_refresh_after_mutation` + repositories), а текущие
   публичные функции сделать тонкими обёртками (open conn → delegate). То же для
   categories/settings.
5. **Atomic apply** (direct-import, используется UI в Фазе 3) в ОДНОЙ connection/
   транзакции: `UPDATE budget_proposals SET status='applied' WHERE id=? AND
   status='pending'` (rowcount≠1 → выход, двойной approve не создаёт две транзакции) →
   проверка preconditions (`target_updated_at`/snapshot, расхождение → `rejected`+
   `error_json`) → мутация через `*_in_connection`, уважая
   [`_ensure_not_locked`](D:/GitHub/AI-Assistant/budget/budget/services/transactions.py:45)
   и пересчёт балансов → commit; запись `result_json`/`applied_at`.
6. **Retention worker** (стиль ADR-0005, фоновая задача budget-daemon, brick #12):
   `expire_pending` (14д), `purge_expired` (`rejected`/`expired`, 30д),
   `sanitize_applied` (90д, обнуляет `payload_json`, оставляя hash/status/summary),
   `delete_applied` (365д). **В тестах фоновая задача отключена/детерминирована**
   (функции вызываются напрямую с фиксированным `now`) — без флейков и скрытых
   сайд-эффектов.

## Фаза 3 — Assistant-UI панель ревью + apply (локальный E2E)

1. Роуты в [`main.py`](D:/GitHub/AI-Assistant/assistant-ui/app/main.py) (direct import,
   **auth + CSRF**): list / approve / reject budget proposals. Approve вызывает
   atomic-apply из Фазы 2.
2. Панель по образцу memory ([`memory.html`](D:/GitHub/AI-Assistant/assistant-ui/app/templates/memory.html),
   [`memory.js`](D:/GitHub/AI-Assistant/assistant-ui/app/static/memory.js)); диалоги —
   `<md-dialog>` через общий helper (brick #15), без кастомных `<div role="dialog">`.
3. **E2E локально:** локальный агент `budget_propose_transaction` → pending в панели
   → approve в UI → транзакция в журнале; `balances.verify_invariant` без нарушений.

## Фаза 4 — Внешний propose-контур (после доказанного локального E2E)

1. Gateway `budget.propose` scope + маршруты `/budget/propose_transaction|
   propose_category_change|propose_plan_change` (все точки enumeration как в Фазе 1) +
   отдельный profile `budget_write` и endpoint `/mcp/budget/write` (НЕ трогая `/mcp/write`).
2. Gateway инжектит provenance (subject/scope/ip/request_id) в forwarded body для
   budget-propose маршрутов; budget-tool читает её как серверную, игнорирует
   client-supplied reserved-поля.
3. (Reserved в ADR до утверждения политики; отдельный явный opt-in)
   `budget.read.transactions`: минимизированный tool/проекция (field allowlist, без
   сырых комментариев, clamp limit) — только если анализу реально нужны построчные
   данные наружу.

## Затрагиваемые подсистемы и документация

- `budget/` — db, schemas, services (settings/rules, proposals), mcp_app,
  runtime_contract; обновить `budget/docs/MCP_TOOLS.md`, `budget/AGENTS.md`,
  `budget/ARCHITECTURE.md` (снять «read-only v1»).
- `gateway/` — config, backends, scope_policy, auth, app (DCR/metadata/MCP_*),
  openapi.yaml, README, tests.
- `assistant-ui/` — main.py, шаблон+статик панели, tests.
- `docs/adrs/0010-*`; правка строки про budget в корневом
  [`AGENTS.md`](D:/GitHub/AI-Assistant/AGENTS.md:275).

## Верификация (end-to-end)

- **budget:** `uv run ruff check .`, `uv run pytest`; `/healthz`; `EXPECTED_TOOLS`
  == фактическая регистрация (нет approve/reject/list); propose→approve(UI)→apply на
  временной `BUDGET_DB_PATH` создаёт транзакцию; двойной approve = одна транзакция;
  stale precondition → reject; **failure-injection: если мутация падает ПОСЛЕ смены
  статуса — вся транзакция откатывается, proposal НЕ остаётся `applied`**; retention-
  функции вызываются детерминированно (fixed `now`); `balances.verify_invariant` чистый.
- **gateway:** `uv run pytest`; `budget.read` доступен со scope и закрыт без;
  `budget.*` НЕ в DCR-default; SUPPORTED_SCOPES/metadata/MCP_TO_ROUTE/_mcp_tools/
  openapi синхронны (drift-тест); clamp limit и response cap покрыты тестами;
  approve/reject недостижимы наружу.
- **assistant-ui:** `uv run ruff check .`, `uv run pytest`; CSRF на approve/reject;
  Playwright-smoke панели (desktop/dark/mobile, `<md-dialog>`).

## После утверждения плана (процесс)

1. PM-MCP задачи по подсистемам (K.2/K.4) — см. шапку; связать зависимостями (K.7):
   gateway.read зависит от budget rules; UI зависит от proposals; gateway.propose
   зависит от локального E2E. Записать id в шапку плана.
2. Реализация по фазам 0→4; атомарные кросс-сабсистемные коммиты (J.4) со ссылкой
   на ADR/задачу.
3. Skills для реализации: `pm-mcp-task-flow` (задачи), `migration-discipline`
   (снятие read-only v1), `frontend-verification`/`impeccable` (панель ревью).
4. Hooks: новый не нужен; gateway scope/tool-descriptor drift покрыть тестами.
   Tech-stack brick «propose→approve→apply поверх read-daemon» фиксировать только
   после реализации и проверки guardrails (предложить с подтверждением).
5. Outcome в AI-memory (project=budget и portfolio).

## Риски

- **Приватность:** финансы уходят в ChatGPT. Mitigation: раздельные scope,
  data-minimization, комментарии off by default, явный DCR opt-in, ADR фиксирует
  набор полей.
- **Обход UI-подтверждения:** approve не существует как MCP-tool; только UI route
  с auth+CSRF.
- **Двойной/устаревший apply:** atomic `UPDATE ... WHERE status='pending'` +
  idempotency_key + preconditions.
- **Неатомарность из-за self-commit сервисов:** mitigation — connection-aware
  apply-слой (`*_in_connection`), единая транзакция status+мутация (Фаза 2).
- **Реальные имена счетов наружу:** ADR решает real vs alias; рекомендация —
  alias-map для external `budget.read`.
- **Финансовые payload в proposals:** retention `purge_expired`/`sanitize_applied`.
- **Scope/manifest drift:** единый источник в `EXPECTED_TOOLS` + gateway, покрыт
  drift-тестами.
