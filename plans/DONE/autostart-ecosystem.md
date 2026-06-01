# План: автозапуск экосистемы AI-Assistant (минимальная версия)

## Контекст

После последней миграции четыре подсистемы живут в монорепо
`D:\GitHub\AI-Assistant\`. По ADR-001 публичные daemon'ы
`AI-memory-read`, `AI-memory-write-daemon`, `PM-MCP-server-public`
заменяет единый ingress `gateway/`. Локальные правила
(`ai-memory/AGENTS.md:52`, `pm-mcp-server/AGENTS.md` и
`docs/adrs/0001-target-architecture.md:99`) запрещают новые публичные
поверхности в этих подсистемах.

Gateway пока работает с echo-backend (реальные адаптеры к ai-memory/pm-mcp
ещё не подключены), поэтому он сознательно остаётся **вне boot chain
этого плана**. Включение gateway — отдельная задача после его доводки.

Цель плана: после перезагрузки Windows 11 (Zaxva) автоматически
поднимаются `ai-memory` (loopback 8765), `pm-mcp-server` HTTP (8766) и
`assistant-ui` (8000); работают maintenance-задачи retention/review;
управление состояниями `Active/Idle/Stopped` централизовано в
`pm-mcp-server` Process State Manager (allowlist + EcoQoS).

После доработок при перезагрузке поднимаются: **ai-memory** (main 8765),
**pm-mcp-server** (HTTP 8766), **assistant-ui** (8000).
**Gateway** и публичные read/write daemon'ы — нет (см. контекст).
`!start.bat` остаётся как утилита перезапуска через PM-MCP PSM.

## Карта целевых Scheduled Tasks

| Задача | Тип | Порт | Команда | Триггер | Пользователь | Статус сейчас |
|---|---|---|---|---|---|---|
| `AI-memory-daemon` | network daemon | 8765 | `python -m memory.cli daemon` | WMI ProcessStartTrace + AtLogOn | Zaxva | Installed, путь монорепо — оставить |
| `PM-MCP-server-http` | network daemon | 8766 | `python -m app.http_server` (env `PM_MCP_HTTP_PORT=8766`) | AtLogOn | Zaxva | Missing — создать |
| `Assistant-UI` | network daemon | 8000 | `uvicorn app.main:app --host 127.0.0.1 --port 8000` | AtLogOn | Zaxva | Missing — создать |
| `AI-memory-write-retention` | maintenance | — | существующий retention CLI | Daily 04:00 | `ai-memory-write` | Missing — установить через готовый PS-скрипт |
| `AI-memory-write-review` | maintenance | — | существующий review CLI | Hourly | `ai-memory-write` | Missing — установить через готовый PS-скрипт |
| `MCP-Daily-Backup` | maintenance | — | `daily-backup.ps1` | Daily | Zaxva | Installed — оставить |

**НЕ создавать** в рамках этого плана:
- `AI-memory-read-daemon` (8767) — публичный surface, заменяется gateway.
- `AI-memory-write-daemon` (8770) — публичный surface, заменяется gateway.
- `PM-MCP-server-public` (8769) — публичный surface, заменяется gateway.
- `Gateway` (8780) — пока echo backend; включать после доводки реальных
  адаптеров отдельной задачей.

Healthcheck-цепочка в action скриптах:
- `PM-MCP-server-http` wait `http://127.0.0.1:8765/healthz` (ai-memory).
- `Assistant-UI` wait `http://127.0.0.1:8766/health` (pm-mcp).
Таймаут 60 с, при превышении — daemon стартует с warning в лог.

## Стратегия секретов и auth

PM-MCP HTTP требует bearer-токен, если существует
`~/.assistant-os/auth.token`. Сейчас
`assistant-ui/app/mcp_client.py:434` `_pm_mcp_auth_headers()` читает
только env `PM_MCP_AUTH_TOKEN` и не читает файл. Это блокер для
self-register всех клиентов (ai-memory и assistant-ui под Zaxva).

**Решение**: расширить `_pm_mcp_auth_headers()` (а для ai-memory
написать аналогичный helper в новом utility-модуле) так, чтобы при
пустом env читалось содержимое `~/.assistant-os/auth.token`. Это
единая правка кода, без прокидывания env через Scheduled Task.

