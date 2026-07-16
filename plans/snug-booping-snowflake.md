# Разделение даты операции бюджета на План и Факт

PM-MCP: #1128

## Контекст

Сейчас у операции бюджета одна дата — `transactions.op_date TEXT NOT NULL`. Из неё
же выводится всё: признак плана (`is_planned = op_date > today`,
[transactions.py:178](../../../D:/GitHub/AI-Assistant/budget/budget/services/transactions.py)),
курс FX (план → курс сегодня, факт → курс своей даты), порядок и материализованные
балансы (`ORDER BY op_date, sort_index, id`), блокировка `lock_date`, фильтры
`planned`, помесячная аналитика, повторяющиеся операции и внешний MCP-контур.

Нужно различать **план** (когда операция ожидается) и **факт** (когда реально
произошла). Модель: два поля `plan_date` (NOT NULL) и `fact_date` (nullable);
`план ⇔ fact_date IS NULL`. Имя `op_date` уходит из проекта целиком, включая
внешнюю границу MCP (по решению пользователя — ради ясности, ценой переавторизации
Budget-коннекторов).

Все дизайн-решения зафиксированы с пользователем (см. раздел «Согласованные
правила»). Отдельный отчёт «план vs факт» и запись планов через MCP — вне scope.

## Согласованные правила (инварианты)

- **Признак:** `is_planned ⇔ fact_date IS NULL`; «просрочено» ⇔ `fact_date IS NULL AND plan_date < today`.
- **Рабочая дата** (сортировка, группировка журнала, порядок балансов, блокировка,
  колонка даты) = `COALESCE(fact_date, MAX(plan_date, today))`.
  Просроченный план встаёт на **сегодня**, исходный `plan_date` не затирается.
- **Дата курса FX** = `COALESCE(fact_date, today)` (любой план — по курсу сегодня;
  факт — по курсу своей даты). Совпадает с текущим поведением ⇒ исторические
  CHF-итоги не меняются.
- **Реализованный баланс** (`final_balances`) считает только факты
  (`fact_date IS NOT NULL AND fact_date <= today`); планы в реализованный баланс не
  входят, но участвуют в проектной цепочке `account_balance_after`.
- **Зеркало факта:** если задан только факт (кейс ChatGPT и импорт истории) →
  `plan_date := fact_date`. `plan_date` всегда NOT NULL.
- **Валидация:** `fact_date <= today` (факт в будущем запрещён); `fact_date` может
  быть раньше `plan_date`; `plan_date` — любая; хотя бы одна из дат задана.
- **Блокировка:** сравнивается по рабочей дате ⇒ просроченный план (рабочая = сегодня)
  никогда не попадает в закрытый период.
- **Журнал:** одна колонка рабочей даты + бейдж «план»/«просрочено»; просроченные
  планы визуально собираются в группу «сегодня».

## Миграция (SCHEMA_VERSION 10 → 11)

Полный rebuild таблицы `transactions` (паттерн уже есть —
`_rebuild_transactions_without_movement_lookup` в
[db.py](../../../D:/GitHub/AI-Assistant/budget/budget/db.py)). Новая схема столбцов
даты: `plan_date TEXT NOT NULL`, `fact_date TEXT` (вместо `op_date TEXT NOT NULL`).
Перенос данных из старого `op_date` (где `migration_today = date.today()`):

- `plan_date := op_date` (всегда);
- `fact_date := CASE WHEN op_date <= migration_today THEN op_date ELSE NULL END`.

Итог: прошлые/сегодня → `plan = fact = старая дата`; будущие → `plan = старая, fact = NULL`.
FX-нейтрально. Сохранить все прочие столбцы (`need_kind`, `recurring_*`, суммы,
derived). Обновить индексы: `idx_tx_date_sort` → `(plan_date, sort_index)`, добавить
`idx_tx_fact_date (fact_date)`.

## Централизация (чтобы не размазывать COALESCE по 20+ местам)

Новый модуль `budget/budget/services/op_dates.py`:

- SQL-фрагменты: `work_date_sql(alias)` → `COALESCE({a}.fact_date, MAX({a}.plan_date, ?))`,
  `fx_date_sql(alias)` → `COALESCE({a}.fact_date, ?)` (параметр `?` = `today` в ISO);
- Python: `working_date(plan, fact, today)`, `fx_date(fact, today)`, `is_planned(fact)`.

Все SQL-хотспоты используют эти фрагменты с биндом `today`.

## Backend — домен

**[budget/budget/db.py](../../../D:/GitHub/AI-Assistant/budget/budget/db.py)** — новая DDL
(`SCHEMA_STATEMENTS` + `TRANSACTIONS_TABLE_SQL`), bump версии, миграция-rebuild,
индексы.

