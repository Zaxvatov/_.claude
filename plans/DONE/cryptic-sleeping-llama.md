# План: миграция личного бюджета из Excel в Assistant-UI (MVP)

## Контекст

Семейный бюджет ведётся в Excel-файле (`Бюджет.xlsx`, путь не хардкодим):
- **11 листов**, ~2900 записей в `Журнал операций`, ~30 строк в `Категории`, листы с большим числом вычисляемых столбцов (`Продукты` — 27 col, `Аналитика` — 49 col, `Алименты` — 804 col).
- Excel-формулы хрупкие, привязаны к именованным диапазонам и Excel Tables; нет AI-интеграции; неудобно с мобильных устройств.

**Цель MVP**: перенести 4 вкладки (`Категории`, `Журнал операций`, `Продукты`, `Аналитика`) в Assistant-UI с **визуально-идентичным форматированием** (шапки, ширины, формат дат и чисел, скрытие нулей, conditional formatting), импортировать всю историю операций, источник истины переезжает в БД.

**Документ организован как спецификация миграции Excel-логики** (Excel-contract), а не как заметки нового web-раздела. План адресует ревью Codex: курсы валют, схема балансов, разделение счетов и derived CHF, поле `movement`, нормализация категорий, полный layout `Продукты`/`Аналитика`, строгие acceptance, правила Assistant-UI AGENTS.

### Согласованные с пользователем решения

| # | Решение |
|---|---|
| Хранилище | Новый файл `assistant-ui/data/budget.db` (SQLite + WAL, см. tech-stack-choices §3). Не смешивать с `pm-mcp` БД. |
| История | Импортируем все ~2900 записей одноразовым скриптом. |
| UI журнала | Только-чтение таблица для истории/плана; добавление и редактирование — через md-dialog с формой. |
| Зависимости вне MVP | Алименты и ABC как листы не переносим. Нужные параметры переезжают как **константы** в `Настройки`. |
| Подвкладки `/budget` | Tabs внутри одной страницы (Material Web `<md-tabs>`). |
| Курсы валют | Excel-compatible chain (расчёт из соседних столбцов журнала с протягиванием предыдущего). Внешний API — будущая опция. |
| Балансы | Сырьё в `transactions`, материализованный баланс по (transaction, account) с пересчётом «от точки изменения». Window-function-инвариант проверяется в тестах. Без блокчейна. |
| Аналитика | Серверный пересчёт из `transactions`, read-only HTML. |
| `/settings` | Общая страница настроек Assistant-UI; пока внутри только бюджет; позже добавятся другие группы. |
| Excel-путь в импортёре | Обязательный CLI-аргумент `--xlsx-path`. Никаких хардкодов абсолютных путей (AGENTS root §E.3). |
| Movement | Поле `movement TEXT NULL` в `transactions`. Импорт сохраняет Excel `S` как есть (включая пустые значения для `Перевод`, разнонаправленные для `Долг`). Для новых операций UI **предлагает default** на основе типа (`+` для `Доход`/`Возврат`, `-` для `Расход`/`Накопление`), но пользователь может изменить или оставить пусто. Историю не переписываем. |
| Категории | Нормализация при импорте: trim, привести варианты вида «Хоз. Товары» к «Хоз. товары», лог расхождений в отчёт импорта. |
| Sort | `sort_index INTEGER` в `transactions` — порядок Excel-строк сохраняется; для новых записей — autoincrement в пределах даты. |
| Reconciliation | Сверка конечных балансов с Excel; при отклонении импортёр корректирует `opening_balance`. |
| Аналитика, тоггл A8 | Перенести как UI-переключатель «фикс. периоды (7 строк) / все месяцы». |
| Аналитика, шум | Не переносим: пустые helper-блоки `AI..AW`, плейсхолдеры «Столбец/Столбец2..4». |
| Аналитика, итог | Строка «Итог» (A62) — footer-строка таблицы. |

## Соответствие правилам репозитория

| Правило | Адресовано |
|---|---|
| AGENTS root §E.1 | Все комментарии/докстринги — русский. |
| AGENTS root §E.2 | Имена переменных/функций — английский. |
| AGENTS root §E.3 | Запрещены hardcoded paths. Excel путь — CLI; БД — `Path(__file__).parents[2] / "data" / "budget.db"`. |
| AGENTS root §K.2 | После approval плана — создать PM-MCP задачу для `assistant-ui/`; status `К выполнению`. Зависимостей нет. |
| AGENTS root §J.1 | Все коммиты в `main`, без PR без явного запроса. |
| AGENTS root §M.2/M.3 | `uv run ruff check .` и `uv run pytest` обязательны. |
| AGENTS root §Q.1 | Источник истины переезжает в БД целиком; xlsx остаётся как архив, но из веб-сценариев не читается. |
| `assistant-ui` AGENTS §«Architecture invariants» | Routes остаются в `app/main.py`. Шаблоны в `app/templates/`. Никаких inline HTML строк. |
| `assistant-ui` AGENTS §«UI and Design System» | Никаких новых project-level CSS файлов и правил. Новые стили — в `../Design-system/assets/`. |
| tech-stack-choices §3 | SQLite + WAL, FTS5 не нужен (бюджет не имеет search-кейсов). |
| tech-stack-choices §6 | FastAPI + Jinja2 + Material Web Components. |
| tech-stack-choices §7 | `uv sync --dev` для зависимостей; `openpyxl` в `pyproject.toml`. |

Отклонений от tech-stack-choices нет.

## Excel contract

Этот раздел — точная спецификация **что и как мы переносим** из каждого листа. Он же — основа acceptance criteria.

### Лист «Категории» — справочник

Видимые столбцы данных:
- `A` Столбец1 — длинная расшифровка категории (опционально; используется как hover/tooltip).
- `B` Категория — короткая группа.
- `C` Статья — название статьи (UNIQUE после нормализации).
- `E` Вид — `Постоянная` / `Переменная`.
- `F` Период планирования — `Ежемесячно` / `По требованию`.
- `G` Срок планирования — `Краткосрочные (ежемесячно)` / `Среднесрочные (несколько раз в год)` / `Долгосрочные`.
- `H` Вид потребностей — `Базовые` / `Блага` / `Развлечения (эмоции)`. **Может быть пустым** (например, `Услуги`), что не означает «не отображать в Аналитике».

