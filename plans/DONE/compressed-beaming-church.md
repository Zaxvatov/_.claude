# Черновик: типология запросов к агенту и последовательности работ

Статус: **реализован и архивирован в `DONE` 2026-07-16**.
Build-new-path, migration и remove-old-path стадии завершены.

PM-MCP: #1233 (planning/review-цикл черновика, project `plans`).
Implementation PM-MCP: #1245 «Внедрить маршрутизацию запросов R1–R6 и контроль
последовательности работ» (`D:\GitHub\_engineering_rules`).
Follow-up PM-MCP: #1243 «Свернуть локальные AGENTS.md до уникальных проектных
правил» (`D:\GitHub\_engineering_rules`, реализация завершена; дочерние
project-scoped work items #1246–#1259 созданы для всех фактически изменённых
owning scopes).

## Контекст

- Инцидент 2026-07-14: агент начал писать диагностический артефакт до
  `start_task`; граница «диагностика vs change request» существует одним
  предложением-исключением в K.2. Повтор более раннего feedback
  (AI-memory decision 2522 от 2026-07-12 о порядке работ перед правкой кода).
- Сейчас в правилах три несвязанные классификации: recall Type A/B/C
  (master Section C), «change request / explanation-only» (K.2), статусная
  модель работ (K.3 + Section D). Классы работ (планы, миграции, frontend,
  git) описаны отдельными процедурами без общей рамки.
- Версия v0.1 смешивала в одном плоском списке разные оси: намерение
  пользователя, масштаб, известность объекта, дефектность, домен, риск и форму
  взаимодействия. Один реальный запрос поэтому одновременно попадал в
  несколько Т-типов. В v0.2 эти признаки разделены на основной маршрут и
  независимые модификаторы.
- Аудит wiring выявил отдельный implementation-gap: текущий Codex
  `config.toml` ссылается на семь уже отсутствующих hook-скриптов, а hooks
  README всё ещё утверждает, что у Codex нет generic hooks. Кроме того,
  `context_gate` включает Plan Mode по одному слову «план» и не различает
  «создай/измени план» и «проанализируй план». До утверждения плана это только
  зафиксированные находки, не выполненные исправления.
- Редакции v0.2–v0.3 внесены targeted patch после обязательного
  `start_task #1233`. В текущем Codex surface вызов `EnterPlanMode` оказался
  недоступен; это capability-gap и дополнительный fixture для будущего
  plan-intent/runtime contract, а не новая норма обхода Plan Mode.
- Решение пользователя 2026-07-15: по возможности локальные `AGENTS.md`
  удаляются. Если файл необходим, он содержит только короткую ссылку на
  глобальный контракт и действительно уникальную project-specific дельту.
  Любая инструкция, обнаруженная минимум в двух проектах, поднимается в
  глобальный master; повторяемый workflow/guard может быть вынесен в общий
  skill/hook, на который ссылается глобальный master. Локальные копии после
  миграции удаляются.
- Цель: единая таксономия «тип запроса -> полная последовательность работ»
  (recall -> work item -> план -> выполнение -> верификация -> память ->
  закрытие). Пользователь предложил стартовые типы А–D; корпус реальных
  обращений даёт больше ситуаций.

## Фактура (снимок 2026-07-15)

| Источник | Прочитанный объём | Что считается пользовательским материалом |
|---|---|---|
| PM-MCP work items | 1490: 1242 project tasks по 19 проектам + 248 knowledge-записей | Live `list_work_items`, все возвращённые элементы |
| AI-memory | 1116 записей, id 8–2573: 686 change / 183 fact / 131 decision / 75 note / 41 task_context | Live `get_recent_memory`; авторы: codex 864, assistant-ui 153, claude 78, claude-code 3, chatgpt 8, ai-memory-admin 1, без автора 9 |
| Логи Claude | 63 JSONL, 90 262 729 байт; 246 primary non-meta user events, из них 245 с текстом + 1 media-only; 212 уникальных непустых текстов | Только `type=user` без `toolUseResult`, `isSidechain=true` и `isMeta=true`; не считаются `last-prompt`, queue и tool-result |
| Логи Codex | 509 JSONL, 1 809 017 434 байта; 2809 реальных user events, 28 пустых/context-only, 316 точных дублей, 2465 уникальных непустых запросов | Только авторитетный `event_msg.payload.type=user_message`; model-input `message role=user`, compaction и tool transcripts исключены |
| Codex rollout summaries | 61 сводка, 120 именованных подзадач | Кураторский слой фактически выполненных работ; не заменяет сырые сессии |

Снимок воспроизводим, но не является вечным «100%»: сессионные логи и live
хранилища продолжают расти. Полностью прочитаны файлы и записи, существовавшие
на момент снимка; это не означает стопроцентную точность семантической
классификации.

Методика исправлена относительно v0.1:

1. Файлы читаются потоково, без загрузки гигантского JSONL целиком.
2. Для Codex извлекается реальный запрос после оболочек `Files mentioned`,
   `Selected text` и `Response annotations`; служебный model input не
   считается повторным пользовательским сообщением.
3. Для Claude отделяются настоящие prompts от тысяч `tool_result`-строк.
4. Exact-dedup выполняется после извлечения запроса. Сырые тексты и приватные
   данные не сохраняются в отчёт или план.
5. Классификация multi-label: один запрос может иметь несколько доменов,
   рисков и форм взаимодействия. Регулярки дают ориентир, а не взаимоисключающие
   статистические классы; сценарии дополнительно сверены по случайным образцам
   и rollout summaries.

PM-MCP подтверждает именно многомерность: среди 1242 задач одновременно
встречаются frontend/UI (501), данные/память (491), feature/change (478),
task lifecycle (411), runtime/service (314), research/review (295),
verification/tests (282), планы/архитектура (277), rules/hooks/skills (233),
дефекты/регрессии/perf (225), migration/cleanup (193), security (135),
документы (127), automation/monitoring (117), git/release (94), личный контекст
(72) и calibration/training (19). Числа пересекаются и потому не суммируются.

Rollout summaries дают ту же картину: 30 из 61 затрагивают UI, 29 —
дефект/регрессию, 26 — исследование/решение, 22 — данные, 20 — новую
функциональность, 19 — migration/cleanup, 14 — rules/hooks/skills, 13 — live
verification gap, 11 — runtime/admin и 10 — пакетное закрытие задач.

Из сырых обращений дополнительно выделены: контекстные сообщения без новой
задачи; продолжение/approval/cancel/correction; вложения, URL, скриншоты и
консольный вывод; пользовательский admin-handoff; cross-agent handoff;
browser/OAuth; git/release и destructive actions; install/bootstrap;
data migration/import/backfill; monitoring/automation; интерактивная
калибровка; высокорисковые и чувствительные консультации; live-deployment
разрыв между правильным кодом и устаревшим runtime.

## Готовые классификаторы (чтобы не изобретать велосипед)

