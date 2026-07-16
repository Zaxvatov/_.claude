# Связь задач, бюджета и Google Calendar

Статус: реализован; ожидается повторная Google OAuth-авторизация и отдельный live smoke

PM-MCP:
- pm-mcp-server: #1194
- budget: #1195 (depends #1194)
- service_policy: #1196 (depends #1194, #1195)
- assistant-ui: #1197 (depends #1194, #1195, #1196)
- retrospective follow-up: #1198 (на согласовании)
- postscript плана и коммит доработки формы операций: #1201

## Цель

Последовательно реализовать:

1. Связь WorkItem с нулём или несколькими бюджетными операциями.
2. Инвариант: связанная задача получает `Готово` только после подтверждения всех связанных операций; завершение задачи атомарно подтверждает весь связанный набор.
3. Календарь с timed-зоной `00:00–23:59` и нижней зоной задач без времени той же даты.
4. Двустороннюю синхронизацию timed-задач с основным Google Calendar.

PM-MCP остаётся источником истины для задач и оркестрации, Budget — для финансового plan/fact, `calendar.db` — для Google-событий и состояния синхронизации. Новые framework и daemon не добавляются.

## Этап 1. Task ↔ Budget

- Создать tenant-scoped `task_budget_links(task_id, budget_transaction_id, created_at, updated_at)`.
- Пара `(task_id, budget_transaction_id)` уникальна; один budget transaction может принадлежать только одной задаче.
- WorkItem возвращает `budget_transaction_ids: list[int]`.
- Добавить `link_task_budget_transaction`, `unlink_task_budget_transaction`, `list_task_budget_transactions` и поиск задачи по transaction ID.
- При связывании проверять существование операции, tenant и отсутствие чужой связи.
- В карточке задачи показывать список операций, plan/fact, сумму, комментарий, переход в журнал, добавление и отвязку.
- В журнале бюджета показывать связанную задачу.
- Отвязка не снимает факт и не переоткрывает задачу.

## Этап 2. Завершение пары

- Новый ADR фиксирует: задача может быть `Готово`, только если все связанные операции имеют `fact_date`.
- Любой PM-MCP status-path перед `Готово` атомарно проводит все ещё плановые связанные операции и только затем закрывает задачу.
- Budget получает внутреннюю S2S-команду `budget_sync_linked_transaction_facts`, принимающую список ID, `fact_date`, WorkItem ID и idempotency key.
- `fact_date` берётся из локальной даты `actual_at`; при отсутствии `actual_at` PM-MCP устанавливает время перехода.
- После обычного подтверждения Budget уведомляет PM-MCP; задача закрывается только после подтверждения всего набора.
- При недоступности PM-MCP callback хранится в SQLite outbox и повторяется без нового daemon.
- Correlation/idempotency key предотвращает циклы и дубли.
- Переоткрытие не снимает факты; снятие факта не переоткрывает задачу; `Не актуально` ничего не проводит.

## Этап 3. Начало, конец и две зоны

- `planned_at` — начало timed-задачи, `planned_end_at` — конец; отдельная duration не хранится.
- `planned_date` — дата задачи без времени.
- Timed-инвариант: `planned_at < planned_end_at`; date-only нельзя совмещать с datetime-парой.
- Legacy `planned_at` без конца получает `planned_end_at = planned_at + 60 минут`; `actual_at` не меняется.
- Форма планирования разделяется на дату, начало, конец и режим «Без времени».
- День/неделя: часовая сетка сверху, date-only зона после `23:59`, отдельная зона «Без даты».
- Google `dateTime` события показываются в timed-зоне; Google all-day `date` — в date-only зоне; `end` остаётся исключающей границей.
- Drag сохраняет интервал, resize меняет конец, перенос между зонами меняет datetime-пару и `planned_date`.
- Использовать существующий React/Vite island без новой календарной библиотеки; месяц/квартал/год сохраняют компактный layout.

## Этап 4. Google Calendar write/sync

