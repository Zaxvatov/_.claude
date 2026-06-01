# План: 5 Windows Services + миграция переименования public-{read,write}

## Контекст

Цель: реализовать **железобетонный автозапуск** 5 сервисов AI-Assistant
после загрузки Windows по схеме стандартной Windows-службы с отложенным
запуском (`Automatic (Delayed Start)`). После reboot сервисы поднимаются
без участия пользователя и без видимых окон. Никакой
доп. observability/retry/supervisor — Windows Service Control Manager
сам логирует старт/падение в `Event Viewer → System` (source `Service
Control Manager`), этого достаточно для расследования постфактум.

Вторая часть плана — закрытие хвостов миграции ADR-001 Шаг 8 (gateway
заменяет три публичных daemon `8767`/`8769`/`8770`). Эта миграция в
монорепо НЕ доведена до конца: остались `memory/read/` package,
`pm-mcp-server/app/public_*` модули и связанные тесты/конфиги. По
правилу Q.1 это удаляется целиком (не переименовывается):

- **`memory/read/` (8767)** и **`pm-mcp-server/app/public_*` (8769)** —
  полное удаление. Gateway проксирует read через ai-memory main loopback
  (8765), а pm-функции через pm-mcp loopback (8766) — старые публичные
  daemon никому не нужны.
- **`memory/write/` (8770)** — **остаётся**, переименовать в
  `memory/proposals/`. Это внутренний backend ai-memory для proposal
  queue (gateway проксирует `memory.propose` сюда). Имя `proposals`
  отражает функциональную роль (хранение proposals на approval) и
  явно отличается от внутреннего main daemon.

Обе части идут одним планом, потому что:
- install-скрипты Windows Services используют новые имена; делать сначала
  автозапуск со старыми, потом переименовывать = двойная работа.
- Правило J.4 cross-subsystem атомарный commit.

## Целевое состояние

### Windows Services (через NSSM)

| Service name | Команда | Порт | Dependency | StartupType |
|---|---|---|---|---|
| `AI-Assistant-AI-memory` | `<ai-memory>\.venv\Scripts\python.exe -m memory.cli daemon` | 8765 | — | Automatic (Delayed Start) |
| `AI-Assistant-AI-memory-proposals` | `<ai-memory>\.venv\Scripts\python.exe -m memory.cli proposals-daemon` | 8770 | AI-Assistant-AI-memory | Automatic (Delayed Start) |
| `AI-Assistant-PM-MCP-server` | `<pm-mcp>\.venv\Scripts\python.exe -m app.http_server` (env `PM_MCP_HTTP_PORT=8766`) | 8766 | — | Automatic (Delayed Start) |
| `AI-Assistant-Assistant-UI` | `<assistant-ui>\.venv\Scripts\python.exe -m app.cli serve --host 127.0.0.1 --port 8000` | 8000 | AI-Assistant-PM-MCP-server | Automatic (Delayed Start) |
| `AI-Assistant-Gateway` | `<gateway>\.venv\Scripts\python.exe -m gateway.app` | 8780 | AI-Assistant-AI-memory, AI-Assistant-AI-memory-public-write, AI-Assistant-PM-MCP-server | Automatic (Delayed Start) |

- Все сервисы под `.\Zaxva` с паролем (LSA DPAPI). Доступ к
  `~/.assistant-os/auth.token`, venv, HKCU из коробки.
- StartupType: **Automatic (Delayed Start)** — стандартный механизм
  Windows для системных сервисов; SCM использует системную отсрочку
  (настраивается через `HKLM\...\Services\<name>\AutoStartDelay`, мы
  оставляем default), что снижает нагрузку на boot.
- DependOnService: SCM ждёт зависимости перед стартом. Race conditions
  (assistant-ui раньше pm-mcp, gateway раньше backends) решены нативно
  без wait-loop в коде.
- Логи stdout/stderr через NSSM AppStdout/AppStderr в едином месте
  `D:\GitHub\AI-Assistant\.logs\services\<service-name>\stdout.log` и
  `stderr.log` с автоматической ротацией при 10 MB.
- В `services.msc` все 5 видны под именами `AI-Assistant-*` —
  группируются алфавитно.
- Падения → `Event Viewer → Windows Logs → System`, source `Service
  Control Manager`. Запуск/остановка → там же.

### Read-daemon (port 8767) и PM-MCP-public-daemon (port 8769)

