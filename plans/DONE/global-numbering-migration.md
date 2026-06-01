# Global Numbering Migration Draft

Canonical ADR: [ADR 0003: Global PM-MCP Task Numbering](../../../../docs/adrs/0003-global-task-numbering.md)

Status: Draft for a separate planning session.

## Scope

Plan the PM-MCP migration from per-project task counters to one global task
counter. Do not implement migration code in this draft.

## Required Inputs

- Current PM-MCP SQLite schema and migrations.
- Storage code paths for task creation, lookup, history, dependencies, and
  metadata.
- Assistant-UI and other clients that display or parse task IDs.
- AI-memory links and historical task references.
- Backup and rollback procedure for SQLite with WAL accounted for.

## Draft Checklist

- [ ] Inventory schema and write paths.
- [ ] Choose ID display format.
- [ ] Choose `metadata.legacy_id` structure.
- [ ] Produce dry-run mapping artifact.
- [ ] Define rollback path.
- [ ] Define documentation rewrite targets.
- [ ] Define verification queries and tests.