- OAuth scopes: `calendar.events` и `calendar.calendarlist.readonly`; после изменения нужна повторная авторизация.
- Default calendar выбирается по ID в настройках и отображается как «Степан Захватов»; имя не хардкодить.
- Автоматически экспортируются только operational WorkItem с `horizon IS NULL` и корректной парой start/end.
- Date-only задачи пока остаются локальными. Исторический backfill — только через preview, не автоматически.
- `calendar.db` хранит task/event mapping, calendar/event ID, `etag`, ревизии сторон, sync status и error.
- `extendedProperties.private` хранит стабильный WorkItem ID и версию связи.
- Двусторонне синхронизируются title/summary, start и end; остальные Google-поля не перезаписываются.
- Изменение одной стороны применяется; изменение обеих создаёт конфликт с выбором источника.
- Удаление Google event не удаляет задачу; mapping получает `missing_remote`.
- Завершить dormant `insert`, `patch`, `get`, `delete`; delete только по явному подтверждению.

## Проверки

- Multiple task-budget links, запрет одной операции у двух задач, tenant isolation.
- Атомарное подтверждение набора; последняя подтверждённая операция закрывает задачу.
- Idempotent callbacks/outbox retry; обратный откат отсутствует.
- Timed/date-only layout, drag/resize, interval across midnight и DST Europe/Zurich.
- Google start/end/title sync в обе стороны без циклов; конфликт не теряет данные.
- Для Python subsystems: `uv --cache-dir .uv-cache run ruff check .` и `uv --cache-dir .uv-cache run pytest`.
- Assistant-UI: `npm run build`, desktop/mobile `verify_frontend.py --page roadmap` и `--page budget`.
- Google write smoke только на отдельном тестовом событии.

## Pre-close retrospective

| Ось | Вердикт | Итог |
|---|---|---|
| `tech-stack-choices.md` | `follow-up-task` | #1198: brick #25 всё ещё описывает Calendar как read-only; обновление scopes и оценка общего SQLite outbox/S2S brick требуют отдельного подтверждения. |
| Design-system | `no-change` | Использованы существующие MD3 tokens и Material Web; нового общего примитива не появилось. |
| Skills | `no-change` | Существующие plan/task/migration/frontend verification workflows покрыли реализацию. |
| Hooks | `no-change` | Нового детерминированного guard-сценария не выявлено. |

## Результат реализации

- PM-MCP хранит tenant-scoped task-budget links, возвращает `budget_transaction_ids`, проводит единый completion flow и поддерживает task/event mapping, writable calendar selection и conflict resolution.
- Budget атомарно подтверждает набор связанных операций, хранит idempotency run и доставляет callback через SQLite outbox из существующего lifespan worker.
- Assistant-UI показывает бюджетные связи в карточке и журнале, разделяет timed/date-only/undated области, поддерживает start/end drag/resize и явное разрешение Google-конфликтов. Форма «Изменение операций» содержит колонку «Задача»: номер связанного WorkItem виден в каждой строке, ввод или очистка номера привязывает/отвязывает операцию через `/api/roadmap/tasks/{id}/budget`.
- ADR-0031 и subsystem docs фиксируют инварианты и границы синхронизации.

Проверки 12.07.2026:

- PM-MCP: Ruff; `264 passed`; security-review подтвердил обязательный `confirmed=true` для Calendar delete.
- Budget: Ruff; `91 passed`.
- Assistant-UI: Ruff; `168 passed`; `npm run build`; roadmap, budget и settings desktop/mobile verification.
- Assistant-UI, доработка «Задача» в форме операций (12.07.2026): Ruff; `169 passed`; budget desktop/mobile verification с проверкой колонки «Задача» и live-аннотации журнала на временной связи #1197 ↔ операция 5930 (после проверки снята). Ограничение: на ширине окна ~1440px и уже колонка комментария журнала ужимается до нуля, номер задачи в таблице виден на более широких окнах.
- service_policy: Ruff; `28 passed`; service_identity: Ruff; `11 passed`.
- `git diff --check` завершился с code 0; сообщения CRLF/LF информационные.

Не выполнен только live Google write smoke: новые scopes требуют повторной OAuth-авторизации, а проверка по плану допускается исключительно на отдельном тестовом событии. Исторический backfill не запускался.
