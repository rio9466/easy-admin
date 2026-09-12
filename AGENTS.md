# When coding, follow these guidelines:

- Think before coding. State assumptions when the task is ambiguous, and ask only if a reasonable assumption would be risky.
- Prefer the simplest implementation that fully solves the request. Do not add speculative features, abstractions, configurability, or broad error handling unless needed.
- Make surgical changes. Touch only files and lines directly related to the request. Match the existing code style. Do not refactor unrelated code.
- If a change creates unused imports, variables, functions, or files, clean up only those introduced by the change.
- Define verifiable success criteria for non-trivial work. Prefer reproducing bugs with tests, then fixing them. Run relevant tests or explain why they could not be run.
- Surface tradeoffs and uncertainty clearly. Do not hide confusion or silently pick among materially different interpretations.

# Roles (who does what)

- **Conversation pi** — runs on `main` (this checkout). May perform any git operation across all branches with the user's explicit permission; advancing `main` always needs that approval. Never writes business code. May edit process/rule docs on `main-relay`.
- **Orchestrator pi** — runs on `main-relay`. Scope is only `main-relay` and the task branches: writes the PRD and task docs, maintains `docs/tasks/STATUS.md`, creates task branches, and merges task branches into `main-relay`. Must never operate on `main`.
- **Executor pi** — runs on a `<task>` branch. Implements one task, reports command + result, never merges.

See `ORCA_WORKFLOW.md` §2 for the full role contract.

# Document ownership (strong rules)

- Cross-cutting documents are owned by the main line (`main-relay`, later published to `main`): the PRD (`docs/prd/`), ADRs (`docs/adr/`), architecture, the root `AGENTS.md`, and `ORCA_WORKFLOW.md`.
- If you are on any task branch (anything other than `main-relay`), do NOT create or edit those documents. Stop and tell the user: "This is cross-cutting documentation owned by main-relay; open a pi terminal on main-relay (or a docs branch cut from it)."
- An area branch owns only its own area's specs and `AGENTS.md` (`server/AGENTS.md` or `server/admin/AGENTS.md`). Never write the other area's docs.
- The orchestrator writes task documents on `main-relay`; an executor reads its assigned task doc and must not rewrite it without approval.
- Refuse these requests even when asked to do them "just this once". Redirect to the correct branch instead.

# Git workflow

- `main-relay` is the AI integration branch and the default base for all AI work. Do not commit directly to `main`.
- Never merge into `main` unless the user explicitly agrees first.
- All branch merges happen on `main-relay`. Branch feature work off `main-relay` and merge back into `main-relay`.
- There are no long-lived area branches. Every piece of work is an ephemeral task branch created from the AI working branch `main-relay` (e.g. `be-bootstrap`, `admin-rbac`), never from `main` or from another task branch.
- Use Orca-managed git worktrees. `main-relay` and every task branch are checked out under `~/orca/workspaces/easy-admin-main/`; the repo root `~/orca/projects/easy-admin-main` stays on `main`.

### Branch constraints

- Each branch edits only its own area. The areas for this project are `server/` (Go API service) and `server/admin/` (Vue management console). Cross-area changes need separate task branches or an explicit exception in the task document.
- New task branches are cut from `main-relay` and merged back into `main-relay`; `main` stays untouched unless the user explicitly approves.
- Never reintroduce the original `easy-admin` repo's git history, add it as a submodule, or re-add the original repo as a remote beyond `origin`.
- Area rules live in `server/AGENTS.md` (Go service, excluding `server/admin/`) and `server/admin/AGENTS.md` (management console); the nearest layered `AGENTS.md` wins. The typed API contract is generated from `server/docs/openapi.yaml`.
- Full workflow, roles, task dispatch, and initialization paths: see `ORCA_WORKFLOW.md`.

# Task claiming

- Task documents live in `docs/tasks/`; their status ledger is `docs/tasks/STATUS.md`, owned by `main-relay`. Executors must not edit `docs/tasks/**`.
- Work only on the branch named in your assigned task doc, cut from `main-relay`. Never start a task that is already `in-progress` in the ledger.
- Claim a task by restating scope / assumptions / plan and making your first commit `chore(<ID>): claim task`. Use `feat(<ID>): ...` / `fix(<ID>): ...` for implementation commits.
- Do not merge your task branch. Report command + result and let the orchestrator review and merge into `main-relay`.