**Полное удаление**, не регистрация как сервис, не переименование.
Это легаси первой реализации прямого ChatGPT-доступа, отменённой
ADR-001. По правилу Q.1 — старого пути не существует.

## Удаление легаси (8767, 8769) и переименование write-daemon

### Часть 1 — Полное удаление read-daemon (port 8767)

По правилу Q.1, после удаления должно выглядеть так, как будто read-daemon
никогда не существовал.

**В `ai-memory`**:
- Удалить пакет `memory/read/` целиком (`app.py`, `daemon.py`,
  `filters.py`, `security.py`, `__init__.py`).
- Удалить тесты: `tests/test_read_app.py`, `tests/test_read_daemon.py`,
  `tests/test_read_filters.py`, `tests/test_read_security.py`.
- В `memory/cli.py`: удалить команду `read-daemon` и связанные функции.
- В `memory/config.py`, `memory/runtime_contract.py`, `memory/secrets.py`,
  `memory/daemon.py` — удалить любые ссылки на read-daemon (URL, env vars,
  paths, AI_MEMORY_READ_*).
- В `tests/test_daemon.py`, `tests/test_server_cli.py` — выкинуть тесты,
  ссылающиеся на read-daemon.
- Удалить lock-файлы и cache dirs из `data/`: `ai-memory-read-daemon.lock`,
  `uv-cache-read` (если есть).
- Удалить документацию `docs/AI-MEMORY-READ.md` если осталась.
- Env vars `AI_MEMORY_READ_*` — удалить из default'ов и examples.

**В `gateway`**:
- `gateway/config.py`: удалить `ai_memory_read_*_url` если есть.
- `gateway/app.py:625` `upstreams` dict — убедиться что нет ключа
  `ai-memory-read`.
- `gateway/backends.py`: удалить любые refs на read-daemon endpoint.

**В `pm-mcp-server`, `assistant-ui`**: `grep -r "read-daemon\|ai-memory-read\|AI_MEMORY_READ"` → удалить все находки.

### Часть 2 — Полное удаление PM-MCP public-daemon (port 8769)

**В `pm-mcp-server`**:
- Удалить модули целиком:
  `app/public_http_server.py`, `app/public_app.py`,
  `app/public_contract.py`, `app/public_daemon.py`,
  `app/public_admin.py`, `app/public_secrets.py`,
  `app/public_security.py`.
- Удалить связанные тесты (`tests/test_public_*.py` если есть).
- В `app/config.py`, `app/auth.py`, `app/secrets.py` — удалить ссылки
  на public-daemon (URL, env vars, paths).
- В `app/http_transport.py` и `app/audit.py` — проверить, нет ли
  tool-имён, специфичных для public.
- Удалить install-скрипты public daemon если остались.
- Env vars `PM_MCP_PUBLIC_*` — удалить из default'ов.

**В `gateway`**:
- `gateway/config.py`: убедиться, что `pm_mcp_base_url` указывает на
  loopback (8766), не на public (8769).
- Удалить ссылки на 8769.

**В `assistant-ui`**: `grep -r "public_http_server\|pm-mcp-public\|PM_MCP_PUBLIC"` → удалить находки.

### Часть 3 — Переименование write-daemon → proposals-daemon

Write-daemon остаётся (gateway проксирует `memory.propose` сюда),
переименовать в `proposals-daemon` — функциональное имя, отражает роль
(хранение proposals на approval). Убирает термин «write» из сервисного
слоя ai-memory; «write» остаётся только как операция в API (`store_memory`).