**[budget/budget/schemas.py](../../../D:/GitHub/AI-Assistant/budget/budget/schemas.py)** —
`TransactionIn`: заменить `op_date: _date` на `plan_date: _date | None` +
`fact_date: _date | None`; `model_post_init`/валидатор — зеркало факта, `fact_date <= today`,
требование хотя бы одной даты.

**[budget/budget/services/transactions.py](../../../D:/GitHub/AI-Assistant/budget/budget/services/transactions.py)** —
основной объём:
- `TransactionRow`: `plan_date`, `fact_date`, свойства `work_date`, `is_planned`;
- `SELECT_SQL`, `_row_to_tx`, `_insert_raw`, `_update_raw` — новые столбцы;
- `list_transactions`/`count_transactions`: фильтр `planned` → `fact_date IS [NOT] NULL`;
  фильтр по дате и `month` → по рабочей дате; `ORDER BY` рабочая дата;
- `_next_sort_index`, `_ensure_empty_row_for_date` — ключ по рабочей дате;
- `_ensure_not_locked` — по рабочей дате;
- `_rebuild_derived` + `_fx_requirements_for_rebuild` — FX-дата и порядок по рабочей дате;
- `replace_transactions_for_date` → работать по рабочей дате; каждая строка несёт
  `plan_date` + опц. `fact_date`;
- `create/update_transaction_in_connection` — новые поля.

**[budget/budget/services/balances.py](../../../D:/GitHub/AI-Assistant/budget/budget/services/balances.py)** —
`ORDER BY` в `_recompute_for_account` и `verify_invariant` → рабочая дата (+ бинд `today`);
`final_balances` — cutoff `fact_date IS NOT NULL AND fact_date <= ?`, сортировка «последней»
строки по рабочей дате.

**analytics.py / products.py** — помесячные группировки и cutoff по рабочей дате;
режим `planned/fact` — через `fact_date IS NULL`.

**recurring_transactions.py / fx_rates.py / importer.py / mcp_app.py** — заменить
конструкции `TransactionIn(op_date=...)` и SQL: сгенерированные повторы = планы
(`plan_date=occurrence`, `fact_date=NULL`); импорт истории = факты (зеркало
`plan_date=fact_date=imported`). Проверить `settings.py` (там, вероятно, только строковые
ключи настроек, не столбец — правок может не быть).

## Frontend — assistant-ui

**[app/main.py](../../../D:/GitHub/AI-Assistant/assistant-ui/app/main.py)** — проекция строки
журнала (`tx.__dict__` + дата, ~стр. 222) отдаёт `plan_date`, `fact_date`, `work_date`;
preview предложений (~246–278) читает `fact_date`; `_budget_journal_row_date` и
«insert empty row» эндпоинты (~414–420) — по рабочей дате; payload ошибки блокировки
(~2071); read-инструмент MCP (~2233) — фильтр по рабочей дате.

**[app/templates/_budget_journal_rows.html](../../../D:/GitHub/AI-Assistant/assistant-ui/app/templates/_budget_journal_rows.html)** —
колонка даты показывает `work_date`; бейдж `is_planned`→«план», просрочено→«просрочено»;
`data-op-date` → рабочая дата; добавить ховер-кнопку «✓ провести» и точку привязки
контекстного меню (строка уже имеет `data-tx`).

**[app/static/budget.js](../../../D:/GitHub/AI-Assistant/assistant-ui/app/static/budget.js)** —
диалог операций: в режиме `edit` две даты на строку «План»(`plan_date`) + «Факт»(`fact_date`),
в режиме `create` только «План»; заголовки, `createOperationRow`, submit-payload,
`dataset.originalDate`→рабочая дата, recurrence `start_date = plan_date`.
Журнал: ховер-кнопка «провести» (POST выставляет `fact_date := plan_date`) и
контекстное меню по правому клику (Провести по плану / Провести сегодня / Изменить /
Удалить). Быстрый путь «провести» — новый узкий эндпоинт или переиспользование
update.

## MCP / Gateway (убрать op_date из контракта)

**[gateway/openapi.yaml](../../../D:/GitHub/AI-Assistant/gateway/openapi.yaml)** —
`BudgetTransactionIn.op_date` (стр. 31, required 70) → `fact_date` (required);
read-фильтр `op_date` (стр. 572) → `fact_date` (+ `month` по рабочей дате); описание
инструмента (стр. 1680).

**[gateway/gateway/app.py](../../../D:/GitHub/AI-Assistant/gateway/gateway/app.py)** —
`BUDGET_TRANSACTION_FILTERS` (стр. 177), `BUDGET_TRANSACTION_IN_SCHEMA` (стр. 210, 268),
ключи `_validated_budget_date_filter` (стр. 761), описание (стр. 1680): `op_date`→`fact_date`.

