# Формат данных для создания операции бюджета через ChatGPT

## Контекст

Этот документ описывает формат, в котором ChatGPT должен отправлять данные
для создания новой операции в бюджете — как через REST API (assistant-ui),
так и через будущий MCP write-tool (budget daemon на `127.0.0.1:8767`).

**Текущий статус**: прямая запись от ChatGPT в бюджет **не реализована**.
Budget MCP v1 предоставляет только read-tools. ChatGPT write-доступ к бюджету
запланирован как отдельная фаза с ADR (см. `replicated-popping-chipmunk.md`,
раздел Trust model). Этот документ фиксирует контракт заранее, чтобы будущий
имплементатор мог опереться на единый source of truth.

## Справочники (нужно запросить перед созданием операции)

ChatGPT обязан знать актуальные ID перед отправкой. Данные получаются через
read-tools budget MCP:

```
list_categories()       → [{id, item, group_text, need_kind, ...}]
list_operation_types()  → ["Расход", "Доход", "Перевод", "Накопление", "Долг", "Возврат"]
list_accounts()         → [{id, name, currency}]
```

Счета и их валюты (константы системы):
| Поле суммы       | Счёт    | Валюта |
|------------------|---------|--------|
| `inc_raiff_chf`  | Raiff   | CHF    |
| `inc_mono_uah`   | Mono    | UAH    |
| `inc_cash_chf`   | Наличные| CHF    |
| `inc_tks_rub`    | ТКС     | RUB    |
| `exp_raiff_chf`  | Raiff   | CHF    |
| `exp_mono_uah`   | Mono    | UAH    |
| `exp_cash_chf`   | Наличные| CHF    |
| `exp_tks_rub`    | ТКС     | RUB    |
| `exp_children_chf`| —      | CHF    |

## Формат через REST API (`POST /api/budget/transactions`)

Content-Type: `application/json`

### Обязательные поля

| Поле          | Тип          | Пример               | Описание                                        |
|---------------|--------------|----------------------|-------------------------------------------------|
| `op_date`     | string (ISO) | `"2026-06-07"`       | Дата операции                                   |
| `op_type`     | string (enum)| `"Расход"`           | Тип операции из `list_operation_types()`        |
| `category_id` | integer      | `12`                 | ID категории из `list_categories()`             |

### Опциональные поля

| Поле              | Тип           | Пример   | Описание                                    |
|-------------------|---------------|----------|---------------------------------------------|
| `movement`        | string / null | `"-"`    | `"+"` доход, `"-"` расход, `null` — для Перевода/Долга |
| `comment`         | string        | `"Икеа"` | Свободный комментарий                       |
| `inc_raiff_chf`   | number        | `0.00`   | Поступление на Raiff (CHF)                  |
| `inc_mono_uah`    | number        | `0.00`   | Поступление на Mono (UAH)                   |
| `inc_cash_chf`    | number        | `0.00`   | Поступление наличными (CHF)                 |
| `inc_tks_rub`     | number        | `0.00`   | Поступление на ТКС (RUB)                    |
| `exp_raiff_chf`   | number        | `0.00`   | Расход с Raiff (CHF)                        |
| `exp_mono_uah`    | number        | `0.00`   | Расход с Mono (UAH)                         |
| `exp_cash_chf`    | number        | `0.00`   | Расход наличными (CHF)                      |
| `exp_tks_rub`     | number        | `0.00`   | Расход с ТКС (RUB)                          |
| `exp_children_chf`| number        | `0.00`   | Расход на детей (CHF)                       |

Хотя бы одно поле суммы должно быть ненулевым.

### Правило `movement` по типу операции

| `op_type`    | `movement` по умолчанию | Примечание                                 |
|--------------|-------------------------|--------------------------------------------|
| Расход       | `"-"`                   | Обязательно                                |
| Доход        | `"+"`                   | Обязательно                                |
| Возврат      | `"+"`                   | Обязательно                                |
| Накопление   | `"-"`                   | Обязательно                                |
| Перевод      | `null`                  | Одновременно inc и exp на разных счетах    |
| Долг         | `"+"` или `"-"`         | Зависит от направления долга               |