**В `ai-memory`**:
1. CLI: `python -m memory.cli write-daemon` → `python -m memory.cli proposals-daemon` (в `memory/cli.py`). Также `write-retention-run-now` → `proposals-retention-run-now`, `write-review-*` → `proposals-review-*` и т.п. для всех write-* команд, относящихся к proposal queue.
2. Package: `memory/write/` → `memory/proposals/` (Python snake_case). Все импорты обновить.
3. Service user (если используется): `ai-memory-write` → `ai-memory-proposals` (удалить старого, создать нового с теми же ACL).
4. Env vars: `AI_MEMORY_WRITE_*` → `AI_MEMORY_PROPOSALS_*` (`ENABLED`, `PROJECT_ALLOWLIST`, `ALLOWED_HOSTS`, `EXTERNAL_BASE_URL`).
5. PSM `process_key`: `ai-memory-write` → `ai-memory-proposals`. Migration script.
6. Lock-файлы / cache dirs: `data/ai-memory-write-daemon.lock` → `ai-memory-proposals-daemon.lock`, `data/uv-cache-write` → `data/uv-cache-proposals`.
7. Документация: `docs/AI-MEMORY-WRITE.md` → `docs/AI-MEMORY-PROPOSALS.md`. Ссылки в README/AGENTS/ARCHITECTURE обновить.
8. Тесты: `tests/test_write_*.py` → `tests/test_proposals_*.py`. Импорты внутри обновить.
9. Maintenance scripts: `install_write_retention_scheduled_task.ps1` → `install_proposals_retention_scheduled_task.ps1`; `install_write_review_scheduled_task.ps1` → `install_proposals_review_scheduled_task.ps1`. Env vars внутри тоже обновить на `AI_MEMORY_PROPOSALS_*`. Имена задач планировщика `AI-memory-write-retention` → `AI-memory-proposals-retention`, `AI-memory-write-review` → `AI-memory-proposals-review`.

**В `gateway`**:
- `gateway/config.py`: `ai_memory_write_mcp_url` → `ai_memory_proposals_mcp_url`. Env `AI_ASSISTANT_GATEWAY_AI_MEMORY_WRITE_URL` → `..._PROPOSALS_URL`.
- `gateway/app.py:626`: ключ в `upstreams` `"ai-memory-write"` → `"ai-memory-proposals"`.
- `gateway/backends.py`: hardcoded refs обновить.

**В `pm-mcp-server`, `assistant-ui`**: `grep -r "write-daemon\|ai-memory-write\|AI_MEMORY_WRITE"` → переименовать все находки в `proposals`-варианты.

### Migration через PM-MCP tools (не прямой SQL)