В таблице `categories` дополнительно появляются два поля **на основе Excel-структуры**, заполняемые импортом:
- `show_in_analytics BOOL` — TRUE для всех статей, которые в Excel включены в колонки `Аналитики` `B..AH` (в т.ч. доходных `Зарплата`/`Фриланс` и статей с пустым `need_kind`).
- `analytics_measure TEXT` — `'income_chf_total'` для доходных статей, `'expense_chf_total'` для расходных. Определяется по знаку соответствующих транзакций в `Журнал_операций`.

**НЕ переносим**: `D` Deutsch (по требованию).

Вспомогательные dropdown-списки на том же листе:
- `J` — список «Вид потребностей» (Базовые, Блага, Развлечения (эмоции)). Дублирует значения из `H`, используется Excel-ом для data validation.
- `M` — список «Движение» (`+`, `-`).
- `N` — список «Тип операции» (Расход, Доход, Перевод, Накопление, Долг, Возврат).

При импорте `J/M/N` мигрируем в отдельные малые таблицы `need_kinds`, `movements`, `operation_types`.

### Лист «Журнал операций» — основная таблица

Заголовки строки 2. Excel-Table `Журнал_операций` (диапазон `A2:AD2904`; столбцы `AE..AJ` — за пределами таблицы, плейсхолдеры «Столбец1/Столбец2»).

Видимые «сырьевые» столбцы (вводятся пользователем):

| Excel | Имя | Тип | Excel формат | Описание |
|---|---|---|---|---|
| B | `op_date` | DATE | `ddd dd/mm` | Дата операции. |
| E | `inc_raiff_chf` | NUMERIC(12,2) | `0.00;\-0.00;;@` (скрывать нули) | Поступление на Raiff (CHF). |
| F | `inc_mono_uah` | NUMERIC(12,2) | то же | Поступление на Mono (UAH). |
| H | `inc_cash_chf` | NUMERIC(12,2) | то же | Поступление в наличные (CHF). |
| I | `inc_tks_rub` | NUMERIC(12,2) | то же | Поступление на ТКС (RUB). |
| K | `exp_raiff_chf` | NUMERIC(12,2) | то же | Расход с Raiff (CHF). |
| L | `exp_mono_uah` | NUMERIC(12,2) | то же | Расход с Mono (UAH). |
| N | `exp_cash_chf` | NUMERIC(12,2) | то же | Расход наличными (CHF). |
| O | `exp_tks_rub` | NUMERIC(12,2) | то же | Расход с ТКС (RUB). |
| Q | `exp_children_chf` | NUMERIC(12,2) | `#,##0.00` | Расход на детей (CHF). |
| S | `movement` | TEXT | `+` / `-` | Знак движения; правило ниже. |
| T | `op_type` | TEXT (FK) | строка | Тип операции (FK на `operation_types`). |
| U | `category_id` | INT (FK) | строка | FK на `categories.id` (по `Статья`). |
| W | `comment` | TEXT | строка | Свободный комментарий. |

Скрытые столбцы Excel (`G`, `J`, `R`, `X`, `Y`, `AB`) — **вычисляемые**. В БД материализуются явно (`fx_chf_uah`, `fx_chf_rub`, `income_chf_total`, `expense_chf_total`) или вычисляются на лету в SQL/service layer (`G inc_mono_chf = F/X`, `AB balance_mono_chf = balance_mono_uah / fx_chf_uah`). Все эти величины доступны через детали транзакции (`GET /api/budget/transactions/{id}`) и в тестах. В основной UI-таблице не показываются.

Балансовые столбцы Excel (`Z`, `AA`, `AC`, `AD`) — вычисляемые, в БД материализуются в `account_balance_after` (см. ниже).

Колонка V «Вид потребности» — derived lookup в `categories.need_kind`, не хранится.

#### Excel→derived (что мы пересчитываем при импорте и при чтении)

| Excel | Имя | Формула Excel (сокращённо) |
|---|---|---|
| A | `week_iso` | `ISOWEEKNUM(B)` |
| C | `daily_food_budget` | если дата та же что в предыдущей строке → 0; иначе `monthly_food_budget(date) / days_in_month(date)`. Месячный бюджет зависит от даты: **1722 для `date <= 2023-04-14`, 825 для `date >= 2023-04-15`** (Excel serial 45030 = 2023-04-14, проверено) — см. `app_settings`. |
| D | `daily_children_budget` | если дата та же → 0; иначе если `date < separation_date` (2023-04-15) → 0; иначе по дню недели (Fri/Sat/Sun) и формулам алиментов. |
| G | `fx_chf_uah_inc_link` (UAH→CHF при доходе) | `F/X` (`IFERROR(F/X, 0)`) — это **не курс**, а CHF-эквивалент дохода `F`. |
| J | `income_chf_total` | `E + G + H` (т. е. CHF-эквивалент всех доходов). |
| M | `exp_mono_chf` | `ROUND(L/X, 2)` — CHF-эквивалент расхода Mono. |
| P | `exp_tks_chf` | `ROUND(O/Y, 2)` — CHF-эквивалент расхода ТКС. |
| R | `expense_chf_total` | 0 если `op_type='Перевод'`, иначе `K + M + N + P`. |
| V | `need_kind` | lookup по `category_id`. |
| X | `fx_chf_uah` | Курс CHF/UAH в этой строке. Логика chain (см. ниже). |
| Y | `fx_chf_rub` | Курс CHF/RUB в этой строке. Логика chain. |
| Z | `balance_raiff_chf` | `Z[prev] + E − (K если op_type != 'Накопление' иначе 0)`. |
| AA | `balance_mono_uah` | `AA[prev] + F − L`. |
| AB | `balance_mono_chf` | `ROUND(AA / X, 2)`. |
| AC | `balance_cash_chf` | `AC[prev] + H − N`. |
| AD | `balance_total_chf` | `Z + AB + AC` (Raiff + Mono в CHF + Cash). ТКС в `AD` в Excel не учтён — копируем поведение 1:1; задокументировать. |