| Система | Типы | Что берём |
|---|---|---|
| ITIL/ITSM | Incident, Service Request, Standard Change, Normal Change, Problem | Словарь для nature-of-work: дефект, операция, change, root cause |
| Agile/Jira | Bug, Task, Story, Epic, Spike | Аннотации масштаба и характера, но не основной router |
| Conventional Commits | feat, fix, refactor, docs, chore, test, perf | Маппинг на этапе закрытия/коммита, не для маршрутизации |

Вывод: готовые классификаторы полезны как метки, но не решают вопрос
полномочий, памяти, риска и формы взаимодействия. Основной router должен быть
локальным: **один маршрут результата + независимые модификаторы**.

## Модель v0.3: один маршрут + независимые модификаторы

### Принцип

Основной маршрут отвечает только на вопрос: **какой результат пользователь
разрешил получить сейчас?** У запроса ровно один маршрут R1–R6. Известность
объекта, дефектность, домен, риск, необходимость памяти и форма взаимодействия
записываются отдельными модификаторами и не порождают новые верхнеуровневые
типы.

До выбора маршрута обрабатывается **контекстное событие K0**. К нему относятся:
«продолжай», «выполняй», выбор варианта, approval, correction, cancel, «не
надо», пользовательский отчёт о выполненной команде, лог/ошибка, вложение,
скриншот, URL и cross-agent handoff. K0 не создаёт самостоятельную задачу:
оно изменяет или продолжает активный маршрут. Если активного контекста нет и
цель из сообщения не выводится, задаётся один короткий вопрос.

### Порядок классификации

1. Найти активный контекст: текущая PM-MCP задача, утверждённый/черновой план,
   недавний handoff и явно указанный объект. K0 применить до новой
   классификации.
2. Установить границу полномочий:
   - «объясни / найди / почему / оцени / проверь / не меняй» — только ответ или
     read-only исследование;
   - «исправь / добавь / перенеси / выполняй / доделывай» — изменение;
   - push, публикация, отправка, удаление, внешняя запись, admin-действие —
     только если соответствующий side effect явно разрешён;
   - голый симптом внутри уже активной implementation-задачи наследует
     разрешение на исправление; вне активной задачи сначала выполняется
     read-only диагноз с явно названным допущением, а рискованный side effect
     не угадывается.
3. Выбрать один маршрут R1–R6 по ожидаемому результату.
4. Добавить модификаторы объекта, nature-of-work, масштаба, домена, риска,
   памяти и взаимодействия.
5. Разложить смешанный запрос на части. Ответная часть может остаться R1/R2,
   но каждая разрешённая мутация проходит R3/R4/R5 и собственный task preflight.
6. До действия загрузить обязательные skills и применить доступные hooks;
   отсутствие hook surface не отменяет ручной контракт AGENTS.md/skill.

### Ленивая route card и переходы

Route card — минимальное session/task-state, а не отдельный документ и не новый
обязательный tool call. Она материализуется только перед первым инструментом,
границей риска или мутацией; очевидный R1 chat-only завершается без карточки и
без загрузки routing skill.

Минимальные поля:

```text
route; authority; object; nature; scale; domain; risk; memory;
task_required; plan_required; verification_profile; confidence
```

Правила экономии и коррекции:

1. K0 наследует route card активной задачи/сессии; повторная классификация на
   каждое «продолжай» или уточнение не выполняется.
2. Для R3/R4/R5 карточка присоединяется к уже обязательному task state или
   `start_task`, а не записывается отдельной PM-MCP операцией.
3. Hook инжектирует только короткую route-specific директиву. Полная таблица
   типов и все skills в контекст каждого запроса не загружаются.
4. Маршрут можно повысить или сузить при новой evidence: R2 -> R3/R4/R5 до
   первой мутации, R3 -> R4 при найденной архитектурной развилке, R5 -> R6 при
   длительном наблюдении. Переход фиксируется только если меняет разрешённую
   последовательность или completion contract.
5. `confidence=low` требует вопроса только перед необратимым, внешним или
   высокорисковым действием. Для обратимого локального discovery агент может
   продолжить с явно названным допущением.
6. Если два названия приводят к одной минимальной безопасной
   последовательности, это один маршрут с модификаторами, а не два типа.

### R1. Ответ или преобразование только в чате

Критерий: результатом является ответ, перевод, краткое переписывание,
объяснение или консультация; файлы и внешнее состояние не меняются.

Последовательность:

1. Определить M0/M1/M2 и профильный skill, если домен специализированный.
2. При M1/M2 получить память; при актуальной/высокорисковой теме проверить
   live/официальные источники.
3. Выполнить преобразование или дать ответ в чате.
4. Не создавать PM-MCP задачу. Память не писать для одноразового ответа;
   применять end-of-chat checklist только если возникло устойчивое решение или
   подтверждённый факт.

Перевод в указанном файле — уже не R1, а R3 с доменом `docs`.

### R2. Read-only исследование, диагностика, ревью или проверка

Критерий: пользователь просит найти, сравнить, проанализировать, объяснить
причину, проверить состояние, провести audit/review или запустить проверку без
исправления исходников.

Последовательность:

1. Зафиксировать read-only границу; не создавать артефакт и не править объект.
2. Выполнить onboarding/recall по модификаторам. Источник истины:
   live state/API/БД -> текущий checkout и docs -> AI-memory -> интернет.
3. Использовать профильный review/audit/security/data skill.
4. Для запуска теста/verify подтвердить owning root и команду, задать
   десятиминутное окно и классифицировать результат по exit status.
5. Дать evidence-backed вывод, отделив факт, inference и неизвестное.
6. PM-MCP задача не нужна, кроме явно заказанного долговечного отчёта или
   большого аудита, чей skill требует task/artifact. Значимая находка может
   получить `note`; обычная диагностика и runtime error в память не пишутся.
7. Если пользователь разрешает исправление, перейти в R3/R4/R5 **до первой
   мутации**.

«Проанализируй план» — R2. «Доработай черновик плана» — R4.

### R3. Ограниченное локальное изменение

Критерий: один логический результат в коде, конфигурации, документе или
локальном артефакте; архитектурная развилка и отдельная программа работ не
нужны. Объект может быть известен или требовать локализации.

Последовательность:

1. `project-onboarding`: текущая директория, branch/status, root/local
   AGENTS.md, архитектурные документы и owning subsystem.
2. Определить M0/M1/M2; для продолжения проекта и дефектов выполнить recall.
3. Найти существующую задачу либо создать одну на логический результат и
   вызвать `start_task`. Если изменение уже явно разрешено, задача стартует до
   локализации объекта. Любой scratch/debug-файл тоже считается мутацией.
4. До первого code edit определить acceptance guard: тест, verifier, render,
   data invariant или обоснованное исключение.
5. Локализовать объект при O1, обновить описание задачи при существенном
   уточнении и внести минимальную правку.
6. Запустить целевой guard, затем обязательный gate затронутого owner root;
   для UI/runtime добавить browser/live verification.
7. Перечитать AGENTS.md, проверить scope/diff, docs и отсутствие случайных
   артефактов. Commit/push/branch/PR — только по отдельному разрешению.
8. Записать outcome как `change` (для code/config/data change), при
   необходимости `decision`/`fact`; закрыть задачу только после успешной
   верификации и обязательной ретроспективы.