**budget MCP (propose/read)** — apply: `fact_date` → факт (+ зеркало `plan_date`); read-проекция
возвращает `plan_date` + `fact_date`, без `op_date`.

Правка **[docs/adrs/0012](../../../D:/GitHub/AI-Assistant/docs/adrs/0012-budget-transaction-read-and-need-kind.md)**:
новый раздел про `plan_date`/`fact_date`, рабочую дату, «propose фиксирует только факт»
и переименование внешнего поля `op_date → fact_date`.

> Внешний контракт меняется ⇒ Budget Propose и Budget Read коннекторы ChatGPT нужно
> переавторизовать/переконфигурировать после деплоя.

## Риски

- **Шифрованная БД** (sqlcipher): rebuild выполнять внутри `connect_database`, сохранить
  FK/индексы (следовать существующему rebuild-паттерну).
- **Регрессия балансов:** на текущих данных (без просроченных планов) новый порядок
  обязан давать те же балансы — снять снапшот `final_balances` до/после миграции.
- **Обрыв внешних коннекторов** до переавторизации — согласовать с деплоем.

## Критерии приёмки / Верификация

1. **Миграция:** прогнать `init_db` на копии старой БД; прошлые строки → `fact=plan`,
   будущие → `fact IS NULL`; `verify_invariant()` без расхождений; снапшот
   `final_balances` совпадает до/после.
2. **budget:** `uv run pytest` (в т.ч. `test_recurring_transactions`, `test_balances`,
   `test_mcp_policy`, `test_server_smoke`) зелёные; тесты обновлены под plan/fact.
3. **assistant-ui:** `uv run pytest` (`test_api_endpoints`) + `verify_frontend.py`;
   запустить приложение, открыть бюджет — журнал показывает рабочую дату и бейджи;
   форма «Изменить» = План+Факт, «Новая» = только План; ховер «провести» переводит
   план в факт; блокировка/лок ведёт себя по рабочей дате.
4. **gateway:** `uv run pytest` (`test_openapi`, `test_gateway_contract`, `test_integration`)
   зелёные; в схемах нет `op_date`.
5. Прогнать `verify` (verify.ps1 / pytest) перед коммитом.

## Порядок работ

1. Схема + миграция + `op_dates.py` (+ снапшот балансов для регресс-проверки).
2. Домен: schemas, transactions, balances, analytics, products, recurring, fx, importer.
3. Внутренний UI: main.py, шаблон журнала, budget.js (формы + перевод план→факт).
4. Контракт MCP/gateway + правка ADR-0012.
5. Тесты всех трёх пакетов + `verify`.

## Статус: реализовано (2026-07-06)

Все этапы выполнены. `op_date` полностью удалён из схемы, домена, UI и внешнего
контракта (остались только: migration-функция `_split_transactions_date`,
helper-модуль `op_dates`, legacy-фикстуры тестов и внутренние локальные имена).

- **Схема/миграция:** SCHEMA_VERSION 10→11, `_split_transactions_date`
  (`ALTER RENAME COLUMN op_date→plan_date` + `ADD fact_date` + backfill), индексы
  перенесены в `_apply_migrations` (иначе падали на legacy-БД).
- **Централизация:** `budget/budget/services/op_dates.py` (SQL-фрагменты
  `work_date_sql`/`fx_date_sql` + Python `working_date`/`fx_date`/`is_planned`/`is_overdue`).
- **Домен:** schemas (валидация + зеркало факта), transactions (в т.ч. новая
  `set_transaction_fact` для «провести»), balances (fact-only реализованный баланс,
  проектная цепочка по рабочей дате), analytics/products (факт-аналитика),
  recurring/importer/fx/settings.
- **UI:** журнал — одна колонка рабочей даты + бейджи «план»/«просрочено», ховер
  «✓ провести» и контекстное меню (Провести по плану / сегодня / Изменить / Удалить);
  форма «Изменить» = План+Факт, «Новая» = только План; узкий эндпоинт
  `POST /api/budget/transactions/{id}/mark-fact`.
- **MCP/gateway:** `op_date` убран из `BudgetTransactionIn` и read-фильтра
  (→ `fact_date`), read проецирует `plan_date`+`fact_date`; ADR-0012 дополнен.
- **Проверки:** budget 59 ✓, assistant-ui 202 ✓, gateway 50 ✓; `node --check`
  budget.js ✓; live-рендер `/budget` показывает бейджи/кнопки, `op_date` в HTML
  отсутствует. Отдельные новые тесты: миграция split и `set_transaction_fact`.

**После деплоя:** переавторизовать/переконфигурировать ChatGPT-коннекторы
Budget Propose и Budget Read (внешнее поле переименовано `op_date → fact_date`).