#### Курсы валют (X, Y) — Excel-compatible chain

```
X[r] = ROUND( IF( IFERROR(F[r]/K[r], 0) = 0,
                  X[r-1],                    -- протянуть предыдущий
                  F[r]/K[r] ), 2 )
Y[r] = IFERROR( ROUND( IF( IFERROR(I[r]/K[r], 0) = 0,
                            Y[r-1],
                            I[r]/K[r] ), 2 ),
                0 )
```

При импорте: проходим транзакции в порядке `(op_date, sort_index)`; для каждой строки вычисляем `fx_chf_uah` и `fx_chf_rub` и сохраняем в `transactions.fx_chf_uah/fx_chf_rub`. Это **точно копирует Excel** и при отсутствии конвертационной операции в день, курс протягивается с прошлого.

В будущем — отдельный adapter `services/fx_rates.py` с провайдером `exchangerate.host`; он работает только когда явно включён через `app_settings.fx_provider='api'`. На MVP `fx_provider='excel_chain'`.

### Лист «Продукты» — точный layout

Лист не «один в один», а с конкретной шапкой управления. Полный layout:

```
Строка 1 (заголовки группы):
  D1:E1 (merged) = "План на конец дня"
  F1 = False                                  -- флаг toggle «показать факт»
  O1..T1 = "1 день", "2 дня", ..., "6 дней"

Строка 2 (заголовки таблицы + панель параметров):
  A2..H2 = "Дата" "На день я" "На день дети" "Мой" "Общий" "Факт" "D" "Бюджет"
  I2 (hidden) = "День недели"
  K2 = "<месяц yyyy>" формула =EDATE(44866, K4)
  M2:M3 (merged) = "Плановый бюджет"
  N2 = 300 (фактический месячный бюджет CHF)
  O2..T2 = N2/days, *2, *3, *4, *5, *6 (прогноз 1-6 дней)

Строка 3+ (данные по дням):
  A: дата (формула EOMONTH(K2,-1) + n)
  B: норма на день для меня (из N2 или O4)
  C: норма на детей (из Алименты по дате)
  D: «Мой» план — накопительный остаток
  E: «Общий» план — учитывает детей
  F: Факт — SUMIFS по transactions (категория «Продукты» + дата)
  G: D = Бюджет − Общий (отклонение)
  H: Бюджет — динамический баланс
  I (hidden): день недели

Строка 4 (дополнительные параметры):
  K4 = offset месяца от октября 2022 (для пагинации между месяцами)
  M4 = True/False — флаг «плановый бюджет» (переключает B/D/E)
  N4 = 750 (плановый месячный бюджет CHF)
  O4..T4 = N4/days, *2..*6 — плановый прогноз
```

Web-замена:

- **Контролы сверху**: селектор месяца (md-outlined-select), переключатель «Плановый бюджет» (md-switch), отображение текущего бюджета и прогноза 1-6 дней.
- **Таблица 30/31 строка**: A2..H2 столбцы; столбец I (день недели) — не отображается (как в Excel), но доступен в API. Conditional formatting: D < 0 → красный, F > B → красный.
- Зависимость от Алименты — через `app_settings.alimony_*` и `app_settings.separation_date`. Без листа Алименты.
- Серверный расчёт через `services/products.py`. Read-only в MVP.

### Лист «Аналитика» — точный layout

```
Колонки B..AH = статьи (включая ДОХОДНЫЕ: Зарплата/Фриланс, и расходные с пустым
                 need_kind вроде «Услуги»). Список фиксируется при импорте полем
                 categories.show_in_analytics, и categories.analytics_measure
                 указывает, какую агрегатную величину использовать
                 (income_chf_total для доходных, expense_chf_total для расходных).

Строки 1-7 — метрики (точные формулы Excel):
  1: Регулярность       = COUNTIF(col > 0) / total_months
  2: Среднее арифметическое = AVERAGEIF(col, ">0")  (среднее ненулевых месячных)
  3: Медиана 30         = MEDIAN(IF(col <> 0, col))  (медиана ненулевых,
                          НЕ «последние 30»)
  4: Вер-ть дневная     = UNIQUE_DAYS_WITH_EVENT / DAYS_FROM_FIRST_EVENT_TO_TODAY
  5: Вер-ть месячная    = 1 - (1 - row4) ^ 30
  6: Прогноз на мес     = row5 * row3
  7: Отклонение         = STDEV.S(non-zero monthly values)

Строки 8-9 — управление:
  A8 = True/False — флаг «использовать фиксированные периоды»
  B8 = True/False — связанный флаг (роль уточним при имплементации)
  A9 = "Дата" — заголовок столбца дат

Строки 10-16 — фиксированные периоды (если A8=True):
  Конкретные даты из условий пользователя (формулы =IF($B$8=TRUE, serial, ""))

Строки 17-61 — динамический список месяцев:
  Шаг 1 месяц от A17 = 2023-05-01, до текущего месяца минус 1
  (формулы IFERROR + DATE(YEAR, MONTH+1, 1))

Строка 62 = "Итог" — суммы по столбцам за весь период

Колонки AI..AW — пустые helper-блоки (НЕ переносим)
Столбцы «Столбец», «Столбец2..4» — пустые плейсхолдеры Excel-Table (НЕ переносим)
```

Web-замена:

- Заголовок столбцов: `SELECT item, analytics_measure FROM categories WHERE show_in_analytics = 1 AND active = 1 ORDER BY sort_order` (сохраняет Excel-перечень дословно).
- 7 строк метрик — серверный пересчёт через `services/analytics.py`. Для каждой статьи берём `income_chf_total` или `expense_chf_total` (по `analytics_measure`) и группируем по месяцам.
  - Регулярность, среднее, прогноз — SQL CTE с агрегатами.
  - Медиана 30 и stdev — выгружаем месячные значения и считаем в Python (`statistics.median`, `statistics.stdev`).
  - Вер-ть дневная — `COUNT(DISTINCT op_date) / (today - first_event_date)`.