### Примеры запросов

**Расход с Raiff в магазине:**
```json
{
  "op_date": "2026-06-07",
  "op_type": "Расход",
  "category_id": 12,
  "movement": "-",
  "exp_raiff_chf": 45.30,
  "comment": "Мигрос"
}
```

**Доход (зарплата) на Raiff:**
```json
{
  "op_date": "2026-06-01",
  "op_type": "Доход",
  "category_id": 1,
  "movement": "+",
  "inc_raiff_chf": 5200.00,
  "comment": "Зарплата июнь"
}
```

**Перевод Raiff → наличные:**
```json
{
  "op_date": "2026-06-07",
  "op_type": "Перевод",
  "category_id": 5,
  "movement": null,
  "exp_raiff_chf": 200.00,
  "inc_cash_chf": 200.00,
  "comment": "Снял наличные"
}
```

**Расход с Mono в гривнях (с явным курсом не нужен — chain подхватит из последней конвертационной операции):**
```json
{
  "op_date": "2026-06-07",
  "op_type": "Расход",
  "category_id": 22,
  "movement": "-",
  "exp_mono_uah": 850.00,
  "comment": "Аптека"
}
```

## Будущий формат через Budget MCP write-tool

Когда будет реализован write-доступ, ChatGPT вызовет MCP tool
`propose_budget_transaction` (или `create_budget_transaction` — зависит от решения
о proposal queue аналогично AI-memory write gateway).

Ожидаемая схема (по образцу `chatgpt-write-gateway.md`):

```json
{
  "name": "propose_budget_transaction",
  "description": "Submit a budget transaction for human review. The entry appears in the journal only after approval.",
  "inputSchema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["op_date", "op_type", "category_id"],
    "properties": {
      "op_date":          {"type": "string", "format": "date", "description": "ISO 8601 date, e.g. 2026-06-07"},
      "op_type":          {"type": "string", "enum": ["Расход", "Доход", "Перевод", "Накопление", "Долг", "Возврат"]},
      "category_id":      {"type": "integer", "description": "ID from list_categories()"},
      "movement":         {"type": ["string", "null"], "enum": ["+", "-", null]},
      "comment":          {"type": "string", "maxLength": 500},
      "inc_raiff_chf":    {"type": "number", "minimum": 0},
      "inc_mono_uah":     {"type": "number", "minimum": 0},
      "inc_cash_chf":     {"type": "number", "minimum": 0},
      "inc_tks_rub":      {"type": "number", "minimum": 0},
      "exp_raiff_chf":    {"type": "number", "minimum": 0},
      "exp_mono_uah":     {"type": "number", "minimum": 0},
      "exp_cash_chf":     {"type": "number", "minimum": 0},
      "exp_tks_rub":      {"type": "number", "minimum": 0},
      "exp_children_chf": {"type": "number", "minimum": 0}
    }
  }
}
```

Успешный ответ: `{"transaction_id": <int>, "status": "created"}` (direct write)
или `{"proposal_id": <int>, "status": "proposed"}` (если proposal queue).

## Валидация на стороне сервера

1. `op_date` не может быть в будущем более чем на 1 день (configurable).
2. Для `op_type = "Расход"` / `"Накопление"`: хотя бы одно `exp_*` > 0.
3. Для `op_type = "Доход"` / `"Возврат"`: хотя бы одно `inc_*` > 0.
4. Для `op_type = "Перевод"`: одновременно inc_* и exp_* на разных счетах.
5. Сервер выставляет `sort_index = MAX(sort_index)+1` для данной даты.
6. `fx_chf_uah` / `fx_chf_rub` — не принимаются от клиента, вычисляются по
   excel-chain из последней конвертационной операции.
7. `income_chf_total` / `expense_chf_total` — вычисляются сервером автоматически.

## Связанные документы

- `plans/DONE/cryptic-sleeping-llama.md` — полная схема БД и Excel-контракт
- `plans/DONE/replicated-popping-chipmunk.md` — архитектура budget module и trust model
- `plans/DONE/chatgpt-write-gateway.md` — образец proposal-queue для AI-memory
