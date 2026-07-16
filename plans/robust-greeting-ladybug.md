# Roadmap task card: draggable popover, subtask consistency, relation links

## Context

On `/roadmap` (`http://127.0.0.1:8000/roadmap`) the task details popover has three problems the user hit on card `#957 Консалтинг`:

1. **Popover overflows off-screen and can't be moved.** `.roadmap-details-popover` uses `max-height: none; overflow: visible;` (frontend/src/styles.css:386), so a card with many subtasks grows past the bottom of the viewport and the lower content (subtask list) is unreachable. The user wants to drag the card around the screen (in-place, not a separate window).

2. **Subtask data disagrees between two views.** The roadmap popover lists many subtasks for `#957`, but the `/work-item/957` page shows "Подзадачи —" (none). Root cause confirmed via PM-MCP: `get_goal_tree(root_goal_id=#957)` returns subtasks in `linked_work_items`, **not** in `children`. The roadmap builder scans all work items by `parent_id` (`roadmap.py:_work_item_child_relations` + `_populate_child_relations`) and finds them; the work-item page builder (`dashboard.py:_work_item_relations`) only reads `root.children`, which is empty → shows none.

3. **Relation numbers are plain text.** In the roadmap popover, subtask/parent ids are rendered as non-clickable `<strong>` (`RelationField`). The header task number is already a clickable link that opens the work-item preview overlay; relations should behave the same.

Outcome: the popover is draggable so off-screen content is reachable, the two views agree on subtasks, and subtask/parent ids are clickable links opening the existing work-item preview overlay.

## Changes

### Issue 2 — backend subtask consistency (smallest, do first)
File: `assistant-ui/app/dashboard.py`, function `_work_item_relations` (line ~208).
- After collecting `children` from `root["children"]`, also iterate `root.get("linked_work_items")` (list of work-item dicts) and append each as a child relation `{id, title}`, de-duplicating by `id` against existing children. Reuse the same id/title extraction style already used for `children` (`id` or `slug`; `title` or `text`).
- This makes `/work-item/{id}` show the same subtasks the roadmap shows. No template change needed — `work_item.html` already loops `work_item.relations.children`.
- Add/extend a unit test in `assistant-ui/tests/` (the file covering `work_item_detail_model`) feeding a `relation_tree_payload` with `linked_work_items` and asserting they appear in `relations.children`.

### Issue 3 — clickable relation links (frontend island)
File: `assistant-ui/frontend/src/main.tsx`.
- Add a helper `workItemUrlForId(id: string): string | null` mirroring `workItemUrl` (line 430): return `/work-item/${encodeURIComponent(id)}` only when `id` starts with `#`, else `null`.
- Refactor `displayRelationItems` (line 447) into a `dedupeRelationItems(relations): RoadmapRelation[]` that returns the deduped `{id,title}` objects (keep the existing prefer-`#`-id dedupe logic); derive the display string from it where still needed.
- Change `RelationField` (line 1123) to accept an `onOpen: (url: string) => void` prop. For each relation render: if `workItemUrlForId(relation.id)` is non-null, render an `<a className="work-item-preview-link">` whose `onClick` calls `event.preventDefault()` + `onOpen(url)` (same pattern as the header link, main.tsx:1983-1994), showing `id` then `: title`; otherwise render the plain `<strong>` text as today.
- Pass `onOpen={setWorkItemPreviewUrl}` to both `RelationField` instances (Подзадачи / Родительские задачи, lines 2010-2011).
- Existing `.work-item-preview-link` CSS already styles these; the existing `.roadmap-relation-list strong` 2-line clamp stays for non-link items.

### Issue 1 — draggable popover (frontend island)
File: `assistant-ui/frontend/src/main.tsx` (+ small CSS in `styles.css`).
- Add state `const [popoverOffset, setPopoverOffset] = useState({ x: 0, y: 0 })` and a `popoverDragRef` for in-flight drag (start pointer + start offset).
- Reset `popoverOffset` to `{0,0}` whenever the popover target changes (effect keyed on the popover item id) so each newly opened card starts at its computed anchor.
- Add pointer handlers on the details popover `<header>` (drag handle): `onPointerDown` records start and sets `setPointerCapture`; window/`onPointerMove` updates offset by delta; `onPointerUp` ends. Ignore drags that start on the close `md-icon-button` (check `event.target.closest`).
- Apply the offset by composing it with `popoverStyle(popover.x, popover.y)`: add `left += offset.x`, `top += offset.y` (drop the existing viewport clamp in `popoverStyle` for the details popover, or let the offset push beyond it, so the user can pull content fully into view). Keep the create popover unchanged.
- CSS: header gets `cursor: move; user-select: none;` and the close button keeps `cursor: pointer`. Add to `styles.css` near `.roadmap-popover header` (line 391).
- Keep it pointer-events based (works for mouse + touch) and self-contained inside the island, per ADR-0013 / AGENTS frontend invariants (only viewport/UI draft state in React).

## Build & verification

From `assistant-ui/frontend/`:
- `npm run build` (rebuilds the gitignored bundle under `app/static/roadmap/`).

From `assistant-ui/`:
- `uv run ruff check .`
- `uv run pytest` (incl. the new `_work_item_relations` test).
- `uv run python scripts/verify_frontend.py --page roadmap` (desktop + mobile screenshots; binds ephemeral port, never 8000).

Manual check against the running service:
- Open `/roadmap`, click `#957`: subtask/parent ids are links; clicking one opens the work-item preview overlay; drag the popover header to pull the long subtask list into view.
- Open `/work-item/957`: "Подзадачи" now lists the same subtasks shown on the roadmap.

## Notes
- No new dependencies or tech-stack additions (existing React/Vite island + FastAPI/Jinja). No backend MCP contract change — purely a read-model fix in Assistant-UI.
- After completion, record a short AI-memory `change` entry (consistent with prior roadmap popover entries #1040/#1038) linking the touched files.