- Переключатель A8 → UI-toggle: «Фиксированные периоды» (A8=True) / «Все месяцы» (default, A8=False). Список фиксированных периодов в `app_settings.analytics_fixed_periods_json`.
- Динамический список месяцев — SQL `MIN(op_date) .. (TODAY() − 1 month)`.
- Итоговая строка — footer таблицы.
- Conditional formatting — см. раздел «Conditional formatting» ниже.

### Conditional formatting Excel — 5 правил, 1:1 в CSS

Извлечены из xlsx (openpyxl conditional_formatting). Переносим как CSS-классы (правила в Design-system).

| # | Где | Excel-правило | CSS-замена |
|---|---|---|---|
| 1 | Журнал, `A3:A2903` | `AND(ISNUMBER(B3), TRUNC(B3) <= TODAY())` — подсветить № недели для прошедших дат | `.budget-cell-past` (фон `surface-variant`). |
| 2 | Журнал, `B3:B2903` | `FLOOR(B3,1)=TODAY()` (сегодня) + `ABS(B3-TODAY()-MOD(-TODAY()+1,7)+3)<=3` (текущая неделя) | Сегодня → `.budget-row-today` (более тёмный фон + bold). Текущая неделя → `.budget-row-this-week` (светлее). |
| 3 | Журнал, `E3:Q2903` и `Q2904` | `$B3 > TODAY()` — будущие плановые операции | `.budget-row-future` (курсив, opacity 0.7). |
| 4 | Журнал, `AD3:AD2903` | `colorScale` 3-цветный по `Total full` | `.budget-balance-cell` + JS-функция в шаблоне: вычисляет нормализованную позицию `(balance − min) / (max − min)` и интерполирует через CSS custom property `--balance-gradient-pos`, который красится через `linear-gradient` в Design-system. На сервере min/max вычисляются один раз и передаются в template context. |
| 5 | Продукты, `A3:I33`, `F3:F33` (`today` + текущая неделя), `G3:G33` (`colorScale` отклонение) | Аналогично правилам 2 и 4 | Те же классы `.budget-row-today`, `.budget-row-this-week`, `.budget-balance-cell`. |

Все классы и градиент-кейс — в `D:/GitHub/Design-system/assets/assistant-ui.css` (с проверкой через `DESIGN_SYSTEM.md`). Конкретные цвета берём из MD3 токенов: `--md-sys-color-tertiary-container` для «сегодня», `--md-sys-color-surface-variant` для прошлого, и т. д. Точные значения утверждаются при реализации (это уже UI-кладка, не план).

## БД (assistant-ui/data/budget.db)

Идемпотентная инициализация (как [pm-mcp-server/app/tasks_db.py:12-78](D:/GitHub/AI-Assistant/pm-mcp-server/app/tasks_db.py)). PRAGMA `journal_mode=WAL`, `foreign_keys=ON`. SCHEMA_VERSION константа.

### Справочники

- **`categories`**(`id` PK, `group_text`, `item` UNIQUE, `description`, `kind`, `planning_period`, `planning_horizon`, `need_kind`, `sort_order`, `active` BOOL DEFAULT 1).
- **`operation_types`**(`name` PK).
- **`movements`**(`name` PK, CHECK `name IN ('+', '-')`).
- **`need_kinds`**(`name` PK).
- **`accounts`**(`id` PK, `name`, `currency` TEXT — 'CHF'/'UAH'/'RUB', `opening_balance` NUMERIC, `opening_date` DATE, `active` BOOL).
  - Изначально: Raiff (CHF), Mono (UAH), Cash (CHF), ТКС (RUB). **«в CHF» — не счёт**, это derived view над Mono (см. ниже).
- **`app_settings`**(`key` PK, `value` TEXT, `kind` TEXT, `description` TEXT).

Ключи `app_settings` (значения проверены по xlsx):
- `monthly_food_budget_v1` = 1722
- `monthly_food_budget_v1_until` = `2023-04-14`  (Excel serial 45030)
- `monthly_food_budget_v2` = 825
- `monthly_food_budget_v2_from` = `2023-04-15`
- `monthly_food_budget_planned` = 750
- `separation_date` = `2023-04-15`  (именованный диапазон `Дата_раздельного_проживания` = `Алименты!I32`)
- `alimony_amount_chf` = 400  (из `Алименты!J32`)
- `alimony_weekday_factors_json` = `{"Fri": p34, "Sat": q34, "Sun": r34}`  (формулы в Excel — это разности времён `P33-P32` и т. д.; импортёр читает уже посчитанные значения через `data_only=True`)
- `fx_provider` = 'excel_chain'
- `analytics_fixed_periods_json` — JSON-массив дат (`A10..A16` Excel)

### Транзакции

```sql
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    op_date DATE NOT NULL,
    sort_index INTEGER NOT NULL,           -- порядок Excel-строк; для новых = max+1 в пределах дня
    op_type TEXT REFERENCES operation_types(name),   -- nullable: Excel содержит редкие строки с пустым T
    category_id INTEGER REFERENCES categories(id),
    movement TEXT REFERENCES movements(name),        -- nullable: Excel S пуст для большинства Переводов, бывает + или - для Долгов
    comment TEXT,

    -- доходы (raw, валютные)
    inc_raiff_chf NUMERIC(12, 2) DEFAULT 0,
    inc_mono_uah NUMERIC(12, 2) DEFAULT 0,
    inc_cash_chf NUMERIC(12, 2) DEFAULT 0,
    inc_tks_rub NUMERIC(12, 2) DEFAULT 0,

    -- расходы (raw, валютные)
    exp_raiff_chf NUMERIC(12, 2) DEFAULT 0,
    exp_mono_uah NUMERIC(12, 2) DEFAULT 0,
    exp_cash_chf NUMERIC(12, 2) DEFAULT 0,
    exp_tks_rub NUMERIC(12, 2) DEFAULT 0,
    exp_children_chf NUMERIC(12, 2) DEFAULT 0,

    -- курсы (chain: либо из самой строки, либо протянуты с предыдущей)
    fx_chf_uah NUMERIC(12, 4) NOT NULL,
    fx_chf_rub NUMERIC(12, 4) NOT NULL,

    -- автозаполняемые derived (избегают пересчёта на каждое чтение для критичных полей)
    income_chf_total NUMERIC(12, 2) NOT NULL,    -- Excel J
    expense_chf_total NUMERIC(12, 2) NOT NULL,   -- Excel R

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_tx_date_sort ON transactions(op_date, sort_index);
```

