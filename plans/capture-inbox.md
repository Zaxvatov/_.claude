# План: Capture / Inbox — интейк и посев дерева смысла

**Статус:** ЧЕРНОВИК-набросок (заготовка; прорабатывать после `goal-intent-tree` W0). Задачи PM-MCP не созданы.
**Дата:** 2026-06-04
**Зависит от:** `goal-intent-tree` (intent tree должен существовать, чтобы было куда крепить узлы).
**Затронутые подсистемы:** `assistant-ui` (Inbox, capture, модерация), `pm-mcp-server` (ingest, `personal`, опц. `propose_goal`), `gateway` (опц. scope), `ai-memory` (capture фактов о себе).
**Связанные ADR:** ADR-0001 (scope tree, propose-flow), ADR-0004 (idea contract). Возможен новый ADR на capture-таксономию.

---

## 1. Зачем отдельный план

По ревью Codex (п.8) интейк вынесен из intent-tree MVP, чтобы не раздувать ядро дерева. Здесь живёт всё, что «роняет узлы» на дерево: захват, разбор, модерация, посев из реальности (Todoist), дом личных задач. Корни идеи — в первом архитектурном обсуждении (Universal Capture = «3-в-1» router+classifier+knowledge routing).

## 2. Двунаправленная раскрутка (agent-skill)

- **Снизу вверх (induction):** куча задач → кластеризация по «зачем» → предложить родительские цели/подцели (посев из реальности, ответ «зачем эта задача существует»).
- **Сверху вниз (deduction):** цель → декомпозиция в подцели/задачи.
- Data-model (рекурсивный `parent_goal_id` из `goal-intent-tree`) уже поддерживает обе — это требование к **skill**, не к схеме.

## 3. Интейк — source-agnostic, pluggable

Источник входящего = сменный адаптер за общим Inbox: Todoist, chat-агент, ручной capture. Канон — PM-MCP (work items) + дерево (goals.yaml). Адаптер добавляется/убирается без влияния на дерево.

## 4. Посев из реальности — 5 шагов

1. **Сбор** входящего в одну кучу (Todoist-домен + captured).
2. **Раскрутка вверх** агентом → черновик дерева (рассуждение — любой агент, в т.ч. ChatGPT; запись — локально).
3. **Модерация** пользователем (accept/merge/rename/reject, дописать `why`, задать `horizon`) — вариант proposals-вкладки `/memory` (#737) для целей/задач, либо отдельная `/inbox`.
4. **Консолидация + приоритеты:** одобренное → goals.yaml (`parent_goal_id`) + `strategic_weight`/`horizon`; значимые Todoist-задачи → канонические PM-MCP work items + `goal_ids` (`todoist→SQLite ingest`).
5. **Спуск вниз:** `recommend_next_global` по дереву, два потока — системные (repo `project_path`, Codex/Claude) и персональные (проект `personal`, опц. зеркало в Todoist).

## 5. Куски и решения

- **`todoist→SQLite ingest`:** promote выбранной Todoist-задачи в каноническую PM-MCP-задачу + `goal_ids`. Todoist остаётся read-only зеркалом; ingest однонаправлен.
- **Дом `personal` (Codex п.9 — решить конкретно):**
  - **(A) реальный `project_path` «personal» (рекоменд.):** work items скоупятся штатно, получают `goal_ids`, полный статус-lifecycle; по ADR-0003 нумерация глобальная; регистрация через первый task-anchor или прямой каталог под `PM_MCP_PROJECTS_ROOT`.
  - (B) виртуальный домен (как `todoist`/`finance`): read-mostly mirror — НЕ подходит для задач с полным write-lifecycle.
  - (C) отдельный adapter — это внешняя система, не наш случай.
- **Todoist — переходный сателлит, не опора.** Ничто не зависит от Todoist; вероятный end-state — chat-driven постановка (= Universal Capture), ретайр = убрать один адаптер (migration-discipline).
- **Gateway `propose_goal` (Q10):** сейчас `pm.propose` = только `propose_task` (`gateway/scope_policy.py`); ChatGPT не пишет цели. Варианты: goal-induction в локальном чате (loopback write после модерации) ИЛИ позже добавить `propose_goal` в scope tree (новый ADR + ревизия `scope_policy.py`).
- **Universal Capture / Capture decision:** «3-в-1» способность (router + classifier + knowledge routing) — классифицирует ввод и подвешивает на нужную ветку дерева; неоднозначное → proposal/Inbox для модерации человеком.

## 6. Открытые вопросы

- Дом `personal`: A/B/C (рекоменд. A — реальный `project_path`).
- `propose_goal` в Gateway: добавлять роут или держать goal-writes локально.
- Capture-таксономия: нужен ли отдельный ADR (controlled `metadata.type` словарь) — связано с Universal Capture из первого обсуждения.
- Модерация целей/задач: расширять `/memory` proposals или отдельная вкладка `/inbox`.

## 7. Вне scope

- Ядро intent tree (recursive goals, `get_goal_tree`, scoring, визуализация) — в `goal-intent-tree`.
- Relationship Graph, домены `contact` / `learning-item` — позже, по реальной боли.

## Процедурные заметки (по central-plan-workflow)

- Прорабатывать после `goal-intent-tree` W0 (контракт дерева должен быть зафиксирован первым).
- До технологических решений сверяться с `D:\GitHub\_engineering_rules\tech-stack-choices.md`.
- Задачи PM-MCP — после согласования, через `pm-mcp-task-flow`, с `link_task_dependency` на волны `goal-intent-tree`.
- Важный итог согласования — через `ai-memory-capture` (`project=portfolio`).
