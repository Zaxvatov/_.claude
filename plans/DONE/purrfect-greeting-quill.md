# Консолидация хуков: единый Write/Edit path-guard + enforcement деприкейта file-memory + фиксы покрытия

- **Slug:** `purrfect-greeting-quill` (не переименовывать)
- **PM-MCP:** создать после approval — корневая задача в `D:\GitHub\_engineering_rules` + по задаче на фазу (P1→P5, зависимости по порядку)
- **Tech-stack:** новых bricks не требуется (stdlib-хуки; brick #19 pwsh; контракт hook-authoring). Паттерн `path_rules` + dispatcher задокументировать в `hooks/README.md`, не в tech-stack-choices.
- **AI-memory:** решение-основание уже записано (id 1632). Outcome зафиксировать при закрытии.

## Context

Сегодня встроенная Claude Code file-memory (`~/.claude/projects/<proj>/memory/`) деприкейтнута в пользу AI-memory MCP (решение id 1632, задача #852): уникальные факты перенесены, три папки удалены. Но у фичи **нет штатного тумблера** — инструкцию `# Memory` инжектит харнес, поэтому без жёсткого guard'а другая сессия/агент воссоздаст папки. Нужен PreToolUse-хук, который детерминированно запрещает запись туда. **Цель (решение пользователя): никакой агент вообще не может воссоздать file-memory** — поэтому guard двусторонний: file-инструменты (Edit/Write/MultiEdit/NotebookEdit → `write_edit_guard`) и шелл-запись (Bash/PowerShell → `shell_memory_guard`). AGENTS.md-правило мягкое и конкурирует с системным промптом — недостаточно.

При анализе hook-набора под этот guard вскрылись смежные проблемы, которые логично закрыть тем же заходом (пользователь: «без полумер»):

- **B1 (баг).** `frontend_verification_reminder.py` и `migration_doc_guard.py` проводены как **PostToolUse**, но зовут `session_context()` → отдают `hookEventName:"SessionStart"`. Это тот же дефект, что чинили для `plan_lifecycle_guard` в #816; здесь пропущен → оба напоминания, скорее всего, не доходят. (`remind_complete.py` на `Stop` тоже шлёт `SessionStart` — проверить контракт Stop.)
- **O1 (дыра покрытия).** Все guard'ы на matcher `Bash` (`admin_guard`, `python_env_guard`, `require_active_task`) не видят инструмент **`PowerShell`** (его `tool_name` ∉ `BASH_TOOLS`; matcher в settings = `Bash`). На Windows, где PowerShell — основной шелл, мутации (`Remove-Item`), admin-команды (`nssm`/`sc.exe`/`Register-ScheduledTask`) и правило active-task **обходятся**. Паттерны `admin_guard` написаны именно под PowerShell-команды, но события до них не доходят — явный недосмотр проводки.
- **Q1 (шум).** `migration_doc_guard` срабатывает почти на каждом редактировании: `is_migration_sensitive_path` ловит `.py/.md/.sql/.ps1/.yaml/.json/.toml`. Cry-wolf даже после фикса B1.
- **Q2 (производительность).** На каждый `Edit` стартует ~6 python-процессов (2 PreToolUse + 4 PostToolUse). Холодный старт на Windows ~0.15–0.3 c → ~1–1.8 c латентности на правку.
- **Фрагментация.** Path-логика размазана по 4 хукам (`plan_lifecycle_guard`, `frontend_verification_reminder`, `migration_doc_guard`, `powershell_version_guard`).
- **Битая ссылка (следствие сегодняшнего удаления).** `tech-stack-choices.md` brick #19 в Evidence ссылается на удалённый `...\D--GitHub-AI-Assistant\memory\windows-environment.md`. Напомнить должен был `migration_doc_guard` — но он сломан (B1). Чинится здесь же.

**Итог:** свести path-логику в декларативную таблицу + диспетчеры, закрыть PowerShell-дыру, починить B1 by construction, сузить Q1, схлопнуть процессы (Edit: 6→3).

### Учтено по ревью (Codex)
P3 фиксирует живой PowerShell-payload до правки settings; честно описан pipe→ask trade-off + read-only-pipeline allowlist; `admin_guard` split deny/ask (не блокировать non-admin logon-задачи); `changed_texts` покрывает MultiEdit; migration ушёл на shell-delete (реально видит удаления); memory-deny на `ntpath`-нормализации + матрица path-hardening тестов; brick #19 evidence остаётся файловым + id как ссылка; fail-safe (pre→ask) / fail-open (post→allow); admin/env/task НЕ затягиваются в `path_rules`. Цель усилена до «никакой агент не воссоздаёт file-memory» → добавлен `shell_memory_guard` (шелл-запись в memory-путь → deny, консервативный детект); P3/P4 говорят `Bash|<подтверждённый tool_name>` (не хардкод `PowerShell`); лог-хук P3 с safety-условиями; `NotebookEdit.new_source` под проверку payload'ом; P5 sweep на `session_context(` в PostToolUse-хуках.

## Целевая архитектура

**Новый общий слой**
- `hooks/lib/path_rules.py` — упорядоченный список **path-based** правил для Write/Edit. `Rule(name, phase['pre'|'post'], applies(event)->bool, decision['deny'|'ask'|'context'], message)`. Строго про путь/контент редактируемого файла — admin/env/task сюда НЕ затягиваем (см. ниже).
- `hooks/lib/files.py` — добавить:
  - `is_claude_memory_path(path)` — robust-нормализация через `ntpath.normpath` + `os.path.expanduser`/`expandvars`, затем проверка сегментов `…\.claude\projects\…\memory\…`. Покрывает `/`↔`\`, регистр, `~`, `%USERPROFILE%`, `..`.
  - `changed_texts(event) -> list[str]` — единый helper «что добавлено»: Write→`content`, Edit→`new_string`, **MultiEdit→каждый `edits[].new_string`**, NotebookEdit→`new_source` (имя поля **проверить sample-payload'ом**, как и PowerShell `tool_name` — не закладывать вслепую). Без него powershell-правило пропускает MultiEdit.
  - `is_replacement_signal(text)` — маркеры замены/удаления в добавленном контенте (для сужения migration-сигнала на стороне edit).

**Новые хуки — 2 Pre + 2 Post:** `write_edit_guard`, `shell_memory_guard` (PreToolUse); `edit_reminders`, `shell_reminders` (PostToolUse)
- `hooks/write_edit_guard.py` (PreToolUse, matcher `Edit|Write|MultiEdit|NotebookEdit`). Прогоняет pre-правила по приоритету (deny > ask), первое решающее, иначе `allow()`. Правила: (1) **memory-deny (file-tool сторона)** `is_claude_memory_path`→`deny`; (2) plan-вне-plan-mode→`ask`; (3) `Write` поверх существующего `.md`→`ask`. **Fail-safe:** любая неожиданная ошибка → `pretool_ask` с краткой причиной (не падать молча, не пропускать guard).
- `hooks/shell_memory_guard.py` (PreToolUse, matcher `Bash|<подтверждённый PS tool_name>`). **memory-write-deny (шелл-сторона)** — закрывает цель «никакой агент не воссоздаёт file-memory»: команда, пишущая/создающая в `…\.claude\projects\…\memory\…`, → `deny`. Детект консервативный: нормализованный memory-path в строке команды + индикатор записи (`>`/`>>`, `Set-Content`/`Out-File`/`New-Item`/`Add-Content`/`Tee-Object`, `Copy-Item`/`Move-Item -Destination`). Чтение/удаление (`Get-*`/`Remove-Item`) НЕ блокируются. Fail-safe→`ask`. Остаточный риск (честно): запись через произвольную программу (`python -c …`) строковый детект не ловит — это defense-in-depth поверх file-tool deny + AGENTS.md.
- `hooks/edit_reminders.py` (PostToolUse, matcher `Edit|Write|MultiEdit|NotebookEdit`). Собирает все подходящие context-сообщения через `changed_texts` и отдаёт одним `post_tool_context()` (чинит B1, схлопывает 4→1). Правила: plan-lifecycle, frontend-verification, powershell.exe→pwsh, migration-сигнал в контенте (`is_replacement_signal`). **Fail-open:** ошибка → `allow()` (подсказка не должна ломать работу).
- `hooks/shell_reminders.py` (PostToolUse, matcher `Bash|<подтверждённый PS tool_name>`). Migration-напоминание на **реальных удалениях/перемещениях** (`rm`/`mv`/`Remove-Item`/`Move-Item`/`git rm`/`git mv` через `classify_bash`/паттерн) — удаления идут через шелл, а не Edit. Fail-open.

Этим migration-сигнал перестаёт висеть на каждом edit (Q1) и реально видит удаления (закрывает противоречие из ревью): тяжёлая часть — на shell-стороне, лёгкая (replacement в контенте) — на edit-стороне.

**admin/env/task — отдельные концерны, НЕ объединяются в `path_rules`.** Сознательно: `path_rules` про путь файла, а `require_active_task`/`admin_guard`/`python_env_guard` — про задачу/команду/окружение (это не path-логика, иначе кривая абстракция). Их логику в этом плане не трогаем; меняем только проводку под PowerShell (P3).

**Удаляются (migration-discipline):** `plan_lifecycle_guard.py`, `frontend_verification_reminder.py`, `migration_doc_guard.py`, `powershell_version_guard.py` — логика переходит в `path_rules` + три диспетчера (`write_edit_guard` + `edit_reminders` + `shell_reminders`).

**Остаются как есть (отдельные концерны):** `require_active_task` (task-state), `admin_guard`, `python_env_guard`, `memory_write_guard`, `task_state`, `plan_retrospective`, `plan_archive_reminder`, `context_gate`, `session_start`, `remind_complete`.

## Поведенческое изменение — требует явного «ок» (P3)

После закрытия O1 **мутирующие PowerShell-команды начнут требовать активную PM-MCP задачу, а admin-PowerShell — гейтиться** (как сейчас для Bash). Это меняет ежедневный флоу (ручной `Remove-Item`/`nssm`/`Register-ScheduledTask` через PowerShell-инструмент), поэтому фаза вынесена отдельно.

Поправки по ревью (иначе обещание «read-only PS остаётся allow» неверно — `classify_bash` сейчас метит **любой pipe `|` как `ask`**, и `Get-Content x | ConvertFrom-Json` стал бы спрашивать задачу). В P3:
- расширяем `classify_bash`: чистый read-only pipeline (каждый сегмент начинается с известного read-only-префикса и нет mutating-паттерна) → `allow`;
- `admin_guard` разделяем на **deny vs ask**: явная эскалация/служба (`-Verb RunAs`, `RunLevel Highest`, `sc.exe`/`nssm` service-ops, principal SYSTEM) → `deny`; неоднозначный/user-level (`Register-ScheduledTask` с `LogonType Interactive`/`RunLevel Limited`, без admin-маркеров) → `ask` с инструкцией — не блокировать легитимные non-admin logon-задачи;
- **остаточный trade-off (честно):** составные команды через `;`/`&&` и не-allowlist-нутые конвейеры остаются `ask`.

## Фазы (каждая — PM-MCP задача, по порядку)

- **P1 — общий слой.** `lib/path_rules.py` + расширение `lib/files.py` (`is_claude_memory_path`, `changed_texts`, `is_replacement_signal`). Юнит-тесты: **path-hardening матрица** для memory-path (`/`↔`\`, регистр, `~`, `%USERPROFILE%`, `..`, путь к `MEMORY.md` и к per-fact-файлу → match; соседние не-memory пути → no-match); `changed_texts` для Write/Edit/MultiEdit/NotebookEdit; replacement-signal; plan-path.
- **P2 — новые хуки.** `write_edit_guard.py` + `shell_memory_guard.py` + `edit_reminders.py` + `shell_reminders.py` + тесты: memory→`deny` (file-tool); **shell-запись в memory → `deny`** (`Set-Content`/`>` в memory-путь), а чтение/удаление → не блок; plan→`ask`; existing-md→`ask`; каждый reminder с корректным `hookEventName:"PostToolUse"`; объединение сообщений; **MultiEdit-`edits` видны powershell-правилу**; shell-delete→migration-reminder; **malformed/empty stdin → fail-safe** (pre→ask, post→allow). Фиксит B1.
- **P3 — PowerShell-покрытие (поведенческое, по отдельному «ок»).**
  1. **Сначала зафиксировать живой payload:** снять реальное PreToolUse-событие PowerShell-инструмента временным лог-хуком, подтвердить фактический `tool_name` и поля (`tool_input.command`). Matcher/`BASH_TOOLS` подгоняем под факт, не под предположение `"PowerShell"`. **Safety лог-хука:** писать только в `D:\GitHub\_engineering_rules\.state\hooks\` (gitignored), one-shot/короткая retention, НЕ коммитить; `command` целиком не логировать (риск секретов) — только `tool_name`/ключи payload; сразу после capture проводку лог-хука удалить.
  2. `classify_bash`: read-only-pipeline → `allow` (консервативный per-segment allowlist) + тесты.
  3. `admin_guard`: split deny vs ask (эскалация/служба → deny; user-level scheduled task → ask) + тесты.
  4. Проводка: `settings.json` matchers `Bash`→`Bash|<подтверждённый tool_name>` (admin/env), `<тот же tool_name>` в matcher require_active_task; `require_active_task.BASH_TOOLS += "<факт>"`. Тесты с подтверждённым `tool_name`: мутирующая PS → ask/deny; read-only (вкл. pipeline) → allow; service/admin PS → deny; user logon-task → ask.
- **P4 — перепроводка + удаление + доки.** `settings.json`: pre `plan_lifecycle_guard`→`write_edit_guard`, добавить `shell_memory_guard` в PreToolUse `Bash|<подтверждённый tool_name>`; post-блок из 4 хуков → `edit_reminders.py`; добавить `shell_reminders.py` в PostToolUse `Bash|<тот же tool_name>`. Удалить 4 старых хука + их obsolete-тесты. Обновить `hooks/README.md`. Починить brick #19 в `tech-stack-choices.md`: **Evidence оставить файловым** (`tools/user-session/install-user-session.ps1`, `run-user-session.ps1`) + добавить AI-memory id 1629/1632 как ссылку на решение (не заменять файлы на id).
- **P5 — верификация.** Полный `python -m unittest discover -s tests` зелёный (≥48 + новые). Smoke каждого диспетчера (memory-deny, plan-ask, post-reminders, shell-delete, PowerShell-cases, fail-safe на битом stdin). `settings.json` → `ConvertFrom-Json` ок. Sweep'ы: `rg` — нет упоминаний удалённых хуков; `rg "session_context\(" hooks/*reminder*.py hooks/*guard*.py` пусто (B1 не просочился в PostToolUse-хуки).

## Критические файлы

- Новые: `hooks/lib/path_rules.py`, `hooks/write_edit_guard.py`, `hooks/shell_memory_guard.py`, `hooks/edit_reminders.py`, `hooks/shell_reminders.py`, тесты под них.
- Меняются: `hooks/lib/files.py` (+helpers), `hooks/lib/commands.py` (read-only-pipeline в `classify_bash`), `hooks/admin_guard.py` (deny/ask split), `hooks/require_active_task.py` (`BASH_TOOLS`), `~/.claude/settings.json`, `hooks/README.md`, `D:\GitHub\_engineering_rules\tech-stack-choices.md` (brick #19 evidence).
- Удаляются: `hooks/plan_lifecycle_guard.py`, `hooks/frontend_verification_reminder.py`, `hooks/migration_doc_guard.py`, `hooks/powershell_version_guard.py` (+ соответствующие куски тестов).
- Опционально (follow-up, вне этого плана): мастер `AGENTS.md` (Google Drive) — строка про деприкейт file-memory и про PowerShell-гейтинг; глобальная русско-язычная преференция (id 1631). Делать через master-путь, не напрямую.

## Переиспользование (не плодить новое)

- Решения PreToolUse — только через `lib/claude.py`: `pretool_deny`/`pretool_ask`/`post_tool_context`/`allow` (контракт `hookSpecificOutput.permissionDecision` из #816). НЕ возвращать top-level `permissionDecision`.
- Предикаты путей — `lib/files.py` (`touched_path`, `normalized_windows_path`, `is_plan_path`, `is_frontend_path`); добавить рядом, не дублировать нормализацию.
- Классификация shell — `lib/commands.py::classify_bash` (уже покрывает PS-cmdlets: `remove-item`/`new-item`/`set-content`/…).
- Состояние задачи — `lib/state.py` (не трогаем).

## Верификация (end-to-end)

```powershell
cd D:\GitHub\_engineering_rules\hooks
C:\Users\Zaxva\AppData\Local\Programs\Python\Python311\python.exe -m unittest discover -s tests   # зелёный
```
Smoke (echo JSON | hook):
- `write_edit_guard`: `file_path` в `...\.claude\projects\X\memory\Y.md` (и `~`/`%USERPROFILE%`/`/`-варианты) → `deny`; plan вне plan mode → `ask`; обычный `.py` → `{}`; битый stdin → `ask` (fail-safe).
- `edit_reminders`: frontend-путь → `hookEventName=="PostToolUse"` + текст; MultiEdit с `powershell.exe` в одном из `edits` → warning; не-триггер → `{}`; битый stdin → `{}` (fail-open).
- `shell_reminders`: `Remove-Item`/`git rm` → migration-reminder; `Get-ChildItem` → `{}`.
- `shell_memory_guard`: `Set-Content` в `...\.claude\projects\X\memory\f.md` (и `/`/`~`-варианты) → `deny`; `Get-Content`/`Remove-Item` того же пути → не блок; обычная команда → `allow`; битый stdin → `ask`.
- `require_active_task` + `admin_guard` с подтверждённым `tool_name`: мутирующая PS без задачи → `deny`/`ask`; read-only pipeline (`Get-Content | ConvertFrom-Json`) → `allow`; `sc.exe create`/service → `deny`; `Register-ScheduledTask -LogonType Interactive` → `ask`.
- `settings.json` → `Get-Content settings.json | ConvertFrom-Json` без ошибок.

## Риски / откат

- Регрессия в рабочих guard'ах при переносе — митигируется тестами до удаления старых хуков (P2 до P4) и migration-discipline (новый путь зелёный → потом удаление).
- PowerShell-гейтинг может удивить в моменте — read-only PS не затронут; при дискомфорте правило ослабляется точечно в `classify_bash`.
- Откат: `git revert` коммитов фаз; `settings.json` — отдельный коммит, откатывается изолированно.