Прочие секреты в рамках этого плана не меняются: пароли локальных
пользователей хранит DPAPI Scheduled Task'а, OAuth-токены ChatGPT
остаются в `ai-memory/data/secrets/`. Gateway secret не нужен (gateway
вне boot chain).

## Изменения по подсистемам

### ai-memory (`D:\GitHub\AI-Assistant\ai-memory\`)

- `memory/process_state.py` **не существует** — модуль удалён при
  миграции. Этот шаг плана отменяется.
- На старте `memory/daemon.py` регистрировать собственный pid в
  pm-mcp-server registry через **HTTP best-effort** (`POST
  http://127.0.0.1:8766/mcp/register_process` с retry: 5 попыток
  по 5 с). При недоступности pm-mcp — warning в лог, daemon
  продолжает работать без registry. Параметры: project_path = root
  ai-memory, key = `ai-memory-daemon`, `supports_idle=True`,
  **`allow_stop=False`** (по правилу resource states: `Stopped`
  только при явном правиле; ai-memory в `!start.bat` не трогается).
  Auth: см. секцию «Стратегия секретов».
- `/healthz` уже есть, не трогаем.

### pm-mcp-server (`D:\GitHub\AI-Assistant\pm-mcp-server\`)

1. **`app/process_state.py`** (line 185 `_set_windows_priority`,
   line 252 `apply_process_state`) — расширить:
   - Добавить `PROCESS_POWER_THROTTLING_STATE` (`ctypes.Structure`,
     Version=1, ControlMask DWORD, StateMask DWORD).
   - В `apply_process_state` после `_set_windows_priority`:
     при `IDLE_STATE` — `SetProcessInformation(handle,
     ProcessPowerThrottling=4, byref(state), sizeof)` с
     `ControlMask=PROCESS_POWER_THROTTLING_EXECUTION_SPEED=0x1`,
     `StateMask=0x1` (EcoQoS on);
     при `ACTIVE_STATE` — `ControlMask=0x1, StateMask=0x0` (off).
   - Graceful fallback на старых Windows (build < 22000) — поле
     `ecoqos` в результате = `"unsupported"`, без падения.
   - Сохранять **единый ключ `ecoqos`** в `process_state_audit.details`
     (JSON): `enabled`, `disabled`, `unsupported`, `failed`.
     Один helper `_apply_ecoqos(handle, target)` возвращает это значение.
   - Чтобы не плодить `OpenProcess`, обернуть оба вызова (priority + EcoQoS)
     в один try/finally с одним handle.
2. **`app/http_transport.py`** — `/health` уже есть (line 124),
   не трогаем. На старте `app/http_server.py` регистрировать
   собственный pid **прямым локальным импортом**
   `app.process_state.register_process()` (свой subsystem,
   границы не нарушаются): key = `pm-mcp-http`, pid = `os.getpid()`,
   executable = `sys.executable`, command = строка процесса,
   `supports_idle=True`, **`allow_stop=True`** (участвует в restart-
   утилите), state = `Active` (на каждом старте обновляется).
   `public_http_server` в этом плане не запускается, регистрацию
   там не добавляем.

### assistant-ui (`D:\GitHub\AI-Assistant\assistant-ui\`)

1. **`app/main.py`** — `/health` уже есть (line 183), не трогаем.
   В `@app.on_event("startup")` (рядом с `start_idle_watcher()`)
   добавить регистрацию pid в pm-mcp registry через
   `PMMCPHttpClient().call_tool("register_process", ...)` (метод
   называется `call_tool`, не `call_pm`; `call_pm` — это враппер
   уровнем выше в `MCPClients`). Допустимо и через
   `MCPClients.call_pm(...)`, если он уже инстанцирован в startup.
   Вызов best-effort: warning при failure, без блокировки startup.
   НЕ использовать прямой импорт `app.task_store.process_registry`
   — это нарушит subsystem boundary к pm-mcp.
   Параметры: key = `assistant-ui`, `supports_idle=True`,
   `allow_stop=True`.
2. **`app/ollama_runtime.py`** — без изменений. Существующий
   watchdog (5/15 мин) уже соответствует правилам AGENTS.md
   о heavy compute.