### R4. Многостадийное или архитектурное изменение

Критерий: несколько стадий/подсистем/репозиториев, пользовательские развилки,
миграция, архитектурное решение или несколько зависимых задач.

Последовательность:

1. Создать/переиспользовать одну planning/review-задачу и работать с
   неутверждённым планом по `central-plan-workflow`. Plan Mode нужен для
   создания или изменения черновика, но не для read-only анализа упоминания
   плана; post-approval status updates допустимы вне Plan Mode.
2. Обязательный M2 recall, полный onboarding, релевантные ADR/docs/skills и
   `tech-stack-choices.md` до технологических решений.
3. Зафиксировать контекст, цель, ограничения, варианты, этапы, риски и
   acceptance criteria в центральном plan-файле; slug не переименовывать.
4. Получить явное approval плана. Затем создать implementation-задачи по
   owning project/subsystem, связать зависимости и записать ID внутри плана.
5. Исполнять стадии как R3/R5; progress, проверки, blockers и scope deltas
   вносить в план точечно.
6. Перед закрытием провести ретроспективу
   tech-stack / Design-system / skills / hooks. Shared изменения требуют
   отдельного подтверждения или follow-up.
7. Записать `change`/`decision`, закрыть задачи и архивировать план в `DONE\`
   только по полному lifecycle-чек-листу.

### R5. Операция с состоянием или внешний side effect

Критерий: меняются данные, PM-MCP/календарь/память, runtime/service,
зависимости/окружение, browser/OAuth/connector, git/release или внешняя система,
а не только исходный текст файла.

Последовательность:

1. Проверить live state и реальную schema/enum/tool contract; не угадывать
   значения и не доверять stale заметкам.
2. Определить риск. Для batch, destructive, irreversible, external,
   security-sensitive и git side effects показать scope/preview и получить
   нужное явное подтверждение.
3. Применить `dry-run`/preview, идемпотентность и before/after invariants, если
   система это поддерживает.
4. Task policy:
   - lifecycle-операция над уже существующей задачей использует эту задачу и
     не создаёт wrapper-task;
   - единичная полностью аудируемая tool-мутация может обойтись без wrapper,
     если локальные правила не требуют его;
   - batch/migration/destructive/cross-domain операция, persistent automation,
     reusable script либо изменение code/config всегда требует активной задачи.
5. Admin-операцию агент не эскалирует: даёт готовый PowerShell-фрагмент,
   пользователь выполняет его в elevated shell, после чего агент проверяет
   фактический результат.
6. Для browser/OAuth сохранять одну browser session; секреты не вводить и не
   логировать. Для git/release применять только явно разрешённые
   commit/push/branch/PR действия.
7. Проверить не только socket/process, а реальный request/data path. Если code
   acceptance выполнен, но live runtime неисправен, отделить runtime incident
   от исходной feature-задачи.
8. Зафиксировать durable outcome в задаче/памяти по значимости.

Если операция требует постоянного скрипта или меняет проектные файлы, её
локальная часть проходит R3; если стадий несколько — R4.

### R6. Ожидание, мониторинг или recurring automation

Критерий: результат зависит от времени или повторения: «подожди», «следи»,
«проверь позже», «раз в месяц», «продолжай до условия».

Последовательность:

1. Определить наблюдаемое состояние, cadence, success/stop condition, срок и
   способ уведомления.
2. Выбрать one-shot wait/monitor либо recurring automation. Перед созданием
   recurring job найти существующую automation и не создавать дубль.
3. Выполнять только разрешённое наблюдение. Автоматическое исправление,
   публикация или иная мутация требует отдельного R3/R5 authorization.
4. Long-running работа не расширяет scope: «до победного» означает
   настойчивость внутри уже разрешённой цели, а не право на новые side effects.
5. Сообщить финальный state; память использовать только для устойчивого
   решения или повторяемого operational contract.

### Модификаторы

| Ось | Значения | Влияние на маршрут |
|---|---|---|
| Полномочия A | A0 chat; A1 read-only; A2 local change; A3 external side effect; A? ambiguous | Определяет допустимую границу; A? не разрешает рискованную мутацию |
| Объект O | O0 указан; O1 локализовать | O1 добавляет grep/browser/runtime inspection, но не новый тип |
| Nature N | feature; defect; regression; performance; refactor; migration; cleanup | Выбирает guard и depth root-cause, но не основной маршрут |
| Масштаб S | S0 atomic; S1 multi-stage; S2 batch/cross-repo | S1 обычно переводит R3/R5 в R4; S2 требует декомпозиции и scope control |
| Домен D | code/backend; frontend/UI; docs/content/media; data; meta rules/skills/hooks; runtime/browser/integration; git/release; security; personal/high-stakes | Подключает профильные skills, CWD и verifier |
| Риск K | K0 read-only; K1 reversible local; K2 external; K3 destructive/irreversible; K4 admin; K5 secret/sensitive | Определяет preview, approval, handoff и security review |
| Память M | M0 skip; M1 light recall; M2 mandatory recall | Независима от маршрута; заменяет попытку привязать старые Type A/B/C к типам работ |
| Взаимодействие I | one-shot; calibration loop; attachment/log input; user-executed handoff; cross-agent handoff; blocked/resume | Определяет размер задачи и способ продолжения, не отдельный верхний тип |

Правила M0/M1/M2:

- M0 — самодостаточный одноразовый запрос без проектного контекста.
- M1 — прошлое решение может повлиять на рекомендацию, стиль или выбор.
- M2 — continuation, PM-MCP, rules/hooks/skills, архитектура, миграция,
  AI-memory, план или повторный дефект. Сначала recent project/portfolio memory,
  затем keyword search при неполном контексте.
- Репозиторий, live state и актуальные документы всегда сильнее памяти.

## Каталог наблюдаемых сценариев

Каталог нужен для полноты и тестовых fixtures, но не является списком
взаимоисключающих типов.

| Сценарий из корпуса | Маршрут и модификаторы | Обязательная дельта процесса |
|---|---|---|
| Перевод/переписывание прямо в чате | R1, обычно M0 | Без задачи и памяти |
| Ответ, консультация, рекомендация | R1/R2; D personal/high-stakes при необходимости | Актуальные источники и safety для high-stakes |
| Поиск в репо, памяти или интернете | R2, O1, M0–M2 | Источники и граница fact/inference |
| Code/security/automation review | R2 + профильный D | Только находки; fixes — отдельный R3/R4 |
| Read-only анализ существующего плана | R2, M2 | Не включать Plan Mode по одному слову «план» |
| Правка известного объекта (исходный A) | R3, O0, S0 | Task -> guard -> edit -> verify -> memory |
| Визуальный эффект, объект неизвестен (исходный B) | R3, O1 | Task до локализации, если change уже разрешён |
| Дефект, регрессия, performance | R2 при диагнозе; R3 при исправлении; N defect/regression/performance | Root cause, regression/baseline guard, before/after |
| UI/design/responsive/accessibility | R3/R4, D frontend | Design skill + authoritative browser verification |
| Документ, PDF, презентация, таблица, изображение | R1 если chat-only; иначе R3/R4, D docs/media | Профильный skill + render/visual verification |
| Единичная data/tool операция | R5, D data | Schema/enum, preview при риске, audit outcome |
| Batch import/backfill/merge/migration | R4/R5, S2, K2–K3 | Task, snapshot/dry-run, invariants, rollback |
| PM-MCP status/close/reopen | R5, K1 | Использовать сам work item, не wrapper-task |
| Runtime/service/install/bootstrap | R5; R3/R4 если меняется code/config | Проверять real request path; admin handoff |
| Browser/OAuth/connector | R5, K2/K5 | Одна session, browser discipline, no secrets |
| Admin-команды выполняет пользователь | R5, I user-executed, K4 | Ready-to-paste PowerShell -> wait -> verify |
| Git/commit/push/PR/release | R5, D git/release, K2/K3 | Каждое действие только по явному запросу |
| Cleanup/legacy removal/destructive migration | R4/R5, N cleanup/migration, K3 | Reference sweep, новый путь сначала, old-path-absence guard |
| Monitor/wait/recurring run | R6 | Cadence, stop condition, dedup automation |
| Интерактивная калибровка/golden cases | R3/R4, I calibration | Одна задача на цикл, project-owned fixtures, full-set convergence |
| «Продолжай», approval, вариант, correction, «не надо» | K0 -> наследуемый маршрут | Не создавать дубль задачи; cancel немедленно останавливает |
| Вложение, screenshot, URL, лог или stack trace без новой цели | K0, I attachment/log | Считать входом текущего маршрута |
| Cross-agent handoff | K0; review -> R2, continue -> исходный R3/R4/R5 | Проверить live diff/task, не доверять только пересказу |
| Resume после user/admin action | K0, I blocked/resume | Перепроверить фактическое состояние и продолжить task |
| Batch-аудит backlog с закрытием готового | R4/R5, S2 | Live truth по каждой задаче; fix/close/obsolete по доказательствам |
| Инцидент поведения агента | R2, D meta; confirmed prevention -> R3/R4 | Finding в note; rules/skill/hook только после approval |
| Sensitive/security/high-stakes запрос | Любой R + K5/D security | Минимизация данных, official sources, security review |
| Code готов, live runtime не принят | Исходный R3/R4 + отдельный R5 incident | Не маскировать service/runtime fault внутри UI bug |
| «До победного» / длительная оптимизация | Исходный маршрут + I continuation | Настойчивость без расширения authority; промежуточные проверки |
| Создание persistent automation | R6 + R5, K2 | Task при проектной автоматизации, explicit schedule/stop/update path |

## Сводная матрица последовательностей

| Маршрут | Recall | Work item | Plan Mode | Основная проверка | Память |
|---|---|---|---|---|---|
| R1 chat-result | M0–M2 | нет | нет | корректность ответа/источников | только устойчивый fact/decision |
| R2 read-only | M0–M2 | обычно нет; нужен для durable audit artifact по skill | нет | evidence, exit status, source attribution | note только для значимой находки |
| R3 bounded change | M0–M2 по контексту | активен до локализации/первой мутации | нет | guard + target + owner gate + live/UI при необходимости | change; decision/fact по результату |
| R4 multi-stage change | M2 | planning/review + implementation tasks | только создание/изменение черновика | по стадиям + интеграция + ретроспектива | change/decision/task_context |
| R5 state/external operation | M0–M2 | по task policy и риску | нет, если не часть R4 | preview/dry-run + before/after + live path | значимый change/note |
| R6 monitor/automation | M0–M2 | для persistent project automation | нет, если не часть R4 | stop/success condition + dedup + observed state | только durable operational contract |
| K0 context event | наследует | новый не создаётся | наследует | перепроверка контекста при resume | отдельно не пишется |

## Контроль правильности маршрута

Правильность определяется не только совпадением метки R1–R6, а двумя
условиями: применена минимальная безопасная последовательность и не выполнены
лишние тяжёлые шаги. Ошибка имени маршрута не должна разрешать опасное действие,
а безопасный простой запрос не должен получать onboarding/task/plan/skill без
необходимости.

Контроль состоит из трёх уровней:

1. **Semantic candidate.** `context_gate`/agent выбирает route card и
   `confidence=high|medium|low`. Keyword/regex — только признаки; слово «план»
   не равно plan mutation, а слово «проверь» не разрешает fix.
2. **Action invariants.** Hooks и server/tool contracts проверяют наблюдаемые
   действия независимо от метки: edit требует active task; M2 — recall proof;
   plan mutation — Plan Mode; test/build/verify — owning CWD и десятиминутный
   cap; external/destructive/admin/git — соответствующую authority; закрытие —
   guard evidence и memory outcome.
3. **Outcome/pre-close.** До `complete_task` сверяются acceptance, owner gate,
   live result, отсутствие ослабленного test contract, корректность task scope,
   недублирование work item и выполненная ретроспектива.

### Golden set и shadow rollout

- Обезличенный golden set хранит для каждого scenario summary ожидаемые route,
  modifiers, обязательные и запрещённые действия. Новый router проходит его как
  воспроизводимый contract test.
- Сначала включается shadow mode: candidate и фактическая последовательность
  записываются, но новый semantic router не блокирует работу. Затем включаются
  предупреждения. Hard deny применяется только к детерминированным инвариантам.
- Claude Desktop и Codex Desktop проверяются раздельными payload fixtures и
  smoke cases; наличие hook-секций в config само по себе не является proof.

Метрики качества:

- unsafe/missing-required-step rate — целевое значение `0`;
- false-block rate и число ручных обходов;
- user-correction rate и route change после первого tool;
- Plan Mode false-positive rate;
- duplicate-task и missing task/recall/verification/memory rates;
- доля `confidence=low` и неизвестных комбинаций;
- лишние onboarding/memory/skill/tool calls, токены и wall time по маршруту.

После shadow периода enforcement расширяется только если golden set проходит,
unsafe rate равен нулю, а false-block/token overhead не хуже утверждённого
baseline. Неизвестный сценарий остаётся безопасным advisory case и не получает
новую authority автоматически.

## Автоматическое расширение базы сценариев

Повторный полный анализ всех чатов не является регулярным процессом. Текущий
корпус — bootstrap. Далее hooks пишут privacy-safe operational telemetry в
`D:\GitHub\_engineering_rules\.state\hooks\request-routing.sqlite3` на
существующем brick SQLite+WAL. `.state` остаётся untracked; AI-memory не хранит
сырые логи маршрутизации.

Записываются только: session/task id, prompt hash, route/modifiers/confidence,
обезличенный scenario signature для аномалии, первый tool class, guard result,
route transition, user correction и outcome. Полный prompt, secrets, вложения и
transcript не сохраняются.

Кандидат на расширение появляется при одном из событий:

- user correction или подтверждённый false block;
- `confidence=low`/ничья маршрутов;
- route transition сразу после первого действия;
- новая комбинация route + modifier + tool;
- пропуск или нарушение ожидаемой последовательности;
- unsafe side effect, который не остановил существующий guard.

Порог promotion:

- unsafe/unauthorized side effect — немедленный review candidate;
- одна и та же пользовательская коррекция два раза — proposal;
- один и тот же low-confidence/unknown сценарий три раза за 30 дней — proposal;
- единичная новая формулировка знакомого workflow — fixture, не новый тип.

| Повторяемая находка | Нормативное действие |
|---|---|
| Новая формулировка существующего workflow | Добавить fixture |
| Новый признак при прежней последовательности | Добавить модификатор |
| Повторяемое недетерминированное рассуждение | Уточнить общий skill |
| Повторяемое наблюдаемое нарушение | Добавить hook/server guard |
| Новый outcome и иная последовательность/completion contract | Предложить новый верхний маршрут |

Еженедельная automation расширяет `agent-session-report`: агрегирует метрики,
группирует только аномалии и при достижении порога создаёт PM-MCP proposal в
`На согласовании`, не меняя правила самостоятельно. Нормативное promotion в
fixture/skill/hook/master выполняется только после user approval. Полный raw-log
reanalysis нужен после крупной смены модели/router, всплеска unknown rate или
для редкого контрольного аудита, но не для каждого нового сценария.

## Контракт задачи, guard и памяти

1. **Task до мутации.** Любая правка project source/config/docs, создание
   артефакта или scratch/debug-файла выполняется только после
   `create/reuse task -> start_task`. Формулировка «при первой правке создать
   задачу» реализуется строже: к моменту первого edit задача уже активна.
2. **Локализация не отменяет task.** Если пользователь уже разрешил change,
   O1-локализация выполняется внутри активной задачи. Если запрос только
   диагностический, R2 остаётся без задачи до явного перехода к исправлению.
3. **Гранулярность.** Одна задача — один логический результат; одна задача на
   calibration cycle, а не на каждую оценку. Cross-subsystem work получает
   отдельный work item для каждого project_path, как требуют локальные правила.
4. **Guard выбирается до первого code edit.** Тест может быть написан до фикса
   или одновременно с ним; для дефекта предпочтителен regression test, который
   воспроизводит исходное нарушение. Механический тест без связи с acceptance
   не считается guard.
5. **Исключение только обоснованное.** Если автоматизировать проверку
   невозможно, в задаче и финале фиксируются причина и альтернативный verifier.
   Нельзя молча считать работу завершённой.
6. **Существующие контракты не ослабляются.** Удаление/изменение существующего
   test/verifier/CI contract требует отдельного review и явного подтверждения;
   новый guard не заменяет старый без migration discipline.
7. **Outcome в AI-memory.** Завершённый code/config/data change получает
   `change` с root cause/границами; архитектурный или workflow-выбор —
   `decision`; стабильный environment fact — `fact`. Routine test run,
   промежуточная отладка и длинные логи не записываются.
8. **Закрытие.** Owner gate, live acceptance, memory outcome и pre-close
   ретроспектива выполняются до `complete_task`. Commit/push не являются
   условием закрытия и делаются только по отдельному запросу.

### Guard по доменам

| Домен/Nature | Минимальный guard |
|---|---|
| Bug/regression | воспроизведение причины + regression test + проверка, что guard ловил старое поведение, когда это практически возможно |
| Feature/API/backend | acceptance/contract test + целевой pytest + обязательный owner gate |
| Frontend/UI | unit/integration guard + authoritative isolated browser scenario; screenshot не единственный критерий |
| Performance | baseline до/после, одинаковые данные/окружение, бюджет времени/ресурсов |
| Hooks/scripts/config | stdlib/unit/contract test входного event payload и allow/deny/reminder результата |
| Docs/Word/PDF/slides/sheets | lint/link/вычитка и обязательный render/visual check по профильному skill; фиктивный unit test не нужен |
| Data mutation/import | preview/dry-run, backup/rollback по риску, counts и инварианты before/after |
| Runtime/service | health + реальный request/tool path + проверка правильного process/session identity; socket readiness недостаточно |
| Migration/cleanup | новый путь работает, все consumers/data перенесены, references и old path отсутствуют |
| Calibration/training | весь накопленный project-owned golden set, а не только последний пример |

## Рабочая директория и запуск команд

CWD определяется **владельцем проверяемого контракта**, а не маршрутом R1–R6.

| Работа | Откуда запускать | Правило/пример |
|---|---|---|
| Git status/diff/rg по репозиторию | root из `git rev-parse --show-toplevel` | Сначала подтвердить branch/status; cross-repo — отдельный проход в каждом root |
| Python subsystem | ближайший owning root по git/project metadata и `pyproject.toml`; local AGENTS.md учитывается только если существует | `uv --cache-dir .uv-cache run pytest`; `uv --cache-dir .uv-cache run ruff check .` |
| AI-Assistant subsystem | `D:\GitHub\AI-Assistant\<subsystem>` | Не запускать subsystem suite из monorepo root и не подменять system Python |
| Assistant-UI browser verification | `D:\GitHub\AI-Assistant\assistant-ui` | Авторитетный `scripts\verify_frontend.py` через subsystem `uv` и isolated runtime |
| Shared hooks lint | `D:\GitHub\_engineering_rules` | `uv --cache-dir .uv-cache tool run ruff format --check hooks` и `uv --cache-dir .uv-cache tool run ruff check hooks --select E501` |
| Shared hooks tests | `D:\GitHub\_engineering_rules\hooks` | `python -m unittest discover -s tests` — намеренное stdlib-исключение из pytest brick |
| Shared skills/rules | `D:\GitHub\_engineering_rules` | Использовать локальный validator/skill contract; не копировать skills в project |
| Central plan | `C:\Users\Zaxva\.claude\plans` | Markdown structure, targeted diff/status; project test suite не запускается |
| Word/PDF/slides/sheets | каталог исходного артефакта либо workspace skill runtime | Запуск renderer/validator из соответствующего skill; визуальная проверка результата |
| PM-MCP/AI-memory/domain connector | tool/MCP interface | Shell CWD не подменяет tool schema; direct DB — только документированный fallback |
| Cross-subsystem/cross-repo | каждый owning root отдельно | Targeted checks по owner, затем интеграционный gate в root интегратора |

Перед каждой командой агент подтверждает существование указанного wrapper/файла
в текущем checkout. Пример пути не заменяет owning contract; локальный
`AGENTS.md` участвует только как действительно project-specific delta.

## Единое десятиминутное окно

1. Первый запуск любого test, lint, build, full verifier, migration check,
   import/recalculation или потенциально долгого анализа получает
   `timeout_ms=600000`. Если команда закончилась раньше — результат принимается
   сразу; резерв не заставляет ждать десять минут.
2. Сначала запускается самый узкий релевантный guard, затем обязательный полный
   gate affected owner. Десятиминутное окно задаётся каждому осмысленному
   запуску, а не всему чату суммарно.
3. На 10:00 агент прерывает только запущенный им PID/process tree, сохраняет
   доступный stdout/stderr и диагностирует: был ли прогресс, где зависло,
   ожидались ли сеть/lock/service, не запущен ли неверный runtime.
4. Автоматически повышать timeout выше десяти минут и просто повторять тот же
   command запрещено. Следующий шаг — сузить тест, разбить batch на chunks,
   устранить deadlock/startup issue или отдельно согласовать действительно
   длительный operational workflow.
5. Для заведомо долгих вычислений работа chunked/monitorable с checkpoints;
   один непрозрачный процесс дольше десяти минут не считается приемлемым.
6. Быстрые read-only команды discovery (`rg`, `git status`, чтение файлов)
   могут иметь меньший timeout. Политика 10 минут — максимум для работ, а не
   обязательная задержка для каждой shell-команды.
7. Для команды, которую выполняет пользователь под admin, инструкция содержит
   ожидаемое время, признак успеха и безопасный способ прерывания; после ответа
   пользователя агент проверяет live state.

## Обязательные skills

Skills выбираются по смыслу маршрута и модификаторов; список ниже означает
«обязателен, когда trigger совпал», а не «загрузить всё всегда».

| Триггер | Обязательные skills |
|---|---|
| Ambiguous/mixed request, correction маршрута или обслуживание taxonomy | новый `request-routing`; не загружать для очевидного R1 и не вызывать отдельным tool call для каждой реплики |
| Любое repo change R3/R4 | `project-onboarding`, `pm-mcp-task-flow`, перед завершением `source-command-verify` и `source-command-cleanup-check`, затем `ai-memory-capture` |
| Continuation, архитектура, plan, rules/hooks/skills, migration | `ai-memory-recall` |
| Создание/изменение центрального черновика | `central-plan-workflow` |
| UI/design/layout/UX/accessibility | `impeccable`, после изменения `frontend-verification` |
| Browser/OAuth/connector/local web UI | `browser-session-discipline` + доступный browser-control skill |
| Code/PR/diff review | `code-review` |
| Security/auth/secrets/data access/dependency risk | `security-review` |
| Migration/refactor с заменой старого пути | `migration-discipline` |
| Создание/правка hook | `hook-authoring` |
| Создание/изменение skill | `skill-creator`; для центрального каталога также `skills-library-maintenance` |
| Repo automation audit | `repo-automation-audit` |
| Аналитика Claude/Codex sessions и routing-quality | `agent-session-report` с privacy contract; агрегировать telemetry/аномалии, не публиковать raw prompts |
| Git publication/release/PR | `repo-release-pr`; GitHub-specific skill при соответствующем запросе |
| DOCX/PDF/slides/spreadsheets/image artifact | соответствующий `doc`/`pdf`/`presentations`/`spreadsheets`/`imagegen` skill |
| Spreadsheet -> DB | `spreadsheet-to-db-migration` |
| Design-system integration | `design-system-integration` |
| Финал meta/planning или durable outcome | `ai-memory-capture` |

R1 не требует project skills для простого chat transform. R2 загружает только
recall и профильный review/research skill. R5/R6 используют task-flow skill,
только когда операция по task policy действительно должна иметь work item.

## Обязательные hooks и граница их ответственности

Skills принимают семантические решения; hooks реагируют только на наблюдаемое
событие. Не создаётся отдельный hook на каждый сценарий каталога.

| Событие | Канонический hook | Обязанность |
|---|---|---|
| SessionStart | `session_start.py` | onboarding context |
| UserPromptSubmit | `context_gate.py` | короткий route/confidence/memory/plan candidate без загрузки всей taxonomy и без выполнения semantic action |
| PreToolUse shell | `pretool_bash.py` | consolidated admin, identity, Python env, shell syntax, active-task и timeout guards |
| PreToolUse edit | `pretool_edit.py` | active task, recall preflight, plan/Markdown/migration path guards |
| PostToolUse recall | `posttool_recall.py` | session-scoped proof M1/M2 recall |
| PostToolUse task start/close | `task_state.py` | active-task state и test-contract baseline lifecycle |
| PreToolUse task close | `pretool_task_close.py` | защита существующих tests/verifiers и будущая verification-evidence проверка |
| PreToolUse memory write | `memory_write_guard.py` | payload, metadata, sensitivity/provenance contract |
| PostToolUse edit/shell | `edit_reminders.py` / `shell_reminders.py` | frontend, plan, migration и destructive reminders |
| PostToolUse close | `close_reminders.py` | 4-axis retrospective и plan archive reminder |

`hooks\lib\routing.py` и `hooks\lib\routing_telemetry.py` являются общими
библиотеками существующих dispatchers, а не новыми hook entrypoints на каждый
тип. Телеметрия fail-open: сбой записи метрик не блокирует работу; safety guard
fail-closed остаётся только там, где уже есть однозначный observable invariant.

### Найденный drift и обязательные follow-up после approval

1. `C:\Users\Zaxva\.codex\config.toml` ссылается на отсутствующие
   `remind_complete.py`, `admin_guard.py`, `python_env_guard.py`,
   `require_active_task.py`, `plan_lifecycle_guard.py`,
   `frontend_verification_reminder.py` и `migration_doc_guard.py`.
   Wiring должен перейти на существующие consolidated hooks из таблицы, без
   восстановления семи устаревших wrapper-скриптов.
2. `hooks\README.md` должен описывать фактический Codex hook surface текущей
   desktop-версии либо честно фиксировать проверенное ограничение; README и
   config не могут противоречить друг другу.
3. `context_gate` должен различать plan-intent. Fixtures обязательно включают:
   «создай план», «измени черновик», «проанализируй план», «проверь план, не
   меняй», post-approval status update и обычное слово «план» в другом домене.
4. `pretool_bash.py` получает timeout-проверку только если event payload
   реально содержит tool timeout. Если runtime не экспонирует поле, остаётся
   deterministic reminder/manual skill rule, а не мнимая enforcement.
5. `pretool_task_close.py` должен проверять наличие test/guard evidence для
   changed runtime code либо явное обоснованное исключение; docs-only и
   tool-only operations не должны получать ложный deny.
6. Hook fixtures и smoke-проверка выполняются отдельно для фактических payload
   Claude Desktop и Codex Desktop. Trusted hashes обновляются только после
   проверки окончательного wiring.

## Owning scope и точный состав implementation

Глобальный `AGENTS.md` хранит только короткий обязательный контракт и ссылки на
общие workflows. Полная матрица не дублируется в стартовом контексте каждого
агента.

### Обязательный центральный scope

1. Global master:
   `G:\Мой диск\Бэкапы, инструкции, настройки, синхронизации\Codex\AGENTS.md`.
   Этот plan-файл остаётся planning/execution record, а не runtime policy.
2. Новый central skill:
   `D:\GitHub\_engineering_rules\skills\request-routing\SKILL.md`.
3. Targeted updates skills:
   `ai-memory-recall`, `central-plan-workflow`, `pm-mcp-task-flow`,
   `hook-authoring`, `source-command-verify`,
   `source-command-cleanup-check`, `agent-session-report`.
4. Existing hooks:
   `hooks\README.md`, `context_gate.py`, `lib\context.py`, `lib\gating.py`,
   `lib\state.py`, `pretool_bash.py`, `pretool_edit.py`,
   `pretool_task_close.py`, `task_state.py`.
5. New hook libraries/tests:
   `lib\routing.py`, `lib\routing_telemetry.py`,
   `tests\test_routing.py`, `tests\fixtures\request_routing_cases.json`, плюс
   targeted updates существующих context/gating/dispatcher/task-state/test-
   contract tests.
6. Runtime wiring:
   canonical target `G:\...\Codex\config.toml` через
   `C:\Users\Zaxva\.codex\config.toml`. Claude `settings.json` сначала только
   проверяется; правится лишь при доказанном wiring gap.
7. Operational state:
   `.state\hooks\request-routing.sqlite3`; файл не коммитится и не является
   AI-memory.

Recommended v1 не меняет production-код AI-Assistant, PM-MCP schema,
Design-system или `tech-stack-choices.md`. Если capability smoke докажет, что
Codex generic hooks не исполняются, server-side enforcement становится отдельным
scope/task; его файлы определяются после onboarding PM-MCP, а не угадываются в
этом плане.

### Миграция локальных AGENTS.md — PM-MCP #1243

Целевое состояние:

1. Если в локальном `AGENTS.md` нет уникальной project-specific дельты, файл
   удаляется после reference sweep и проверки фактической загрузки глобального
   master в Claude/Codex.
2. Если локальный файл нужен, в нём остаются только короткая ссылка на
   глобальный контракт и уникальные нюансы проекта. Базовые последовательности,
   статусы, testing/memory/task/plan правила локально не повторяются.
3. Если смысловое правило встречается минимум в двух проектах, его нормативная
   формулировка переносится в global master. Подробный повторяемый workflow или
   deterministic guard живёт в центральном skill/hook, на который ссылается
   master; локальные копии удаляются.
4. Перед удалением проверяются references, local `CLAUDE.md`/pointer files,
   loader behavior и отсутствие потери уникальных требований. Документация после
   миграции описывает только текущий путь.
5. #1243 владеет inventory, global promotion и общим acceptance. Для каждого
   репозитория, где реально меняются файлы, создаётся отдельный project-scoped
   work item и dependency, как требует cross-repo task contract.

Текущие кандидаты audit sweep:

- `D:\GitHub\_project_template\AGENTS.md`;
- `D:\GitHub\GmbH\AGENTS.md`;
- `D:\GitHub\Best_photo_ai\AGENTS.md`;
- `D:\GitHub\ANKI\AGENTS.md`;
- `D:\GitHub\Hauswirtschaftiche_Pflegeleistungen\AGENTS.md`;
- `D:\GitHub\Verua_automation\AGENTS.md`;
- `D:\GitHub\Design-system\AGENTS.md`;
- `D:\GitHub\Verua_automation\Design-system\AGENTS.md`;
- `D:\GitHub\Stalking_offline\AGENTS.md`;
- `D:\GitHub\AI-Assistant\AGENTS.md`.

Этот список — вход инвентаризации, а не приказ механически оставить десять
коротких файлов: предпочтение отдаётся полному удалению файла, если уникальной
дельты нет и наследование глобального master доказано smoke-проверкой.

## Соответствие действующим правилам

| Действующий слой | Модель v0.2 |
|---|---|
| Recall Type A/B/C | Независимая ось M0/M1/M2; она не определяет вид работы |
| K.2 change request / explanation-only | Граница полномочий A + маршруты R1/R2 против R3/R4/R5 |
| K.3 / Section D statuses | Не меняются; работают внутри task policy |
| Plan workflow | R4 + D plan/meta; Plan Mode только по mutation-intent |
| Local AGENTS ownership | По возможности файл отсутствует; иначе ссылка на global master + только уникальная project delta; повтор в 2+ проектах получает нормативную формулировку в global master, а подробный workflow/guard — в referenced central skill/hook |
| Frontend/migration/git/docs/security | Доменные модификаторы D и обязательные skills/guards |
| Продолжение/approval/log/handoff | K0, наследующий активный маршрут |
| ITIL/Agile/Conventional Commits | Дополнительные nature/scale/close labels, не router |

Migration discipline: сначала добавить и проверить новый router, затем перевести
на него все правила/skills/hooks и только после этого удалить старые Type A/B/C
и T1–T11 из нормативной документации. В итоговом состоянии legacy-модель не
остаётся как параллельный supported path.

## Решения, принятые в v0.3

1. Верхнеуровневые коды — R1–R6; T1–T11 остаются только историческим
   источником сценариев и не мигрируют в правила.
2. Порог work item для data/ops определён task policy, а не числом строк:
   wrapper не нужен для lifecycle существующей задачи и безопасной атомарной
   audited tool-операции; batch/migration/destructive/persistent/script/code
   требует task.
3. Инцидент поведения агента — R2 + D meta; prevention change начинается
   только после approval как R3/R4.
4. `compressed-beaming-church.md` не переименовывается.
5. Golden/calibration cases хранятся в owning project как воспроизводимые
   fixtures/test data; память хранит только финальное правило и ссылку.
6. Semantic «100% coverage» не заявляется. Перед rollout используется
   анонимизированная стратифицированная fixture-выборка не менее 200 сценариев
   плюс обязательные edge cases из каталога.
7. Две-три недели ожидания не являются блокером approval. После реализации
   применяется shadow-mode/fixture validation без изменения authority на
   неизвестных случаях.
8. Новый tech-stack brick не нужен: используются действующие uv/pytest,
   stdlib-hooks, SQLite+WAL для локальной telemetry и isolated frontend
   verification contracts.
9. Route card ленивая и session/task-scoped: очевидный R1 не платит токенами и
   tool calls за классификатор; K0 наследует уже выбранный маршрут.
10. Основной критерий корректности — соблюдение минимальной безопасной
    последовательности при отсутствии лишних тяжёлых шагов, а не только точное
    имя R-класса.
11. Расширение базы semi-automatic: telemetry, clustering и proposal создаются
    автоматически; нормативный fixture/modifier/skill/hook/route меняется после
    user approval.
12. Local `AGENTS.md` по возможности удаляется. Оставшийся файл содержит только
    ссылку и уникальную project delta; повторяемое в 2+ проектах правило
    поднимается в global master/shared workflow. Миграцией владеет #1243.

## Риски и меры

| Риск | Мера |
|---|---|
| Regex/router неверно понимает короткое сообщение | K0 наследует контекст; semantic решение остаётся у агента; рискованный A3 не угадывается |
| Плоская типология снова разрастается | Новый сценарий сначала выражается модификатором; новый R допускается только если меняется outcome/workflow |
| Task explosion | Правила logical result, calibration cycle и no-wrapper для lifecycle/atomic audited operations |
| Hook drift Claude/Codex | Один canonical source, payload fixtures для обоих runtimes, wiring+README sweep |
| Hook ложноположительно блокирует работу | Hooks проверяют только observable invariants; semantic ambiguity остаётся skill/manual |
| Десятиминутный cap режет легитимный batch | Chunking, checkpoints и отдельный monitor; timeout выше 10 минут не повышается молча |
| Memory засоряется отладкой или private logs | End-of-chat taxonomy, короткий durable summary, sensitivity/provenance guard, без transcript |
| Mixed request скрывает side effect | Декомпозиция; каждая mutation-часть получает свой authorization/task preflight |
| Live runtime расходится с code/tests | Real request-path verification и отдельный runtime incident |
| Пользовательские/сессионные данные попадают в fixtures | Только обезличенные scenario summaries; raw corpus не коммитится и не сохраняется в plan |
| Router сам увеличивает расход токенов | Lazy route card, session/task cache, однострочная hook-директива, routing skill только по trigger |
| Автоматическое расширение раздувает taxonomy | Автоматически только candidate/proposal; promotion thresholds и user approval; новая формулировка сначала fixture |
| Удаление local AGENTS ломает project instructions | Inventory уникальных правил, reference/CLAUDE pointer sweep и cross-agent loader smoke до удаления |
| Локальное правило ошибочно признано глобальным | Semantic dedup по смыслу, порог 2+ проектов, review владельцев и project-scoped child tasks #1243 |

## Критерии приёмки плана и будущей реализации

Пользователь утвердил эти критерии командой «выполняй» 2026-07-16:

- исходные A/B/C/D однозначно покрыты R3+O0, R3+O1, R2 и R4;
- все найденные дополнительные ситуации выражаются R1–R6 + модификаторами/K0
  без нового верхнего типа;
- read-only diagnosis и authorized fix имеют явную границу;
- task существует до первой мутации, guard определяется до code edit, durable
  outcome записывается до закрытия;
- CWD выбирается по owning contract, а test/build/verify получают
  `timeout_ms=600000` с диагностикой на 10-й минуте;
- обязательные skills перечислены по trigger, hooks — по event;
- текущий Codex hook drift и plan-intent false positive включены в
  implementation scope, а не объявлены уже исправленными;
- fixture validation покрывает минимум 200 обезличенных сценариев, все K0
  cases, admin/browser/git/destructive и plan-intent edges;
- lazy route card не создаёт отдельного tool call, не загружает routing skill для
  очевидного R1 и корректно наследуется/переходит на K0/R2->R3/R3->R4 cases;
- shadow report содержит unsafe, false-block, user-correction, route-change,
  Plan Mode false-positive, duplicate-task и token/tool/time metrics по route;
- weekly anomaly audit собирает privacy-safe telemetry, применяет утверждённые
  thresholds и создаёт только proposal, не меняя normative taxonomy;
- #1243 содержит правило удаления local AGENTS, promotion при повторе в 2+
  проектах и обязательные project-scoped child tasks; loader/reference smoke не
  допускает broken `CLAUDE.md` или потерю уникальной project delta;
- hook/unit tests проходят, Claude/Codex wiring проверен фактическим smoke,
  old normative Type A/B/C path удалён после миграции;
- shared rules/skills/hooks меняются только после явного approval этого плана.

## Итоги реализации

- #1245 завершена: global master содержит короткий R1–R6/K0/M0–M2 contract;
  создан central `request-routing` skill, реализованы lazy route card,
  privacy-safe SQLite telemetry, task/evidence/timeout guards и исправленный
  Claude/Codex hook wiring.
- Fixture validation: 219 обезличенных сценариев; central gate — 128 tests и
  227 subtests; Ruff и hook/config smoke прошли. Для test/build/verify действует
  единый `timeout_ms=600000` с диагностикой по истечении окна.
- Weekly automation «Аудит маршрутизации запросов» включена по средам в 09:00;
  она создаёт только proposals и не меняет taxonomy автоматически.
- #1243 завершена: проаудированы 10 repository roots и subsystem AGENTS в
  AI-Assistant. `GmbH/AGENTS.md` удалён как не содержащий уникальной дельты;
  остальные файлы сокращены до unique project/subsystem rules. Для всех
  изменённых git-repositories `git diff --check` завершился с exit code 0;
  Verua aggregate verify прошёл (Ruff + 63 pytest + repository guards).
- Старые normative Type A/B/C и T1–T11 references удалены. Durable outcomes
  записаны в AI-memory; code/security review не оставил открытых findings.

## Этапы после утверждения

1. Зафиксировать approval в этом файле, записать portfolio `decision` и закрыть
   planning/review-задачу #1233.
2. Создать implementation work items по владельцам: global master/rules,
   central skills/hooks в `_engineering_rules` и config wiring. Follow-up #1243
   уже создан в `К выполнению` и зависит от #1233; project-scoped child tasks
   создаются только при фактической правке соответствующего репозитория.
3. **Build new path:** добавить в highest-scope master короткий router contract,
   создать `request-routing`, реализовать lazy route card R1–R6/M0–M2 и
   task/timeout/CWD contract без инжекта полной taxonomy в каждый prompt.
4. **Migrate dependencies:** перевести `ai-memory-recall`,
   `pm-mcp-task-flow`, `central-plan-workflow`, onboarding/verify/cleanup
   workflows и проектные ссылки.
5. **Hooks:** исправить Codex wiring/README, сделать plan-intent-aware
   `context_gate`, добавить timeout/guard evidence там, где payload позволяет;
   покрыть stdlib tests.
6. **Telemetry/validation:** добавить локальный SQLite+WAL routing audit,
   обезличенные fixtures и метрики; прогнать route/modifier classifier и
   cross-agent smoke в shadow mode. Затем включить warnings и только
   детерминированные hard guards; неизвестный случай не расширяет authority.
7. **Feedback loop:** расширить `agent-session-report` и после отдельного
   подтверждения создать weekly automation, которая группирует аномалии и
   создаёт proposals по thresholds без автоматической правки правил.
8. **Local AGENTS migration (#1243):** после готовности global contract провести
   inventory, поднять повторяемые правила, удалить лишние local files либо
   сократить их до ссылки + уникальной delta; проверить CLAUDE pointers/loader.
9. **Remove old path:** удалить normative Type A/B/C/T references, локальные
   дубли и stale scripts/docs/config, выполнить reference sweep по migration
   discipline.
10. Прогнать gates каждого owner root с десятиминутным cap, live verification,
   security/code review по затронутому scope; записать memory outcomes.
11. Закрыть implementation-задачи. План переносится в `DONE\` только когда
   implementation завершён, уникальные acceptance criteria перенесены в
   текущие rules/docs и lifecycle-чек-лист выполнен.

## Итоговая инженерная ретроспектива

| Ось | Финальный вердикт | Обоснование |
|---|---|---|
| tech-stack-choices.md | no-change | Новых technologies/bricks не появилось; использованы существующие uv/pytest, stdlib hooks, SQLite+WAL и isolated verification |
| Design-system | no-change | UI primitives/tokens не менялись; повторяемая Design-system ownership-инструкция поднята в global master |
| Skills | no-change | Утверждённые central skills обновлены; новых follow-up изменений не требуется |
| Hooks | no-change | Wiring drift, plan-intent, timeout и evidence gaps исправлены и покрыты tests/smoke |

Approval получен 2026-07-16. Scope #1245/#1243 выполнен без
branch/commit/push/PR. План архивируется без переименования после закрытия
#1243.