### Балансы

```sql
CREATE TABLE account_balance_after (
    transaction_id INTEGER NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    balance NUMERIC(12, 2) NOT NULL,
    PRIMARY KEY (transaction_id, account_id)
);
```

Адресовано к замечанию Codex №2: PK — составной, упорядочивание — по `(op_date, sort_index)`.

### FX rates (опционально, для будущего)

```sql
CREATE TABLE fx_rates (
    date DATE NOT NULL,
    base TEXT NOT NULL,
    quote TEXT NOT NULL,
    rate NUMERIC(12, 6) NOT NULL,
    source TEXT NOT NULL,                  -- 'excel_chain' | 'api:exchangerate.host'
    fetched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (date, base, quote, source)
);
```

В MVP таблица существует, но заполняется только при `fx_provider='api'`. Источник чтения курсов — поля в самой `transactions.fx_chf_*`.

### Балансовая модель: best-practice без блокчейна

Пользовательский вопрос: «может, блокчейн?». Не подходит:
- Личные данные одного пользователя — нет нужды в децентрализации.
- Записи в личном бюджете регулярно правятся (опечатки), а блокчейн immutability мешает.
- Hash-chain и PoW даёт нулевую выгоду для задачи «сумма на счёте».
- SQLite уже даёт ACID-гарантии.

Best practice, которую применяем:

1. **Source of truth = `transactions`** (raw). Никаких балансов в самих транзакциях.
2. **Материализованный баланс** в `account_balance_after` по каждой паре (transaction, account).
3. **Пересчёт «от точки изменения»**: при INSERT/UPDATE/DELETE одной транзакции — обновляется только хвост (`op_date, sort_index ≥ changed_point`). На 2900 строках при window-function-пересчёте — десятки миллисекунд.
4. **Инвариант**: материализованный `balance` == window-function-вычисление. Проверяется тестом на каждой N-й транзакции (`tests/test_balances.py`).
5. **(Опционально позже)** `account_daily_snapshot(date, account_id, balance)` — снимок на конец дня. Полезно при росте >50k транзакций; на MVP не нужен.

## UI

### Sidebar — общий partial

В `assistant-ui/app/templates/` сейчас несколько шаблонов (dashboard.html, overview.html, ideas.html, …) дублируют `<aside class="sidebar">` блок. В рамках задачи:

1. Создаём `templates/_sidebar.html` (один источник навигации).
2. Включаем `{% include "_sidebar.html" %}` во все основные шаблоны, передаём `active` через контекст: `templates.TemplateResponse(request, "...", {"active": "budget", ...})`.
3. Добавляем два новых пункта:

```html
<a href="/budget" class="{% if active == 'budget' %}active{% endif %}">бюджет <span>›</span></a>
<a href="/settings" class="{% if active == 'settings' %}active{% endif %}">настройки <span>›</span></a>
```

Активная подсветка через токен `--md-sys-color-primary-container` (CSS-правило добавляется в Design-system, см. ниже).

### `/budget` — страница с tabs

Шаблон `templates/budget.html`. Material Web `<md-tabs>` + `<md-primary-tab>`. Дефолт — Журнал. URL hash сохраняет состояние (`#journal`/`#products`/`#analytics`).

Контент tab'ов — Jinja partials, рендерятся в одном GET-запросе:
- `templates/_budget_journal.html`
- `templates/_budget_products.html`
- `templates/_budget_analytics.html`

#### Tab «Журнал операций»

**Тулбар сверху**:
- `<md-filled-button>` «Новая операция» → открывает md-dialog с формой.
- Фильтры: месяц, статья, тип операции (`<md-outlined-select>`).
- Поиск по комментарию (`<md-outlined-text-field>`).

**Форма ввода (md-dialog)** — кастомный диалог (не помещается в стандартный `assistantConfirm`/`assistantTextInput` из [_assistant_head.html](D:/GitHub/AI-Assistant/assistant-ui/app/templates/_assistant_head.html)). Поля:
- Дата
- Тип операции (md-outlined-select из `operation_types`)
- Статья (md-outlined-select из `categories` с поиском)
- Движение (md-outlined-select, по умолчанию = из правила)
- Комментарий
- Группы сумм по валютам (Raiff CHF, Mono UAH, Cash CHF, ТКС RUB) для доходов и расходов. Пользователь заполняет только нужное.
- Кнопка «Сохранить» → POST `/api/budget/transactions` → пересчёт `account_balance_after` для хвоста.

**Таблица истории** (read-only, `<table class="data-table budget-journal-table">`), порядок столбцов = Excel:

| # | Заголовок | Источник | Формат |
|---|---|---|---|
| 1 | № нед | `week_iso` | целое |
| 2 | Дата | `op_date` | `ddd dd/mm` локаль ru-RU |
| 3 | Бюджет на день | derived `daily_food_budget` | `0.00`, нули скрыты |
| 4 | Продукты на детей | derived `daily_children_budget` | то же |
| 5 | Raiff+ | `inc_raiff_chf` | `0.00;\-0.00;;` нули скрыты |
| 6 | Mono+ | `inc_mono_uah` | то же |
| 7 | Cash+ | `inc_cash_chf` | то же |
| 8 | ТКС | `inc_tks_rub` | то же |
| 9 | Raiff- | `exp_raiff_chf` | то же |
| 10 | Mono- | `exp_mono_uah` | то же |
| 11 | в CHF | derived `exp_mono_chf` | то же |
| 12 | Cash- | `exp_cash_chf` | то же |
| 13 | TKC | `exp_tks_rub` | то же |
| 14 | Руб в CHF | derived `exp_tks_chf` | то же |
| 15 | Дети | `exp_children_chf` | `#,##0.00` |
| 16 | Движение | `movement` | `+`/`-` |
| 17 | Тип операции | `op_type` | строка |
| 18 | Статья | `categories.item` | строка |
| 19 | Вид потребности | `categories.need_kind` | строка |
| 20 | Комментарий | `comment` | строка |
| 21 | Raiff | `account_balance_after` Raiff | финансовый |
| 22 | Моно | `account_balance_after` Mono UAH | финансовый |
| 23 | Cash | `account_balance_after` Cash | финансовый |
| 24 | Total full | derived: Raiff + balance_mono_chf + Cash | финансовый |

Скрытые в Excel столбцы (`G income_chf_total/uah_to_chf`, `J income_chf_total`, `R expense_chf_total`, `X fx_chf_uah`, `Y fx_chf_rub`, `AB balance_mono_chf`) **видны в детали транзакции** (popover при клике на строку), но не в основной таблице.

Серверная пагинация по 200 строк + ленивая подгрузка по Intersection Observer (2900 строк нельзя рендерить в DOM сразу). Группировка по дате как в Excel: если та же что выше — повтор не выводится (CSS-правило в Design-system).

Conditional formatting (CSS-классы на основе значений):
- `op_type='Перевод'` → строка серая
- `expense_chf_total > 0` без категории → строка-предупреждение
- `balance_X < 0` → ячейка баланса красная

#### Tab «Продукты»

Шаблон `_budget_products.html`. Layout строго по Excel (см. Excel contract):

- **Верхняя панель управления**:
  - Селектор месяца (`<md-outlined-select>`), default — текущий.
  - Переключатель «Плановый бюджет» (`<md-switch>`) ↔ `M4`.
  - Карточка «Бюджет месяца» (`N2`=300 фактический или `N4`=750 плановый).
  - Прогноз 1-6 дней (`O2..T2` или `O4..T4`).

- **Таблица 30/31 строка**, столбцы A2..H2:
  - Дата (`ddd dd/mm`)
  - На день я
  - На день дети (из `app_settings.alimony_*` и дня недели)
  - Мой план
  - Общий план
  - Факт (SUMIFS по transactions: категория = «Продукты», `op_date = строка`)
  - D (отклонение Бюджет − Общий, красный при D<0)
  - Бюджет (динамический баланс месяца)

В MVP — read-only. Изменения значений базы (бюджеты, алименты) идут через `/settings → Параметры`.

#### Tab «Аналитика»

Шаблон `_budget_analytics.html`. Layout строго по Excel:

- **Верх**: переключатель режима «Все месяцы» / «Фикс. периоды» (тоггл A8). Список фиксированных периодов — из `app_settings`.
- **Заголовок столбцов**: статьи из `categories` (фильтр по типу расхода и `active=1`).
- **Строки 1-7**: 7 метрик (см. Excel contract). Расчёт на сервере (`services/analytics.py`).
- **Строки динамические**: список месяцев `MIN(op_date)..TODAY()−1m`. Если выбран режим «Фикс. периоды» — выводится короткий заданный список.
- **Footer**: «Итог» — суммы по столбцам за весь период.

Read-only. Кэш не делаем — пересчёт при заходе на tab (для 2900 строк это ~100ms).

### `/settings`

Шаблон `templates/settings.html` с tabs:
- Категории (default)
- Параметры
- Счета
- Списки

#### Tab «Категории»
Таблица: Группа, Статья, Вид, Период планирования, Срок планирования, Вид потребностей. Кнопка «Добавить» открывает md-dialog с формой. Редактирование/удаление по кнопкам в строке. UNIQUE(`item`) обеспечивается SQL.

#### Tab «Параметры»
Формы для всех ключей `app_settings`:
- Месячный продуктовый бюджет (фактический и плановый).
- Дата перехода с прошлого бюджета на текущий.
- Дата раздельного проживания.
- Размер алиментов (CHF/мес) и дневные факторы Fri/Sat/Sun.
- Фиксированные периоды для Аналитики.

#### Tab «Счета»
CRUD: имя, валюта, `opening_balance`, `opening_date`, флаг `active`. Изменение `opening_balance` запускает пересчёт `account_balance_after` для всех транзакций счёта (редкая операция).

#### Tab «Списки»
CRUD для `operation_types`, `need_kinds`, `movements`. Малые таблицы; пересечение со словарём операций не разрешается на UI-уровне (нельзя удалить тип, который используется).

## Backend

Структура внутри `assistant-ui/app/budget/` — **модуль с подмодулями, без `routes.py`** (route остаются в [main.py](D:/GitHub/AI-Assistant/assistant-ui/app/main.py), как требует AGENTS):

```
app/budget/
  __init__.py
  db.py                    # CREATE TABLE IF NOT EXISTS, SCHEMA_VERSION
  schemas.py               # Pydantic: Transaction, Category, Account, Settings...
  repositories.py          # CRUD над таблицами (тонкий слой над SQL)
  services/
    transactions.py        # CRUD + пересчёт account_balance_after
    balances.py            # window-function-инвариант, batch recompute
    analytics.py           # серверный пересчёт сводок Аналитики
    products.py            # расчёт плана/факта для tab «Продукты»
    fx_rates.py            # (опц.) внешний API; в MVP не активен
    categories.py
    settings.py
  importer.py              # импорт из xlsx
```

В `app/main.py` добавляются эндпоинты:

