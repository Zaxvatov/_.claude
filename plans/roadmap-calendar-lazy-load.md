# Roadmap calendar lazy-load

PM-MCP: #1000
Status: completed
Project: `D:\GitHub\AI-Assistant\assistant-ui`
AI-memory: #1785
Completed: 2026-06-18

## Context

`/roadmap` is a React/Vite island backed by Assistant-UI `/api/roadmap`.
Calendar events come from PM-MCP `list_calendar_events(time_min, time_max)`.
The current page accepts a bounded API window, but horizontal movement to the
past does not reliably extend the rendered calendar columns, dates, or events.

Runtime note: the requested `EnterPlanMode` tool is not available in this
Codex session, so this central plan was created once in normal mode after the
required project onboarding, memory recall, and tech-stack check.

## Goal

Make roadmap data load from a today-anchored initial window and extend the
window in the direction where the user moves, matching the lazy-loading idea
already used by the budget journal.

## Constraints

- Keep PM-MCP as the source of truth for calendar events and task/goal data.
- Do not introduce a new frontend framework or client-side domain store.
- Keep the existing ADR-0013 React/Vite island boundary.
- Keep calendar privacy shaping in PM-MCP and Assistant-UI pass-through.
- Do not touch unrelated existing changes in `ai-memory/` or central plans.

## Tech Stack Check

Read `D:\GitHub\_engineering_rules\tech-stack-choices.md`.
Applicable bricks:

- #6 Web UI: FastAPI + Jinja2 + Material Web, with ADR-0013 React/Vite island
  exception for `/roadmap`.
- #25 Google Calendar read-only sync: PM-MCP owns calendar sync and event reads.
- #27 Python tests: pytest.

No new technology or org-level brick is required.

## Implementation Plan

1. Inspect current `/api/roadmap` range parameters, React range state, cache
   keys, and viewport movement handling.
2. Replace fixed full-range calendar rendering with a today-anchored loaded
   range per view mode.
3. Add directional expansion when the React Flow viewport approaches the left
   or right edge of the loaded calendar range.
4. Merge newly fetched payloads without losing filters, collapsed state,
   selection, calendar source visibility, or write flows.
5. Add regression coverage for bounded API calls and client lazy-load behavior.
6. Rebuild the roadmap bundle and run assistant-ui lint, tests, and frontend
   verification.

## Acceptance Criteria

- Initial `/roadmap` load requests a bounded window around today.
- Scrolling left extends the loaded range into earlier dates and renders older
  period columns and calendar events.
- Scrolling right extends the range into later dates.
- Repeated movement does not refetch an already cached window unnecessarily.
- Existing task/goal edits, drag-reschedule, filters, and calendar source
  toggles keep working.
- Verification results are recorded before closing PM-MCP #1000.

## Implementation Result

- `/api/roadmap` without explicit `time_min/time_max` now uses `today..tomorrow`
  instead of the broad `2026-01-01..2040-12-31` calendar window.
- React roadmap starts from the current period only and expands the loaded
  range left or right after user movement.
- Past expansion compensates the React Flow viewport so prepended columns do
  not visually jump the current area.
- Horizontal wheel on the canvas also triggers directional expansion, covering
  scroll gestures where React Flow movement callbacks are not emitted.
- The client fetches the full expanded window after each boundary extension
  instead of stitching independent chunks, keeping graph filtering and edges
  consistent with the existing view-model contract.

## Verification

- `npm run build` in `assistant-ui/frontend`.
- `uv --cache-dir .uv-cache run ruff check .` in `assistant-ui`.
- `uv --cache-dir .uv-cache run pytest tests/test_api_endpoints.py -k roadmap -p no:cacheprovider`.
- `ASSISTANT_CONVERSATION_STORE_PATH=D:\GitHub\AI-Assistant\assistant-ui\logs\pytest-codex-1000-final.sqlite3 uv --cache-dir .uv-cache run pytest -p no:cacheprovider` (176 passed).
- `uv --cache-dir .uv-cache run python scripts/verify_frontend.py --page roadmap`: desktop/mobile start with today window and lazy-load past range `2026-06-18 -> 2026-06-04`.
- `git diff --check` for touched files.

## Retrospective Before Close

| Axis | Verdict | Note |
| --- | --- | --- |
| tech-stack-choices.md | no-change | Existing bricks #6, #25, #27 cover the work; no new technology choice. |
| Design-system | no-change | No shared tokens, components, or Design-system assets changed. |
| Skills | no-change | Existing `frontend-verification` helper was updated locally for this roadmap scenario; no reusable global skill change needed. |
| Hooks | no-change | No deterministic guard/hook emerged beyond existing tests and verification helper. |
