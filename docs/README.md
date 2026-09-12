# Docs

Ownership rules live in the root `AGENTS.md` and `ORCA_WORKFLOW.md`. Summary:

| Path          | What it is                                          | Owner                      |
| ------------- | --------------------------------------------------- | -------------------------- |
| `docs/prd/`   | Product requirements (what/why, cross-cutting)      | `main-relay` (main line)   |
| `docs/api/`   | Frontend ↔ service API contract notes (cross-cutting) | `main-relay` (main line) |
| `docs/adr/`   | Architecture decision records                       | `main-relay` (main line)   |
| `docs/specs/` | Per-area feature specs                              | the area's branch          |
| `docs/tasks/` | One executable task per agent                       | the task's branch          |

Cross-cutting documents (PRD, ADR, architecture) must be written from a `main-relay`
terminal. Task branches must refuse to create or edit them and redirect to `main-relay`.

The service's own documentation stays where it already is:

- `server/docs/openapi.yaml` — the API source of truth.
- `server/docs/architecture.md`, `server/docs/data-stores.md`,
  `server/docs/coding-standards.md` — service docs.

Current documents:

- [`tasks/README.md`](tasks/README.md) — task index and execution order (to be filled by the
  orchestrator).
- [`tasks/STATUS.md`](tasks/STATUS.md) — task status ledger.