3. **`scripts/install_windows_startup.py`** — новый. Папку `scripts/`
   создать. Контракт по образцу
   `ai-memory/scripts/configure_windows_startup.py`:
   - Команды `install`/`remove`/`status`.
   - Resolve Python: `<root>/.venv/Scripts/python.exe` относительно
     `Path(__file__).resolve().parents[1]`.
   - XML: `LogonTrigger` для Zaxva, `InteractiveToken`,
     `RunLevel=LeastPrivilege`, `ExecutionTimeLimit=PT0S`.
   - Action: `powershell.exe -EncodedCommand <wait-loop +
     Start-Process>`. Wait-loop ждёт `http://127.0.0.1:8766/health`
     до 60 с. Затем `Start-Process` запускает
     `uv run uvicorn app.main:app --host 127.0.0.1 --port 8000`
     с env `ASSISTANT_PM_MCP_BASE_URL=http://127.0.0.1:8766`,
     `ASSISTANT_OLLAMA_AUTO_START=true`, `ASSISTANT_OLLAMA_AUTO_STOP=true`.
     Без `--reload`. Без открытия браузера.
   - Логи в `<root>/.logs/assistant-ui-8000.log`.

### pm-mcp-server install-скрипт

**`pm-mcp-server/scripts/install_windows_startup.py`** — новый, по
образцу assistant-ui. Одна задача `PM-MCP-server-http`. Wait-loop:
`http://127.0.0.1:8765/healthz`. Команда: `uv run python -m
app.http_server` с env `PM_MCP_HTTP_PORT=8766`. Логи в
`<root>/.logs/pm-mcp-http-8766.log`.

`PM-MCP-server-public` не создаём (см. контекст).

### Restart-утилита `!start.bat`

Переписать на audited stop через PM-MCP PSM (allowlist-based + audit;
`apply_process_state` line 193 шлёт `SIGTERM` через `os.kill` — на
Windows это эквивалент жёсткого `TerminateProcess`, **не graceful
shutdown**; формулировать как «audited stop by registered pid»).
Затем повторный запуск через `schtasks /Run`.

`!start.bat` лежит в `assistant-ui/`, поэтому пути вычисляются от
`%~dp0` (правило E.3 — никаких hardcoded `D:\GitHub\...` в коде/
скриптах подсистемы):

