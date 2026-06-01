# План: гигиена ADR в монорепо AI-Assistant

## Context

В `docs/adrs/` лежат три принятых ADR (`0001`, `0002`, `0003`) — все актуальны
и согласованы с кодом, миграция M1 (монорепо + workflow + глобальная нумерация
задач) закрыта на 100%. Сами ADR трогать не нужно.

При этом по репозиторию рассыпан **технический долг ADR-гигиены**:

1. **Призраки.** В `pm-mcp-server/ARCHITECTURE.md` (две строки) и в
   `pm-mcp-server/docs/MCP_API.md` стоит ссылка `ADR-010` — такой ADR не
   существует. По контексту это `ADR-0003` (хранение задач в SQLite +
   глобальная нумерация).
2. **Старый формат `ADR-001` без zero-padding.** Шесть мест в `AGENTS.md`,
   корневом `README.md`, подсистемных `AGENTS.md` (ai-memory, pm-mcp-server)
   и двух файлах `ai-memory` (один — production-код, один — тест).
   Канонический формат — `ADR-0001` (zero-padded, как в самих файлах ADR
   и в `docs/adrs/README.md`).
3. **`docs/adrs/README.md` не содержит ADR-0003** в перечне текущих ADR.
4. **Подсистемные `README.md` не ссылаются на ADR.** Разработчик, открывший
   `ai-memory/`, `pm-mcp-server/`, `assistant-ui/`, `gateway/`, не видит,
   какое архитектурное решение её затрагивает.

ADR-011 нигде не упоминается. `ADR-012` встречается в
`assistant-ui/tests/test_api_endpoints.py` только как mock-значение поля
`title` цели — это не ссылка на ADR, не трогаем.

Цель плана — выровнять все перекрёстные ссылки на ADR в репозитории, не меняя
сами ADR.

## Scope

### Делаем

**A. Заменить призраки `ADR-010` на `ADR-0003`** (2 файла, 3 строки):

| Файл | Строка | Текущий фрагмент | После замены |
|------|--------|------------------|--------------|
| `pm-mcp-server/ARCHITECTURE.md` | 55 | `Project tasks после ADR-010 хранятся в SQLite` | `Project tasks после ADR-0003 хранятся в SQLite` |
| `pm-mcp-server/ARCHITECTURE.md` | 227 | `Источник правды по project tasks после ADR-010:` | `Источник правды по project tasks после ADR-0003:` |
| `pm-mcp-server/docs/MCP_API.md` | 32 | `Project tasks после ADR-010 хранятся в SQLite по PM_MCP_TASKS_DB_PATH;` | `Project tasks после ADR-0003 хранятся в SQLite по PM_MCP_TASKS_DB_PATH;` |

**B. Привести формат `ADR-001` → `ADR-0001`** (6 мест):

| Файл | Строка | Контекст |
|------|--------|----------|
| `AGENTS.md` | 250 | `see ADR-001, Шаг 7 for the migration plan` |
| `README.md` | 16 | `audit (см. ADR-001, ...)` |
| `pm-mcp-server/AGENTS.md` | 52 | `after ADR-001 step 5` |
| `ai-memory/AGENTS.md` | 65 | `after ADR-001 step 7` |
| `ai-memory/tests/test_db.py` | 248 | assertion на `'ADR-001 шаг 3: архивирована legacy история чата'` |
| `ai-memory/memory/db.py` | 405 | константа `archive_reason = 'ADR-001 шаг 3: архивирована legacy история чата'` |

⚠️ **Side-effect** по парам `db.py:405` + `test_db.py:248`. Эта строка
пишется в БД ai-memory в поле `archive_reason` при архивации legacy-истории
чата. В реальной БД уже могут лежать записи со значением `'ADR-001 шаг 3:...'`
— после замены они останутся в старом формате (новые архивации получат
`'ADR-0001 шаг 3:...'`). Это не функциональный регресс, только аудитный
разнобой. Если важна полная консистентность БД — отдельная задача миграции
`UPDATE memory SET archive_reason = REPLACE(...)`; в этот план её не
включаем.