**Ретроспектива:** tech-stack `no-change`; Design-system `no-change` (контекстное
меню/бейджи бюджета можно вынести follow-up при переиспользовании); Skills
`no-change`; Hooks `no-change`.

### Доработки по фидбеку (2026-07-06)

- Пустые строки-плейсхолдеры больше не считаются планом: `TransactionRow.is_empty`,
  `is_planned`/`is_overdue` возвращают False для пустых строк (нет бейджа/кнопки).
- Убрана плашка «ПЛАН» (и её CSS) — остались только ховер-кнопка «✓ провести» и
  бейдж «просрочено».
- Форма «Изменить»: «План» и «Факт» — отдельные колонки грида (класс
  `budget-operation-table--with-fact`, area `fact`); поля — прямые дети
  `.budget-form-row` → Excel-стиль без обводки. «Новая» — только «План» в прежнем
  стиле. Убран стек `budget-form-datecell` и цветная обводка `budget-field-fact-date`.
- Прошлые операции: поля План/Факт заполняются (API отдаёт `plan_date`/`fact_date`,
  проверено; причина пустоты была в стековой вёрстке, а не в данных).
- Проверки: budget 59 ✓, assistant-ui api 109 ✓; `verify_frontend.py` — budget
  desktop/mobile без console-ошибок (операционная форма с полем `plan_date`).
  Падение `roadmap/desktop` (select placeholder) — не связано с этой задачей
  (правки только в budget-scope, `app.css` валиден).

### Доработки по фидбеку — раунд 2 (2026-07-06)

- **Чекбокс подтверждения** для операций сегодняшней группы (рабочая дата =
  сегодня, непустые): checked ⇔ факт сегодня, uncheck снимает факт. Ховер-кнопка
  «✓» осталась только для будущих планов; контекстное меню — без изменений.
  Backend: `set_transaction_fact` (провести) + `clear_transaction_fact` (снять),
  эндпоинт `mark-fact` принимает `confirmed`.
- **Пустые строки**: у пустой строки нет факта (только план). Правки:
  `_ensure_empty_row_for_date` создаёт плейсхолдер без факта; импортер не ставит
  факт пустым строкам; миграция `_clear_fact_on_empty_rows` (SCHEMA_VERSION 11→12)
  снимает факт с уже существующих пустых строк. Пустая строка **не смещается**
  на сегодня — `op_dates.work_date_sql`/`working_date` держат её на плановой дате
  (`empty_row_sql`), поэтому пустые прошлые строки остаются в прошлом.
- **Fix якоря журнала**: `_budget_journal_initial_offset` теперь использует
  канонический `op_dates.work_date_sql` (а не инлайн-формулу), иначе начальный
  offset расходился с порядком строк на пустых строках → React-остров журнала
  не виртуализировался. После фикса `verify_frontend`: budget desktop/mobile OK,
  без console-ошибок.
- Атрибут data у кнопки/чекбокса переименован `data-tx`→`data-op-id`, чтобы
  `data-tx` оставался только на строках (island читает ключ по нему).
- Проверки: budget 62 ✓, assistant-ui 202 ✓; `verify_frontend` budget desktop/mobile OK.

### Доработки по фидбеку — раунд 3 (2026-07-06)

- Чекбокс «проведено сегодня» в журнале теперь **только по ховеру** строки
  (`opacity`/`pointer-events`), не висит постоянно.
- Тот же быстрый чекбокс добавлен в форму «Изменить операцию» — поверх ячейки
  «Факт», по ховеру (`setupFactQuickCheck`, grid-area `fact`): ставит Факт =
  сегодня / снимает (client-side, применяется при сохранении).
- Пустая ячейка «Факт» в форме больше не показывает нативный плейсхолдер
  `дд.мм.гггг` — при пустом значении текст поля прозрачный
  (`.budget-field-fact.budget-field-empty`, синхронизируется JS). Подтверждено
  скриншотом edit-диалога: столбцы План/Факт, пустой Факт — чистая ячейка.
- Проверки: assistant-ui 202 ✓; `verify_frontend` budget desktop/mobile — без
  console-ошибок; edit-диалог отрисован корректно.

### Доработки по фидбеку — раунд 4 (2026-07-06)

- Поменял местами вид иконки повтора (глиф `schedule` в обоих местах не меняется,
  только FILL/цвет): индикатор повтора в журнале (`.budget-recurring-indicator`) —
  теперь приглушённый контур (`FILL 0`, `on-surface-variant`); активная кнопка
  повтора в форме (`.budget-row-recurrence-active`) — залитые часы (`FILL 1` на
  самом icon-span, т.к. класс `.material-symbols-outlined` перебивал наследование
  с кнопки). Чисто CSS. `verify_frontend` budget desktop/mobile — без console-ошибок.