Прямой SQL UPDATE в `process_registry` — анти-паттерн (обход PSM,
никакого audit'а). Вместо этого расширить PM-MCP двумя MCP-tools:

1. **`rename_process_key(project_path, old_key, new_key)`** —
   обновляет `process_key` для записи + пишет в `process_state_audit`
   с `reason='registry_rename', details={"old_key": ..., "new_key": ...}`.
   Атомарно через `UNIQUE(project, process_key)` constraint.
2. **`unregister_process(project_path, process_key)`** — удаляет
   запись из `process_registry` + audit `reason='registry_unregister'`.
   Используется для cleanup stale entries после удаления subsystem.

Регистрация: новые имена в `app/http_transport.py` TOOL_NAMES, обе в
`app/audit.py` WRITE_TOOL_NAMES.

Использование во время миграции — Python-скрипт или ручные вызовы
`mcp__PM-MCP-server__*`:
```
rename_process_key("D:\\GitHub\\AI-Assistant\\ai-memory",
                   "ai-memory-write", "ai-memory-proposals")
# ai-memory-read / pm-mcp-public — записей вероятно нет; на всякий случай:
unregister_process("D:\\GitHub\\AI-Assistant\\ai-memory", "ai-memory-read")
unregister_process("D:\\GitHub\\AI-Assistant\\pm-mcp-server", "pm-mcp-public")
```

Tools идемпотентны (no-op если записи нет). Никаких backwards-compatibility
shims (правило Q.3).

## Реализация автозапуска

### Шаг 1 — NSSM

Скачать NSSM (https://nssm.cc/) — один exe, ~330KB. Положить в:
`D:\GitHub\AI-Assistant\tools\nssm\win64\nssm.exe`. Версия 2.24+
(поддерживает StartupType Delayed-Auto через `nssm set <name> Start
SERVICE_DELAYED_AUTO_START` — это нативный SCM-флаг, отсрочка
определяется системными настройками).

NSSM — каноническая реализация (один exe, без зависимостей,
process wrapper для любого исполняемого, поддержка StartupType,
DependOnService, log rotation).

### Шаг 2 — Регистрационный скрипт

**Новый файл** `D:\GitHub\AI-Assistant\tools\register_services.ps1`
(elevated PowerShell). Ключевые контракты:

- **Explicit map service → subsystem path / readme** (не вычисление из
  имени сервиса). `AI-Assistant-AI-memory-proposals` живёт в подсистеме
  `ai-memory`, не `AI-memory-proposals`. Карта явная.
- **DependOnService через splatting** (`& $nssm set $Name DependOnService
  @DependsOn`) или fallback через `sc.exe config <name> depend=
  Service1/Service2` (нативный SCM формат с `/` разделителем). Comma-join
  для NSSM не работает корректно — каждая зависимость отдельным аргументом.
- **AppEnvironmentExtra одним вызовом**: NSSM перетирает значения при
  повторных `set`. Собрать все `KEY=VALUE` строки в массив и вызвать
  `& $nssm set $Name AppEnvironmentExtra @envPairs` один раз.
- **Единое место логов**: `D:\GitHub\AI-Assistant\.logs\services\<service>\`
  (не в каждой подсистеме). Проще найти, единая ротация, единый
  `.gitignore`. Создаётся скриптом перед регистрацией.

Скелет:
```powershell
param([Parameter(Mandatory=$true)][System.Management.Automation.PSCredential]$Credential)

$nssm     = "$PSScriptRoot\nssm\win64\nssm.exe"
$monoRoot = Split-Path -Parent $PSScriptRoot
$logsRoot = Join-Path $monoRoot '.logs\services'
$Username = $Credential.UserName
$Password = $Credential.GetNetworkCredential().Password

# Явная карта: имя сервиса → подсистема + команда + env + зависимости.
$services = @(
    @{ Name='AI-Assistant-AI-memory'
       Sub='ai-memory'
       Args=@('-m','memory.cli','daemon')
       Env=@{}
       Deps=@() },
    @{ Name='AI-Assistant-AI-memory-proposals'
       Sub='ai-memory'
       Args=@('-m','memory.cli','proposals-daemon')
       Env=@{}
       Deps=@('AI-Assistant-AI-memory') },
    @{ Name='AI-Assistant-PM-MCP-server'
       Sub='pm-mcp-server'
       Args=@('-m','app.http_server')
       Env=@{ PM_MCP_HTTP_PORT='8766'; PM_MCP_HTTP_HOST='127.0.0.1' }
       Deps=@() },
    @{ Name='AI-Assistant-Assistant-UI'
       Sub='assistant-ui'
       Args=@('-m','app.cli','serve','--host','127.0.0.1','--port','8000')
       Env=@{ ASSISTANT_PM_MCP_BASE_URL='http://127.0.0.1:8766';
              ASSISTANT_OLLAMA_AUTO_START='true';
              ASSISTANT_OLLAMA_AUTO_STOP='true' }
       Deps=@('AI-Assistant-PM-MCP-server') },
    @{ Name='AI-Assistant-Gateway'
       Sub='gateway'
       Args=@('-m','gateway.app')
       Env=@{ AI_ASSISTANT_GATEWAY_HOST='127.0.0.1';
              AI_ASSISTANT_GATEWAY_PORT='8780' }
       Deps=@('AI-Assistant-AI-memory',
              'AI-Assistant-AI-memory-proposals',
              'AI-Assistant-PM-MCP-server') }
)

foreach ($s in $services) {
    $workDir   = Join-Path $monoRoot $s.Sub
    $python    = Join-Path $workDir '.venv\Scripts\python.exe'
    $logDir    = Join-Path $logsRoot $s.Name
    $readmeRel = "$($s.Sub)\README.md"
    New-Item -ItemType Directory -Force -Path $logDir | Out-Null

    & $nssm install $s.Name $python @($s.Args)
    & $nssm set $s.Name AppDirectory  $workDir
    & $nssm set $s.Name DisplayName   "AI-Assistant: $($s.Sub)"
    & $nssm set $s.Name Description   "AI-Assistant $($s.Sub) service. See $monoRoot\$readmeRel"
    & $nssm set $s.Name Start         SERVICE_DELAYED_AUTO_START
    & $nssm set $s.Name ObjectName    ".\$Username" $Password
    & $nssm set $s.Name AppStdout     (Join-Path $logDir 'stdout.log')
    & $nssm set $s.Name AppStderr     (Join-Path $logDir 'stderr.log')
    & $nssm set $s.Name AppRotateFiles 1
    & $nssm set $s.Name AppRotateBytes 10485760    # 10 MB

    # Env одним вызовом: NSSM перетирает значения при повторных set.
    if ($s.Env.Count -gt 0) {
        $envPairs = $s.Env.GetEnumerator() | ForEach-Object { "$($_.Key)=$($_.Value)" }
        & $nssm set $s.Name AppEnvironmentExtra @envPairs
    }

    # DependOnService отдельными аргументами (splatting), не comma-join.
    if ($s.Deps.Count -gt 0) {
        & $nssm set $s.Name DependOnService @($s.Deps)
        # Fallback / sanity check через sc.exe (нативный формат depend= s1/s2):
        # & sc.exe config $s.Name depend= ($s.Deps -join '/')
    }
}
```

Запуск (elevated):
```powershell
$cred = Get-Credential -UserName Zaxva -Message "Zaxva password for service account"
.\register_services.ps1 -Credential $cred
```

NSSM сохранит пароль в LSA через DPAPI. При смене пароля Windows нужно
вызвать `nssm set <Name> ObjectName .\Zaxva NewPassword` для каждого из 5.

`.logs/services/` добавляется в корневой `.gitignore`.

### Шаг 3 — Cleanup старых механизмов автозапуска

**Удалить Scheduled Tasks** (elevated):
```powershell
schtasks /Delete /TN "AI-memory-daemon" /F
schtasks /Delete /TN "PM-MCP-server-http" /F
schtasks /Delete /TN "Assistant-UI" /F
```

**Удалить старые скрипты автозапуска** (правило Q.1 — старого пути не
существует):
- `D:\GitHub\AI-Assistant\ai-memory\scripts\configure_windows_startup.py` — удалить (заменён NSSM).
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\install_windows_startup.py` — удалить.
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\start_pm_http.ps1` — удалить.
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\run_hidden.vbs` — удалить.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\install_windows_startup.py` — удалить.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\start_assistant_ui.ps1` — удалить.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\run_hidden.vbs` — удалить.
- `D:\GitHub\AI-Assistant\assistant-ui\!start.bat` — **не трогаем в этом
  плане** (оставлен как есть; после удаления старых Scheduled Tasks
  `schtasks /Run "PM-MCP-server-http"` внутри `!start.bat` начнёт
  возвращать ошибку, но это разбирается отдельной задачей).

### Шаг 4 — Документация

В README каждой подсистемы добавить раздел «Autostart / Windows Service»:
- Service name
- Команда `Get-Service AI-Assistant-*`
- Где смотреть логи: services log + `.logs\` + Event Viewer System
- Как перезапустить: `Restart-Service`
- Как удалить: `nssm remove AI-Assistant-<name> confirm`
- Где регистрационный скрипт: `tools/register_services.ps1`

Корневой `D:\GitHub\AI-Assistant\README.md` — раздел «Production autostart»
с общим обзором 5 сервисов, таблицей портов и зависимостями.

`docs/adrs/0001-target-architecture.md` — обновить Шаг 8 (gateway) и
Шаг 7 (PSM) ссылками на новый раздел README.

## Порядок выполнения

1. **PM-MCP work items** (правило K.2) — создать через
   `mcp__PM-MCP-server__create_task` по проектам:
   - `ai-memory`: **(а)** удаление read-daemon целиком (package, тесты,
     cli, refs); **(б)** переименование write-daemon → proposals-daemon
     (cli, modules, env vars, lock files, docs, tests, PSM keys,
     maintenance scripts).
   - `pm-mcp-server`: **(а)** удаление `app/public_*` модулей и
     связанных тестов/refs; **(б)** grep + обновление refs на ai-memory
     write → proposals.
   - `gateway`: удаление refs на ai-memory-read; переименование refs
     `ai_memory_write_*` → `ai_memory_proposals_*`; проверка что
     `pm_mcp_base_url` указывает на 8766, а не 8769.
   - `assistant-ui`: grep + удаление refs на read-daemon и
     pm-mcp-public, переименование write → proposals.
   - `pm-mcp-server`: **(в)** новые MCP-tools `rename_process_key` и
     `unregister_process` (для миграции process_key без прямого SQL).
   - `portfolio` (cross-cutting): новый `tools/register_services.ps1`,
     удаление старых install-скриптов и Scheduled Tasks; миграция PSM
     через вызовы `rename_process_key` / `unregister_process` (не SQL).
2. **Миграция переименования** в коде (один atomic commit на монорепо
   по J.4): все 4 подсистемы + tools одновременно.
3. **NSSM download** в `tools/nssm/win64/nssm.exe`.
4. **Регистрационный скрипт** `tools/register_services.ps1`.
5. **Migration PSM** через MCP-tools: вызовы `rename_process_key
   ai-memory-write → ai-memory-proposals` + `unregister_process` для
   stale записей. Не сырой SQL.
6. **Elevated cleanup**: удалить 3 старых Scheduled Tasks.
7. **Elevated registration**: запустить `register_services.ps1
   -Credential (Get-Credential)`.
8. **Reboot test** + верификация.

PM-MCP work items создаются **после согласования этого плана с Codex**.

## Критические файлы

Изменяются:
- `D:\GitHub\AI-Assistant\ai-memory\memory\cli.py` — переименование
  команд `write-daemon`/`read-daemon` → `public-write-daemon`/`public-read-daemon`.
- `D:\GitHub\AI-Assistant\ai-memory\memory\write\` → `memory\public_write\`
  (переименование package).
- `D:\GitHub\AI-Assistant\ai-memory\memory\read\` → `memory\public_read\`.
- `D:\GitHub\AI-Assistant\ai-memory\scripts\install_read_daemon_service.ps1`
  → `install_public_read_daemon_service.ps1` (переименовать файл и
  содержимое: имя task, env vars, lock files, cache dirs).
- `D:\GitHub\AI-Assistant\ai-memory\scripts\install_write_*` —
  retention/review остаются, но env vars `AI_MEMORY_WRITE_*` →
  `AI_MEMORY_PUBLIC_WRITE_*`.
- `D:\GitHub\AI-Assistant\ai-memory\docs\AI-MEMORY-WRITE.md` →
  `AI-MEMORY-PUBLIC-WRITE.md`. То же для READ.
- `D:\GitHub\AI-Assistant\ai-memory\tests\test_write_*.py` →
  `test_public_write_*.py`. То же для read.
- `D:\GitHub\AI-Assistant\gateway\gateway\config.py` —
  `ai_memory_write_mcp_url` → `ai_memory_public_write_mcp_url`.
- `D:\GitHub\AI-Assistant\gateway\gateway\app.py` (line 624-627) —
  ключи upstreams.
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\*` — grep + переименование
  любых упоминаний `ai-memory-write`, `ai-memory-read`, `write-daemon`,
  `read-daemon`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\*` — то же.

Удаляются:
- `D:\GitHub\AI-Assistant\ai-memory\scripts\configure_windows_startup.py`.
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\install_windows_startup.py`.
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\start_pm_http.ps1`.
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\run_hidden.vbs`.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\install_windows_startup.py`.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\start_assistant_ui.ps1`.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\run_hidden.vbs`.

Создаются:
- `D:\GitHub\AI-Assistant\tools\nssm\win64\nssm.exe` (бинарь, не в git
  — добавить в `.gitignore`).
- `D:\GitHub\AI-Assistant\tools\register_services.ps1`.
- В `pm-mcp-server\app\process_state.py` — новые MCP-tools
  `rename_process_key`, `unregister_process` (заменяют прямой SQL).
- В `pm-mcp-server\app\http_transport.py` — добавить в TOOL_NAMES;
  `app\audit.py` — в WRITE_TOOL_NAMES.
- `D:\GitHub\AI-Assistant\tools\README.md` — инструкции запуска
  `register_services.ps1`.

Обновляются README:
- Корневой `D:\GitHub\AI-Assistant\README.md` — раздел «Production
  autostart».
- README каждой подсистемы — раздел «Autostart / Windows Service».

## Верификация

1. **Чистый reboot**: после загрузки и ~2 минут (Delayed Start)
   `Get-Service AI-Assistant-* | Format-Table Name,Status,StartType` —
   все 5 в статусе `Running`, StartType `Automatic (Delayed Start)`.
2. **Порты**: `Test-NetConnection 127.0.0.1 -Port <p>` для 8765, 8766,
   8000, 8770, 8780 — все True.
3. **Health endpoints**: `/healthz` или `/health` каждого daemon
   отвечает 200.
4. **services.msc**: все 5 видны, описание понятное, StartupType
   `Automatic (Delayed Start)`, ObjectName `.\Zaxva`.
5. **Видимые окна**: `Get-Process | Where MainWindowTitle` не показывает
   ни одного консольного окна, связанного с daemon'ами.
6. **Event Viewer**: после reboot в `Windows Logs → System`, source
   `Service Control Manager`, есть события `Service '<name>' entered the
   running state` для каждого из 5. После `Stop-Service` — `entered the
   stopped state`.
7. **Dependencies**: `Stop-Service AI-Assistant-PM-MCP-server` — SCM
   автоматически остановит `AI-Assistant-Assistant-UI` и
   `AI-Assistant-Gateway` (если предупредить `-Force`).
8. **Логи**: `D:\GitHub\AI-Assistant\.logs\services\<service>\stdout.log` и `stderr.log` существуют для каждого из 5 сервисов, содержат стандартный stdout/stderr daemon'ов, ротируются при 10 MB.
9. **Удаление легаси**: `grep -r "memory.read\|read-daemon\|ai-memory-read\|AI_MEMORY_READ\|public_http_server\|public_app\|pm-mcp-public\|PM_MCP_PUBLIC"` в монорепо возвращает **пустой результат** (всё удалено).
10. **Переименование write → proposals**: `grep -r "write-daemon\|ai-memory-write\|AI_MEMORY_WRITE\|memory/write/"` возвращает только историю в git log, не actual файлы.
11. **PSM registry**: `mcp__PM-MCP-server__list_registered_processes` показывает `ai-memory-proposals` (не `ai-memory-write`); записей `ai-memory-read` или `pm-mcp-public` нет.
12. **Падение симуляция**: `Stop-Service AI-Assistant-AI-memory -Force` → в Event Viewer System log запись о stopped, dependent сервисы тоже остановлены SCM. Запустить вручную → SCM поднимает в правильном порядке по DependOnService.

## Риски

- **NSSM требует admin** для регистрации/удаления сервисов: задокументировать
  в README что регистрация — один раз через `register_services.ps1`
  под admin. Обновления не требуют admin (только `Restart-Service`).
- **Пароль Zaxva в LSA через DPAPI**: при смене пароля Windows — все 5
  сервисов перестанут стартовать с ошибкой `Logon failure`. Документация
  в README: команда `nssm set <name> ObjectName .\Zaxva <new_password>`
  для каждого. Альтернатива — Group Managed Service Accounts (gMSA), но
  это требует domain controller; для personal машины не подходит.
- **Atomic-миграция переименования**: большой объём правок. Митигация —
  Codex делает в одном commit с тестами, перед commit `pytest` каждой
  подсистемы. По правилу M.3 тесты обязательны.
- **PSM migration через MCP-tools**: `rename_process_key` идемпотентен,
  но может конфликтовать если новый daemon успевает self-register между
  migration и stop старого. Митигация — выполнять миграцию **после**
  stop старых сервисов, **перед** регистрацией новых.
- **Gateway upstream `ai-memory-proposals`**: если proposals daemon
  упадёт, gateway пишет warning на старте и **memory.propose не
  работает**, остальные функции работают. Это явный сигнал в Event Viewer
  System log (Service Control Manager: Service entered the stopped
  state).
- **Удаление `configure_windows_startup.py`**: ломает любые внешние
  скрипты, которые на него ссылаются. Митигация — `grep -r
  configure_windows_startup` перед удалением.
- **`memory/write/` → `memory/public_write/` package rename**: если
  где-то в монорепо есть `from memory.write import ...` — упадёт после
  rename. Митигация — глобальный grep + atomic commit, тесты обнаружат.
- **Существующие данные `data/ai-memory-write-daemon.lock` etc**: не
  переносятся автоматически. Митигация — добавить в migration script
  rename файлов или просто удалить старые (daemon создаст новые при
  старте под новым именем).

## Что НЕ входит

- Supervisor / reconciler / RestartCount / Repetition — не нужны (SCM
  стандартен).
- Event Viewer integration в Python коде (NTEventLogHandler) —
  не нужно, SCM сам логирует системные события.
- PSM observability поля (last_seen_at, etc.) — отдельная задача потом.
- Диагностический CLI `investigate-crash` — потом, если понадобится.
- Retry в Assistant-UI startup — DependOnService решает race нативно.
- AI-memory public-read-daemon в автозапуске — переименование делается,
  но сервис не регистрируется (gateway проксирует read через main).
- Удаление старых папок `D:\GitHub\AI-memory\` / `\PM-MCP-server\` /
  `\Assistant-UI\` — отдельная санитарная задача.
- PM-MCP work items — после согласования с Codex по этому плану.
- Group Managed Service Accounts (gMSA) для passwordless service —
  требует domain controller, не подходит для personal.