**C. Дописать ADR-0003 в `docs/adrs/README.md`.** В раздел «Текущие ADR»
добавить третью строку.

**D. Кросс-ссылки в подсистемных `README.md` первого уровня.** В каждый
добавить короткую секцию «Связанные ADR» / «Related ADRs» (язык по
существующему языку README) со списком и относительными ссылками на ADR:

| Подсистема | README | ADR |
|------------|--------|-----|
| ai-memory | `ai-memory/README.md` | ADR-0001 (kinds taxonomy + миграция архивации chat-history) |
| pm-mcp-server | `pm-mcp-server/README.md` | ADR-0001, ADR-0002, ADR-0003 |
| assistant-ui | `assistant-ui/README.md` | ADR-0001 (UI+runtime, Conversation Store) |
| gateway | `gateway/README.md` | ADR-0001 (Gateway as single external ingress) |

### НЕ делаем (out of scope)

- Не меняем содержимое самих ADR `0001/0002/0003`.
- Не пишем новые ADR-0004/0005 — отдельная волна.
- Не пишем retroactive ADR для «призраков» 010/011/012.
- Не унифицируем язык ADR (0002/0003 на английском, 0001 на русском).
- Не правим `assistant-ui/tests/test_api_endpoints.py` (`"ADR-012"` там —
  fixture, не ссылка).
- Не делаем обратную миграцию `archive_reason` в БД ai-memory.

## Файлы к изменению (полный список)

1. `pm-mcp-server/ARCHITECTURE.md` — 2 строки.
2. `pm-mcp-server/docs/MCP_API.md` — 1 строка.
3. `AGENTS.md` (корневой) — 1 строка.
4. `README.md` (корневой) — 1 строка.
5. `pm-mcp-server/AGENTS.md` — 1 строка.
6. `ai-memory/AGENTS.md` — 1 строка.
7. `ai-memory/memory/db.py` — 1 строка-константа.
8. `ai-memory/tests/test_db.py` — 1 assertion.
9. `docs/adrs/README.md` — добавить ADR-0003 в перечень.
10. `ai-memory/README.md` — секция «Связанные ADR».
11. `pm-mcp-server/README.md` — секция «Связанные ADR».
12. `assistant-ui/README.md` — секция «Связанные ADR».
13. `gateway/README.md` — секция «Related ADRs».

Итого: 13 файлов, ~10 точечных правок + 4 коротких новых блока.

## Verification

1. **Грепы должны быть пустыми.**
   - `grep -rn "ADR-010" --exclude-dir=.git --exclude-dir=node_modules` → пусто.
   - `grep -rn "ADR-001\b" --exclude-dir=.git --exclude-dir=node_modules` → пусто.
   - `grep -rn "ADR-011\b"` → пусто (как и было).

2. **Грепы должны находить.**
   - `grep -rn "ADR-0001" --include="*.md" --include="*.py"` → ≥ 6 совпадений.
   - `grep -rn "ADR-0003" pm-mcp-server/` → ≥ 3 совпадения.

3. **Тесты ai-memory проходят:**
   ```
   cd ai-memory && uv run python -m unittest tests.test_db -v 2>&1 | tail -20
   ```

4. **Markdown-ссылки рабочие** — относительные `../docs/adrs/...md` в
   подсистемных README существуют.

5. **`docs/adrs/README.md` содержит упоминание ADR-0003.**

## После approval плана

После закрытия задачи переименовать draft в `<id>-adr-hygiene-cleanup.md`
согласно правилу A `~/.claude/CLAUDE.md`. Если задача в PM-MCP не заводится
(гигиена, тривиальный объём), оставить draft под текущим именем и удалить
после approval, как только outcome зафиксирован в AI-memory.