Пути передаются через env (а не через raw-string подстановку: `%~dp0`
заканчивается на `\`, и Python `r'...\'` — синтаксическая ошибка):

```
@echo off
setlocal
set "UI_ROOT=%~dp0"
set "MONOREPO_ROOT=%UI_ROOT%.."
set "PM_ROOT=%MONOREPO_ROOT%\pm-mcp-server"

:: Audited stop через PM-MCP PSM (по registered pid).
:: Пути читаются из env через os.environ — без проблем с raw-strings.
uv --directory "%PM_ROOT%" run python -c ^
 "import os; from pathlib import Path; ^
  from app.process_state import set_process_state; ^
  set_process_state(Path(os.environ['UI_ROOT']), 'assistant-ui', 'Stopped', reason='manual restart'); ^
  set_process_state(Path(os.environ['PM_ROOT']), 'pm-mcp-http', 'Stopped', reason='manual restart')"

schtasks /Run /TN "PM-MCP-server-http"
schtasks /Run /TN "Assistant-UI"

start "" "http://127.0.0.1:8000/"
endlocal
```

`AI-memory-daemon` не трогаем — его watcher восстановит daemon сам
при первом запросе от Codex/claude/ollama, и `allow_stop=False`
в registry это защищает.

## Порядок выполнения

0. **PM-MCP задачи** (правило K.2: каждое изменение начинается с work
   item). Создать через `mcp__PM-MCP-server__create_task` по проектам:
   - `pm-mcp-server`: «EcoQoS в PSM (`apply_process_state`) + единый
     ключ `ecoqos`», «Self-register pid в `http_server.py`».
   - `ai-memory`: «HTTP-helper `memory/pm_mcp_client.py` с
     auth-fallback на `~/.assistant-os/auth.token`», «Self-register
     в pm-mcp registry на старте `memory/daemon.py` (best-effort,
     retry, `allow_stop=False`)».
   - `assistant-ui`: «Auth-fallback в `_pm_mcp_auth_headers` на
     `~/.assistant-os/auth.token`», «Self-register через
     `PMMCPHttpClient.call_tool('register_process', ...)` в
     `app/main.py` startup», «Новый `scripts/install_windows_startup.py`
     (Assistant-UI Scheduled Task)», «Переписать `!start.bat` через
     `%~dp0` + audited stop через PM-MCP PSM».
   - `pm-mcp-server`: «Новый `scripts/install_windows_startup.py`
     (PM-MCP-server-http Scheduled Task)».
   - `ai-memory`: «Восстановить скрипт подготовки service user
     `ai-memory-write` (создание + `grant_service_batch_logon_rights`
     + ACL) или задокументировать ручную процедуру».
   Каждая задача линкуется к этому плану в metadata.
1. **Code edits** в pm-mcp-server и assistant-ui (EcoQoS в PSM,
   self-register на старте, маршрут регистрации ai-memory pid,
   auth-fallback на токен-файл). Один atomic commit на монорепо
   (правило J.4).
2. **Новые scripts**: `pm-mcp-server/scripts/install_windows_startup.py`,
   `assistant-ui/scripts/install_windows_startup.py`.
3. **Установка задач**:
   - `python pm-mcp-server/scripts/install_windows_startup.py install`.
   - `python assistant-ui/scripts/install_windows_startup.py install`.
   - **Подготовка пользователя `ai-memory-write`** (один раз, если ещё нет
     — скрипт `install_write_daemon_service.ps1` в монорепо отсутствует,
     поэтому процедура ручная):
     1. `New-LocalUser -Name ai-memory-write -Password (Read-Host
        -AsSecureString) -PasswordNeverExpires`.
     2. (elevated) `pwsh ai-memory/scripts/grant_service_batch_logon_rights.ps1
        -AccountName ai-memory-write -RegisterTask` — выставляет
        `SeBatchLogonRight` через временную Scheduled Task.
     3. Выставить ACL на `data/memory.db`, `data/logs`, `data/uv-cache-write`
        для `ai-memory-write` через `icacls` (паттерн RX/M из удалённого
        `install_write_daemon_service.ps1`).
   - `pwsh ai-memory/scripts/install_write_retention_scheduled_task.ps1
     -RegisterTask` (скрипт сам спросит пароль `ai-memory-write`).
   - `pwsh ai-memory/scripts/install_write_review_scheduled_task.ps1
     -RegisterTask`.
4. **Переписать** `assistant-ui/!start.bat`.
5. Reboot. Верификация.

## Верификация

1. `Get-ScheduledTask | Where TaskName -match
   '^(AI-memory-daemon|PM-MCP-server-http|Assistant-UI|AI-memory-write-(retention|review)|MCP-Daily-Backup)' |
   Select TaskName,State` — все 6 задач Ready/Running.
2. `Test-NetConnection 127.0.0.1 -Port 8765` (повторить 8766, 8000)
   — все три `TcpTestSucceeded=True` в течение 60 с после логона.
3. `curl http://127.0.0.1:8765/healthz` → 200; `:8766/health` → 200;
   `:8000/health` → 200.
4. MCP `mcp__PM-MCP-server__list_registered_processes` — видны
   `ai-memory-daemon`, `pm-mcp-http`, `assistant-ui`.
5. `mcp__PM-MCP-server__set_process_state` state=`Idle` для
   `assistant-ui` → `Get-Process -Id <pid> | Select PriorityClass` =
   `BelowNormal`; `get_process_state_audit` показывает
   `ecoqos=enabled` в details.
6. Тест restart: запустить `!start.bat` — `pm-mcp-http` и
   `assistant-ui` корректно останавливаются (SIGTERM), затем стартуют
   заново через schtasks.
7. `powercfg /requests` — ни один daemon не блокирует sleep.

## Критические файлы

Правки кода:
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\process_state.py` — EcoQoS
  (line 185–280 диапазон). Единый ключ `ecoqos` в результатах и audit.
- `D:\GitHub\AI-Assistant\pm-mcp-server\app\http_server.py` — self-register
  прямым импортом `app.process_state.register_process()`.
- `D:\GitHub\AI-Assistant\ai-memory\memory\daemon.py` — self-register pid
  в pm-mcp registry через HTTP best-effort с retry, `allow_stop=False`.
- `D:\GitHub\AI-Assistant\assistant-ui\app\main.py` — self-register pid
  через `app.mcp_client` API (рядом с `start_idle_watcher`, line 175–181).
- `D:\GitHub\AI-Assistant\assistant-ui\app\mcp_client.py` —
  `_pm_mcp_auth_headers()` (line 433) с fallback на
  `~/.assistant-os/auth.token`.
- Новый утилитарный модуль в ai-memory для HTTP-вызова pm-mcp с тем же
  auth-fallback (например, `memory/pm_mcp_client.py`).

Новые:
- `D:\GitHub\AI-Assistant\pm-mcp-server\scripts\install_windows_startup.py`.
- `D:\GitHub\AI-Assistant\assistant-ui\scripts\install_windows_startup.py`
  (включая создание папки `scripts/`).

Переписывается:
- `D:\GitHub\AI-Assistant\assistant-ui\!start.bat` → restart через PSM.

Без правок (используются как есть):
- `D:\GitHub\AI-Assistant\ai-memory\scripts\install_write_retention_scheduled_task.ps1`.
- `D:\GitHub\AI-Assistant\ai-memory\scripts\install_write_review_scheduled_task.ps1`.
- `D:\GitHub\AI-Assistant\ai-memory\scripts\configure_windows_startup.py`
  (текущая `AI-memory-daemon` уже на пути монорепо).
- `C:\Users\Zaxva\AppData\Roaming\Claude\claude_desktop_config.json`
  (уже указывает на монорепо).

## Риски и митигации

- **EcoQoS недоступен (build < 22000)**: graceful fallback, поле
  `ecoqos=unsupported` в audit.
- **Гонка при логоне**: 3 network-daemon стартуют параллельно. Митигация
  — wait-loop healthcheck (Шаг 3).
- **`Start-Process` background + Scheduled Task**: task мгновенно
  переходит в Ready, реальный daemon живёт независимо. Контроль
  остановки/перезапуска — только через PM-MCP PSM (audited stop по
  pid из registry; `os.kill(pid, SIGTERM)` на Windows эквивалентен
  жёсткому `TerminateProcess`, не graceful). Не использовать
  `schtasks /End` для остановки daemon'ов. Реальный graceful
  shutdown (через Windows console control event) — отдельная задача.
- **Зависимость от auth-token**: если `~/.assistant-os/auth.token`
  отсутствует, fallback не сработает и self-register откажет.
  Митигация — best-effort с warning, daemon продолжает работать.
- **`AI-memory-write` пользователь отсутствует**: при первом запуске
  retention-скрипта с `-CreateUser` запросится пароль. Делается один раз
  вручную. Сам пароль хранится в DPAPI Scheduled Task.
- **Изменение схемы регистрации (ai-memory регистрируется в pm-mcp
  снаружи)**: ai-memory должен уметь обращаться к pm-mcp HTTP. На старте
  ai-memory это значит wait до pm-mcp `/health` (циклическая зависимость:
  ai-memory ждёт pm-mcp, pm-mcp ждёт ai-memory). **Решение**: ai-memory
  стартует без регистрации, регистрирует pid через retry с экспонентой
  (например, 5 попыток по 5 с). Если pm-mcp не поднялся — ai-memory
  работает без registry, без блокировки.

## Что НЕ входит в этот план

- Docker / WSL2, NSSM, mcp-proxy.
- `Gateway` в boot chain — после доводки реальных адаптеров (ADR-001
  следующий шаг).
- Публичные daemon'ы `AI-memory-read`/`-write`,
  `PM-MCP-server-public` — отменены ADR-001 в пользу gateway.
- Удаление `pm-agent/` — отдельная задача по ADR-001.
- Миграция OAuth-токенов ChatGPT — остаются в `ai-memory/data/secrets/`,
  один source of truth.
- `AI_ASSISTANT_GATEWAY_SECRET` storage — добавится в задачу включения
  gateway в boot chain (`setx ... User` под Zaxva).
- Удаление старых папок `D:\GitHub\AI-memory\`, `D:\GitHub\PM-MCP-server\`,
  `D:\GitHub\Assistant-UI\` — отдельная санитарная задача после
  верификации, что монорепо полностью покрыл их функциональность.