| Метод | Path | Описание |
|---|---|---|
| GET | `/budget` | Страница с tabs. |
| GET | `/settings` | Страница настроек. |
| GET | `/api/budget/transactions` | Список (пагинация, фильтры). |
| GET | `/api/budget/transactions/{id}` | Детали (включая скрытые derived). |
| POST | `/api/budget/transactions` | Создать. |
| PUT | `/api/budget/transactions/{id}` | Обновить. |
| DELETE | `/api/budget/transactions/{id}` | Удалить. |
| GET | `/api/budget/products` | Данные для tab Продукты (по месяцу). |
| GET | `/api/budget/analytics` | Данные для tab Аналитика. |
| GET/POST/PUT/DELETE | `/api/budget/categories[/{id}]` | CRUD категорий. |
| GET/POST/PUT/DELETE | `/api/budget/accounts[/{id}]` | CRUD счетов. |
| GET/PUT | `/api/settings/budget` | Чтение/обновление `app_settings`. |

Все endpoints возвращают JSON для XHR-вызовов и HTML для страниц. SSE и polling не используются (UI-обновление — после успешного XHR).

## CSS / Design-system

Никаких новых правил в `assistant-ui/app/static/*.css`. Все стили (новые классы `.budget-journal-table`, `.budget-table-cell-currency`, `.budget-row-transfer`, `.budget-amount-negative`, `.sidebar a.active`, …) добавляются в [Design-system/assets/assistant-ui.css](D:/GitHub/Design-system/assets/assistant-ui.css) (или новый файл в `Design-system/assets/`, если нужна изоляция). Изменения коммитятся в **репозитории Design-system**, а не в Assistant-UI.

Перед редактированием — прочитать `D:/GitHub/Design-system/DESIGN_SYSTEM.md`.

## Импорт из Excel

Скрипт `assistant-ui/scripts/import_budget_xlsx.py`. Запуск:

```powershell
cd D:/GitHub/AI-Assistant/assistant-ui
uv run python -m scripts.import_budget_xlsx --xlsx-path "G:\Мой диск\Личные данные\Бюджет.xlsx" --db-path data/budget.db
```

CLI-аргументы обязательны (никаких хардкодов). Допустимый необязательный `--dry-run` — не пишет в БД, только формирует отчёт.

Алгоритм:

1. **Инициализация БД**. `db.init()` — создаёт схему.

2. **Категории** (`Категории`, строки 2..N):
   - Считываем A,B,C,E,F,G,H.
   - Нормализация `item`: trim, схлопывание двойных пробелов, привести к каноническому варианту по словарю опечаток (`{"Хоз. Товары": "Хоз. товары", …}`).
   - Логируем все случаи нормализации в `import_report.txt`.
   - UPSERT по нормализованному `item`.

3. **Списки** из `J`/`M`/`N` — в `need_kinds`, `movements`, `operation_types`.

4. **Settings** — захардкоженные «обнаруженные» значения:
   - `monthly_food_budget_v1=1722`, `monthly_food_budget_v1_until=2023-04-23`, `monthly_food_budget_v2=825` — из формулы C Журнала.
   - `monthly_food_budget_planned=750` — из Продукты!N4.
   - `separation_date` — из именованного диапазона `Дата_раздельного_проживания` (если есть).
   - `alimony_amount_chf` — из `Алименты!J32`.
   - `alimony_weekday_factors_json` — из `Алименты!P34:R34`.
   - `analytics_fixed_periods_json` — из `Аналитика!A10:A16`.
   - `fx_provider='excel_chain'`.

5. **Accounts** — Raiff CHF, Mono UAH, Cash CHF, ТКС RUB. `opening_balance=0` (подкорректируется в п. 8).

6. **Transactions** (`Журнал операций`, строки 3..2904):
   - Игнорируем вычисляемые столбцы Excel; читаем только raw (`B, E, F, H, I, K, L, N, O, Q, S, T, U, W`).
   - `sort_index` = `excel_row_no - 2` (стабильный порядок).
   - Резолвим `op_type` → `operation_types.name` (если значение незнакомо — лог + ошибка).
   - Резолвим `U Статья` → `categories.id` через нормализацию (с тем же словарём опечаток).
   - `movement` берём из S (валидируем по правилу: `'+' для Доход/Возврат, '-' для иных`); при расхождении — лог.
   - Вычисляем `fx_chf_uah/fx_chf_rub` по Excel-chain.
   - Вычисляем `income_chf_total` (Excel J), `expense_chf_total` (Excel R).
   - INSERT.

7. **Пересчёт балансов** — `services/balances.py.recompute_all()`. Заполняет `account_balance_after` для всех транзакций.

8. **Reconciliation (двухпроходный)**.
   - Открываем Excel второй раз с `data_only=True`, читаем последние ненулевые значения `Z`, `AA`, `AC`, `AD` (конечные балансы Excel).
   - Сравниваем с финальными `account_balance_after` из шага 7.
   - При расхождении (>0.01) — корректируем `accounts.opening_balance` на дельту.
   - **Повторно вызываем `services/balances.py.recompute_all()`** — пересчёт хвоста после изменения opening.
   - **Повторно сверяем** финальные балансы с Excel. Если расхождение всё ещё остаётся (например, противоречие в порядке транзакций или потерянная конвертация) — лог и **остановка скрипта** с диагностикой; ручное вмешательство.

9. **Отчёт** — `import_report.txt` (рядом с БД):
   - Число обработанных строк по типам операций.
   - Список нормализаций категорий.
   - Балансы Excel vs БД (по счетам).
   - Резолверские конфликты (unknown op_type/category).
   - Время импорта.

## Acceptance criteria (строгие)

Используются Codex-замечания №8.

### Контрактная сверка

