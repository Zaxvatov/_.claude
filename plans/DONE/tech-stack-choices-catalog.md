# Tech stack choices catalog + новая plans convention

> Каталог зафиксированных технологических выборов («кирпичиков») стека +
> закрепление новой plans-convention (`D:\GitHub\_engineering_rules\plans\`)
> в global AGENTS.md. Каждый кирпичик — выбранный подход с trade-off и
> evidence в реальном коде. Накопление постепенное.

## Context

В существующих проектах (`ai-memory`, `pm-mcp-server`, `assistant-ui`,
`gateway`, отчасти `Design-system`) повторно используются одни и те же
технологические «кирпичики»: NSSM-обёртки для Windows daemons,
`sentence-transformers all-MiniLM-L6-v2` локально, SQLite+FTS5+WAL как
storage, FastMCP для MCP-серверов, `uv` + per-subsystem `.venv`, namespaced
env vars с explicit prefix-chain fallback. Эти выборы фактически приняты, но
нигде не зафиксированы как явный «применяем X, не Y, потому что Z».

Параллельно пользователь ввёл новое convention: все plan drafts живут в
`D:\GitHub\_engineering_rules\plans\` (зарегистрировано в
`~/.claude/settings.json:4`, `plansDirectory`). Это меняет global правило про
`<project>/docs/plans/_drafts/`. Этот переход нужно зафиксировать в global
AGENTS.md одновременно с созданием каталога tech-stack-choices.

Структура `D:\GitHub\_engineering_rules\` становится домом для всех org-level
«кирпичиков»: `plans/` (планы), `tech-stack-choices.md` (этот план),
будущая `Design-system/` (после migration — отдельный план).

## История review

- Codex round 1 (по предыдущей итерации плана) указал 6 правок:
  (1) draft в проекте, harness — pointer; (2) AGENTS.md на English;
  (3) bullet в existing list, не новый раздел R; (4) repo-relative paths;
  (5) 1 PM-MCP задача; (6) смягчить категоричные формулировки. Все учтены.
- Codex round 2 указал 4 правки (3 P1/P2, 1 P3):
  (1 P1) draft всё ещё в harness — нужно перенести в проектную папку;
  (2 P2) stale line numbers в Evidence;
  (3 P2) противоречие repo-relative — Design-system оставлен как
  абсолютный peer-path;
  (4 P3) docs-language формулировка слишком категорична для open question.
- Round 2 правки + новое требование пользователя (plans в `_engineering_rules/plans/`)
  + Design-system migration как открытый вопрос учтены ниже.

## Принятые решения (lightweight + новый location)

| # | Решение | Обоснование |
|---|---|---|
| 1 | Каталог = один markdown-файл `D:\GitHub\_engineering_rules\tech-stack-choices.md` | Org-level (peer с `plans/` и будущей `Design-system/`); доступен всем проектам через absolute path |
| 2 | Plan draft = `D:\GitHub\_engineering_rules\plans\tech-stack-choices-catalog.md` | Новая convention пользователя (`plansDirectory` в settings.json) |
| 3 | Harness `C:\Users\Zaxva\.claude\plans\*.md` = только pointer на draft | Согласно existing global rule про harness; финальное состояние — после approve |
| 4 | Источник кирпичиков: реальный код + утверждённый план 590 | Опираемся на факт, не теорию |
| 5 | Шаблон кирпичика: 5 секций (Что / Почему / Trade-off / Применимость / Evidence) | Минимум, достаточный для агентов без overhead |
| 6 | Накопление: при следующих планах, по факту нового подтверждённого выбора | Нет «спринта на bootstrap», нет stale risk |
| 7 | Discoverability: bullet в `D:\GitHub\AI-Assistant\AGENTS.md:21` existing list + раздел в global AGENTS.md | AI-Assistant агенты обязаны читать root AGENTS.md; global фиксирует org-wide обязательство |
| 8 | Никакой инфраструктуры (frontmatter индексы, MCP search, hooks) | Полностью съело бы выигрыш |
| 9 | Repo-relative paths для AI-Assistant Evidence (`ai-memory/...`); absolute для cross-repo (`D:\GitHub\_engineering_rules\...`) | Repo-relative внутри одного repo, absolute для cross-repo связей |
| 10 | Без line numbers в Evidence (только файлы) | Line numbers быстро устаревают; agents найдут по `Grep` |
| 11 | 1 PM-MCP задача под всю инициативу (зарегистрировать проект `engineering_rules`) | Изменение мелкое; cross-cutting: AI-Assistant AGENTS.md + global AGENTS.md + новый файл |
| 12 | Фиксация новой `plansDirectory` convention в global AGENTS.md (раздел A) | Закрепить правило, чтобы агенты не путались с старым `<project>/docs/plans/_drafts/` |
| 13 | Design-system migration в `_engineering_rules/Design-system/` — отдельный план | Out of scope текущего; этот план только создаёт каталог tech-stack-choices |

## Шаблон кирпичика

```markdown
### <Имя кирпичика>: <выбранный подход>
- **Что**: одна фраза, что именно выбрано.
- **Почему**: 1-3 фразы, основная мотивация.
- **Trade-off**: что приняли как цену.
- **Применимость**: где используется; где НЕ подходит.
- **Evidence**: `<repo-relative path>` или `<absolute path>` для cross-repo.
```

## Первые 7 кирпичиков (содержимое `tech-stack-choices.md`)

См. готовый файл `D:\GitHub\_engineering_rules\tech-stack-choices.md`.

## План на 1 PM-MCP задачу (3 файла, 1-2 коммита)

Регистрация нового PM-MCP проекта `engineering_rules` с `project_path = D:\GitHub\_engineering_rules` ДО создания задачи (так же, как для AI-Assistant subprojects). Если PM-MCP сам не создаёт записи для нового project_path — попробовать первый `create_task`, проверить результат, при ошибке вызвать explicit регистрацию (если такой инструмент существует) или обсудить с пользователем.

| Шаг | Изменение | Файл |
|---|---|---|
| A | Создать каталог 7 кирпичиков | `D:\GitHub\_engineering_rules\tech-stack-choices.md` (Russian — project doc) |
| B | Добавить bullet 5 в existing list `Before any significant change, read` | `D:\GitHub\AI-Assistant\AGENTS.md:21-25` (English — root AGENTS.md) |
| C | Зафиксировать новую `plansDirectory` convention + ссылку на tech-stack-choices в global AGENTS.md | `C:\Users\Zaxva\.codex\AGENTS.md` (импортируется в `~/.claude/CLAUDE.md` через `@`) |

### Текст для AI-Assistant `AGENTS.md` (English, bullet в existing list)

```markdown
Before any significant change, read:
1. This file.
2. The local `AGENTS.md` of the subsystem you're editing.
3. The local `ARCHITECTURE.md` of that subsystem.
4. The relevant ADR(s) in `docs/adrs/`.
5. `D:\GitHub\_engineering_rules\tech-stack-choices.md` when the change introduces or challenges a recurring technology choice.
```

### Текст для global `C:\Users\Zaxva\.codex\AGENTS.md` (English, обновление раздела A)

Заменить существующий bullet `Plan files` (строки 37-47) и добавить новый bullet `Engineering rules root` сразу после.

```markdown
- Plan files: agents keep shared plans in `D:\GitHub\_engineering_rules\plans\`
  (registered in `~/.claude/settings.json` under `plansDirectory`). Use slugs
  in kebab-case (≤40 chars), file name `<slug>.md`. After approval and PM-MCP
  task creation, rename to `<id>-<slug>.md`. The harness file in
  `~/.claude/plans/<random>.md` or the Codex equivalent contains only
  `См. план: D:\GitHub\_engineering_rules\plans\<file>.md`. After the task
  closes, delete the plan only when: the task is closed, the outcome is in
  AI-memory, any architecture rationale is in ADR/docs, and no unique
  acceptance criteria remain only in the plan. Legacy project-local drafts in
  `<project>/docs/plans/_drafts/` are preserved as historical records; new
  drafts go into the central directory.

- Engineering rules root: `D:\GitHub\_engineering_rules\` stores org-level
  building blocks (`tech-stack-choices.md` and similar) applicable to all
  projects. Before introducing an alternative technology — read the
  corresponding entry in `tech-stack-choices.md` and justify deviation in
  the plan.
```

## Verification

- [ ] PM-MCP проект `engineering_rules` создан или авто-зарегистрирован.
- [ ] 1 PM-MCP задача создана в проекте `engineering_rules` со scope трёх файлов выше.
- [ ] `D:\GitHub\_engineering_rules\tech-stack-choices.md` создан, содержит 7 пунктов по шаблону.
- [ ] `D:\GitHub\AI-Assistant\AGENTS.md` содержит bullet 5 в существующем списке (после строки 25 «4. The relevant ADR(s)…»).
- [ ] `C:\Users\Zaxva\.codex\AGENTS.md` содержит обновлённое правило про `plansDirectory` и упоминание `_engineering_rules/`.
- [ ] Draft этого плана `D:\GitHub\_engineering_rules\plans\tech-stack-choices-catalog.md` существует.
- [ ] Harness `C:\Users\Zaxva\.claude\plans\ai-assistant-wiggly-axolotl.md` содержит только pointer.
- [ ] Cross-чек (через 2-3 следующих плана): меньше review-раундов «почему SQLite, а не Postgres», «почему NSSM, а не Docker», «почему локальные embeddings, а не API».

## Maintenance protocol (накопление по факту)

- Новый выбор подхода в плане → добавить пункт в `tech-stack-choices.md` тем же коммитом, что и сам план.
- Изменение существующего выбора → обновить пункт (не удалять — оставить как «было» в Trade-off секции, чтобы избежать loss-of-context).
- Удаление пункта только если technology полностью выведена из стека.
- Quarterly review — опционально, не обязательно для MVP.

## Открытые вопросы (для отдельных планов)

1. **Design-system migration** в `D:\GitHub\_engineering_rules\Design-system\` — отдельный план. Затронет: все потребители (`assistant-ui`, `Best_photo_ai`, `Stalking_offline`, `Verua_automation`), CDN/import paths, git history (через `git mv` или новый repo).
2. **Migration legacy drafts**: `D:\GitHub\AI-Assistant\docs\plans\_drafts\*.md` и аналогичные в других проектах — мигрировать в `_engineering_rules\plans\` или оставить как историю? Отдельный вопрос после стабилизации новой convention.
3. **Будут ли AI-агенты реально читать `tech-stack-choices.md`** — будем смотреть на следующих 2-3 планах; если нет — усилить формулировку bullet 5 (mandatory check вместо нейтрального when).
4. **Если каталог разрастётся на специализированные категории** (UI patterns, data patterns) — рассмотреть переход на `_engineering_rules/tech-stack/<category>.md` структуру.