- [ ] **Видимые/скрытые столбцы** Журнала в БД и UI идентичны Excel-таблице: видимые — в основной таблице, скрытые — только в детали.
- [ ] **Ширины колонок** в UI: соответствуют относительным ширинам Excel (округление до ближайшего CSS-shorthand). Допустимое отклонение — ±20%.
- [ ] **Merged ranges** `Продукты` (`D1:E1`, `M2:M3`) реализованы через `colspan`/`rowspan` в HTML.
- [ ] **Number formats**: даты — `ddd dd/mm` (ru-RU), деньги — два знака, минусы как `-0.00`, нули скрыты для столбцов с `0.00;\-0.00;;`.
- [ ] **Conditional formatting** — все 5 правил из раздела «Conditional formatting Excel» реализованы как CSS-классы и работают на живых данных. Проверка визуальной идентичности на конкретных тестовых датах (сегодня, прошлая неделя, будущее).

### Сверка derived

- [ ] Для случайной выборки из 100 транзакций сравнить все вычисляемые поля `C, D, G, J, M, P, R, V, X, Y, Z, AA, AB, AC, AD` БД ↔ Excel `data_only=True`. Допустимое отклонение — 0.01.
- [ ] `Продукты` для трёх месяцев (включая месяц до 2023-04-15 и после) — все непустые видимые ячейки `A2:H34` (вкл. строку «Всего» row 34) совпадают с Excel.
- [ ] `Аналитика` — все 7 метрик и итог для каждого видимого столбца совпадают.

### Функциональные

- [ ] Импорт скрипта без ошибок; `import_report.txt` зелёный.
- [ ] `SELECT COUNT(*) FROM transactions` ≈ 2902 (точное число фиксируется в отчёте импорта; ровно совпадает Excel↔БД).
- [ ] `GET /budget` отдаёт 200, видна таблица с >100 строк, переключение tabs работает.
- [ ] Форма «Новая операция» — POST → запись появляется в таблице.
- [ ] Редактирование операции — пересчёт балансов хвоста корректен (юнит-тест на инвариант).
- [ ] `GET /settings` → видна Категории-таблица ~30 строк; CRUD работает.

### Инвариант баланса

- [ ] Юнит-тест `tests/test_balances.py`: на каждой 25-й транзакции материализованный `account_balance_after.balance` == window-function-расчёт.
- [ ] Тест мутации: INSERT в середине дня → `sort_index` корректно сдвигает хвост; финальные балансы счетов = тем же, что без мутации (для эквивалентной операции).

### Code quality

- [ ] `uv run ruff check .` в `assistant-ui/` — зелёный.
- [ ] `uv run pytest` в `assistant-ui/` — зелёный.

### Frontend verification (skill `frontend-verification`)

- [ ] Скриншоты desktop + mobile для `/budget` (Журнал/Продукты/Аналитика) и `/settings`.
- [ ] Side-by-side сравнение строки Excel и строки таблицы — визуально идентичны по форматированию.
- [ ] Чек интерактивности: переключение tabs, открытие диалога, сохранение операции, переключение месяца в Продуктах.

## Этапы реализации

Все коммиты — в `main`, без PR, без веток (AGENTS J.1). Каждый этап — отдельный коммит с осмысленным сообщением на русском.

1. **Фундамент**:
   - Создать `_sidebar.html` и подключить во все страницы (рефактор существующих шаблонов).
   - Заглушки `/budget`, `/settings` с tabs (пустыми) — handlers в `app.main`.
   - Стили `.sidebar a.active`, базовая разметка табов — в Design-system.

2. **БД + Категории + Параметры + Счета**:
   - `app/budget/db.py`, `schemas.py`, `repositories.py`.
   - `services/categories.py`, `services/settings.py`.
   - Tabs «Категории», «Параметры», «Счета» в `/settings` работают (CRUD).

3. **Импорт Категории + Параметры**:
   - `scripts/import_budget_xlsx.py` — этапы 1-5.
   - `uv add openpyxl --dev` для импорта.

4. **Импорт транзакций + балансы + FX chain**:
   - `services/transactions.py`, `services/balances.py`.
   - Этапы 6-9 импортёра.
   - Tab «Журнал» (read-only).
   - Тесты `tests/test_balances.py`.

5. **Журнал CRUD**:
   - Форма «Новая операция» (md-dialog).
   - POST/PUT/DELETE endpoints.
   - Пересчёт хвоста.

6. **Продукты**:
   - `services/products.py`.
   - UI tab (read-only).

7. **Аналитика**:
   - `services/analytics.py`.
   - UI tab + тоггл A8.

В конце — frontend-verification всех страниц.

## После approval

По правилам `central-plan-workflow` и AGENTS K.4:

1. Создать **главный** PM-MCP work item в `assistant-ui/` через `mcp__PM-MCP-server__create_task` (status `К выполнению`, `project_path = D:/GitHub/AI-Assistant/assistant-ui`).
2. Создать **отдельный** PM-MCP work item в `D:/GitHub/Design-system/` (cross-subsystem, см. AGENTS K.4) для CSS-правил sidebar/budget-таблиц/conditional-formatting. Связать как зависимость через `link_task_dependency` с `dependency_project_path=Design-system`.
3. Переименовать файл плана в `<id>-cryptic-sleeping-llama.md` (где `<id>` — глобальный номер главной задачи в assistant-ui).
4. Обновить harness-ссылку в `~/.claude/plans/cryptic-sleeping-llama.md` на новое имя.
5. По завершении — записать outcome в AI-memory через `ai-memory-capture` (отдельные записи для `assistant-ui` и `Design-system`, при необходимости — summary в `portfolio`).

## Открытые вопросы (можно решить в процессе)

- **Sparklines** в Excel-ячейках (предупреждение openpyxl) — для MVP игнорируем, можно добавить позже через `<svg>` или Chart.js. Не блокер.
- **Conditional formatting в Excel** (предупреждение openpyxl) — реплицируем ручным набором CSS-правил (см. acceptance). Полный 1:1 не гарантируем.
- **Экспорт обратно в xlsx** — отложен.
- **Внешний API курсов** (`fx_provider='api'`) — post-MVP; адаптер заранее заложен.
- **Multi-user** — текущий Assistant-UI имеет `AuthMiddleware` для одного пользователя; так и используем. Данные не привязаны к user_id.
- **Алименты как полноценный модуль** — post-MVP. Сейчас замораживаем как `app_settings`.
