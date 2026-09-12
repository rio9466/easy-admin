# Orca Development Workflow

How this repository is built with Orca: the branch model, how tasks are dispatched, and how
a project is initialized.

Adapted from the reference implementation `rio9466/saas-website`, with this project's own
areas and branch names.

## 1. Model

- One repo, `easy-admin-main`, with two areas: `server/` (Go API service) and `server/admin/`
  (Vue management console).
- Orca manages git worktrees under `~/orca/workspaces/easy-admin-main/<branch>`; the repo root
  `~/orca/projects/easy-admin-main` stays on `main`.
- One worktree = one branch = one agent writing at a time.

### Branches

| Branch         | Role                                        | Base           | Merge target                     |
| -------------- | ------------------------------------------- | -------------- | -------------------------------- |
| `main`         | Frozen release branch                       | —              | only with explicit user approval |
| `main-relay`   | AI working / integration branch; orchestrator home | `main`   | —                                |
| `<task>`       | Ephemeral branch, one per task              | `main-relay`   | `main-relay`                     |

Rules:

- Never commit directly to `main`; never merge into `main` without the user's explicit
  approval.
- All merges land on `main-relay`.
- New sub-branches (task branches) are always created from the AI working branch
  `main-relay`, never from `main` or another task branch.

## 2. Roles

Three pi roles, each pinned to a branch:

| Role                | Home branch          | Owns                                                                                     |
| ------------------- | -------------------- | ---------------------------------------------------------------------------------------- |
| **Conversation pi** | `main` (main checkout) | Dialogue, workflow guidance, acceptance review, any git operation with user permission |
| **Orchestrator pi** | `main-relay`       | PRD / task documents, the status ledger, task worktrees, merges into `main-relay`        |
| **Executor pi**     | `<task>`             | Implementing exactly one task document; reporting evidence back                          |

### Conversation pi (`main`, main checkout)

This is the agent the user talks to. It does not build the product; it keeps the process
honest.

Responsibilities:

- **Dialogue and decisions** — clarify requirements, resolve ambiguity, capture product and
  process decisions. Surface tradeoffs instead of guessing.
- **Workflow guidance** — tell the user which branch to open, which task is ready (respecting
  dependencies), and hand over a ready-to-paste executor prompt.
- **Acceptance review** — independently verify executor evidence (commands + results) against
  the task's acceptance criteria and the PRD. Approve or send back with concrete feedback; no
  rubber-stamping.
- **Git operations** — with the user's explicit permission it may perform any git operation
  across all branches: create/delete branches, merge, rebase, tag, and so on. Advancing
  `main` (including merging `main-relay` into it) always requires that explicit approval
  first.
- **Cross-cutting approvals** — approve changes to the PRD, ADRs, architecture, the contract,
  and process/rule files (authored on `main-relay`).
- **Persistent memory** — keep a durable, local memory of project facts, conventions, and the
  user's preferences and habits, and read it at the start of every session (see
  *Persistent memory and user preferences* below).

It must not:

- Write business code or implement tasks.
- Advance `main` without the user's explicit approval.
- Do an executor's work on a task branch, or bypass `main-relay`.
- Duplicate the orchestrator's ledger bookkeeping; it verifies, the orchestrator records.

It may edit process/rule documents on `main-relay` (this file, root `AGENTS.md`), but not
business code.

#### Persistent memory and user preferences

The conversation pi keeps durable memory across sessions in a single, **local, git-ignored**
file at the repo root:

```
CONVERSATION_MEMORY.md
```

- It is listed in the root `.gitignore` and **must never be committed**.
- Because it is untracked, it lives in whichever checkout the conversation pi runs in (the
  `main` main checkout) and is not shared through git.

What to keep there:

- **User preferences and habits** — communication language, desired brevity, whether to
  propose a plan before acting, when merges/releases need explicit approval, naming and port
  conventions, review style, and any correction the user repeats.
- **Project facts** — branch model, release baseline and tags, service ports, datastores and
  connection details, external dependencies, environment quirks.
- **Conventions and pitfalls**.
- **Lightweight decisions and rationale** — decisions too small for an ADR.

When to read and write it:

- Read it at the start of a session to restore context and preferences.
- Update it the moment the user states or corrects a preference.
- Update it at checkpoints (decisions, releases, task close-out).
- Keep it concise: deduplicate, merge, and date entries — it is memory, not a log.

Boundaries:

- `CONVERSATION_MEMORY.md` is the conversation pi's own memory; the orchestrator and executors
  do not read or write it.
- It is not a product document: the PRD holds product requirements, `docs/tasks/STATUS.md`
  holds the task ledger, and this memory holds the assistant's durable context and the user
  profile.

### Orchestrator pi (`main-relay`)

Its scope is only `main-relay` and the executor/task branches. It turns the PRD into task
documents, maintains `docs/tasks/STATUS.md`, creates task worktrees from `main-relay`, does
the technical review, and merges task branches into `main-relay` with `--no-ff`.

It must never operate on `main` — no commits, merges, or branch changes there. Only the
conversation pi touches `main`, and only with the user's approval.

### Executor pi (`<task>`)

Implements exactly one task document on its task branch, verifies with the task's commands,
reports command + result, and never edits task docs, the contract, or other areas. It never
merges.

## 3. Task dispatch

1. The orchestrator writes a task document, e.g. `docs/tasks/<task>.md`, with goal, scope,
   out-of-scope, files, acceptance criteria, and verification steps.
2. The user switches to the target branch in Orca and opens a pi terminal in that worktree.
3. The user hands pi the task document (or its path).
4. The executor implements it, runs verification, and reports command + result.
5. The orchestrator reviews and merges into `main-relay`.

Optional CLI dispatch:

```bash
orca terminal create --worktree branch:<branch> --command "pi" --json
orca terminal send --terminal <handle> --text "Read docs/tasks/<task>.md and execute it." --enter --json
```

Rules:

- Hand pi the task document, not a paraphrase.
- One agent per worktree; never run two writers on one branch.
- Prefer a written task doc over a long inline prompt: it is reviewable and reusable.

### Claiming and status

A task is **assigned** when the orchestrator creates its dedicated branch/worktree from
`main-relay`. It is **claimed** when the executor starts work on that branch. Status is
recorded in two places:

- **Durable ledger**: `docs/tasks/STATUS.md` on `main-relay`, maintained by the
  orchestrator. Executors never edit it.
- **Branch evidence**: the task branch's commit history. The first commit is
  `chore(<ID>): claim task`; implementation commits use `feat(<ID>): ...` / `fix(<ID>): ...`.

Status values: `todo` (assigned, unclaimed), `in-progress` (claimed), `in-review`
(executor done, awaiting review), `done` (merged into `main-relay`), `blocked`.

Claiming steps:

1. Orchestrator: create the task worktree from `main-relay` and add its row to
   `docs/tasks/STATUS.md` as `todo`.
2. Executor: read the task doc, restate scope / assumptions / plan, and make the first commit
   `chore(<ID>): claim task`.
3. Executor: implement, verify with the task doc's commands, report command + result, and ask
   for review.
4. Orchestrator: set the ledger row to `in-review`, review, merge `--no-ff` into
   `main-relay`, then set it to `done` with the merge commit as evidence.

Rules:

- Only one branch works a task ID. If a task is already `in-progress`, do not start it again.
- Executors must not edit `docs/tasks/**` (task docs and the ledger); they report status and
  the orchestrator records it.
- Delete a task branch only after its task is `done`.

## 4. Documents and ownership

Different documents have different owners. A branch must refuse work that belongs to another
branch instead of doing it "just this once".

| Document                               | What it is                                          | Owner                      | Location                                   |
| -------------------------------------- | --------------------------------------------------- | -------------------------- | ------------------------------------------ |
| PRD                                    | What to build and why; product level, cross-cutting | `main-relay` (main line)   | `docs/prd/`                                |
| ADR                                    | Cross-cutting technical decision and its rationale  | `main-relay` (main line)   | `docs/adr/`                                |
| Architecture                           | Cross-cutting system design                         | `main-relay` (main line)   | `server/docs/architecture.md`              |
| API contract                           | The OpenAPI source of truth for the service         | `main-relay` (main line)   | `server/docs/openapi.yaml`                 |
| Area spec                              | How to build one feature inside one area            | that area's branch         | `server/docs/` or the area's own docs      |
| Task doc                               | One executable task for one agent                   | the executing branch       | `docs/tasks/`                              |
| `ORCA_WORKFLOW.md` / root `AGENTS.md`  | Process and rules                                   | `main-relay` (main line)   | repo root                                  |
| Area `AGENTS.md`                       | Coding rules for one area                           | that area's branch         | `server/AGENTS.md`, `server/admin/AGENTS.md` |

Strong rules:

- Cross-cutting documents (PRD, ADR, architecture, API contract, root `AGENTS.md`,
  `ORCA_WORKFLOW.md`) are owned by the main line: `main-relay`, later published to `main`.
- If you are on a task branch (anything other than `main-relay`), do NOT create
  or edit those documents. Stop and report: "This is cross-cutting documentation owned by
  `main-relay`. Open a pi terminal on `main-relay` (or on a docs branch cut from it) to do
  this."
- An area branch owns only its own area's specs and `AGENTS.md`. Never write the other area's.
- The orchestrator writes task documents on `main-relay`. An executor reads its assigned
  task doc and must not rewrite it without the orchestrator's approval.

Where to do what:

| Work                                                     | Branch to open a pi terminal on                          |
| -------------------------------------------------------- | -------------------------------------------------------- |
| PRD, ADRs, architecture, contract, process/rule changes  | `main-relay` (or a `docs/<topic>` branch cut from it)    |
| Go service feature work                                  | a service task branch cut from `main-relay`              |
| Management console feature work                          | an admin task branch cut from `main-relay`               |

## 5. Initialization

### Bootstrap of this repository

`easy-admin-main` was bootstrapped as plain vendored source:

- A clean copy of the `easy-admin` `main` working tree (Go service + Vue console), with no
  upstream git history, no `.git`, and no local/secret files.
- Governance docs added at the repo root, existing area `AGENTS.md` files reused, and the root
  `.gitignore` fixed so `server/docs/` is tracked.
- Baseline commit on `main`; `main-relay` cut from `main`.

### A. No scaffold given (greenfield)

Do it on a task branch cut from `main-relay`, commit, and prove it builds.

### B. Scaffold given

Vendor it as plain source. Never keep its git history, and never touch the source repo.

1. Clone upstream into a temp dir (never clone into the repo).
2. Copy into the target directory, excluding `.git`.
3. Keep every `.gitignore`; remove only `.git`.
4. Stage, then force-add any upstream-tracked files that this repo's ignore rules would hide.
5. Commit on the owning branch and merge into `main-relay`. Leave the original scaffold repo
   untouched.

## 6. Branch constraints (AGENTS.md)

`AGENTS.md` files are layered; the nearest one wins.

| File                             | Constrains                                             |
| -------------------------------- | ------------------------------------------------------ |
| `AGENTS.md` (root)               | all branches: coding guidelines, git rules, branch scope |
| `server/AGENTS.md`               | Go service (`server/`, excluding `server/admin/`)       |
| `server/admin/AGENTS.md`         | management console (`server/admin/`)                    |

Core cross-branch constraints:

- A branch edits only its own area. `server/` (Go service) and `server/admin/` (Vue console)
  are separate areas: a service branch must not modify the console and vice versa, unless the
  task document records an explicit exception.
- Never reintroduce the original `easy-admin` git history or a submodule.
- Keep `main` untouched.

## 7. Merge and review

1. The executor runs the acceptance checks in the task doc and reports the exact command and
   result.
2. Merge `--no-ff` into `main-relay`; do not fast-forward away the task history.
3. After a merge, `main-relay` must still build/validate for the areas it touched.
4. `main` is updated only with explicit user approval.

## 8. Environment notes

- The management console's typed API contract is generated from `server/docs/openapi.yaml`
  with `pnpm generate:api`; never edit `types/api.generated.ts` by hand.
- `server/configs/config.local.toml` and other local/secret files are git-ignored; only
  `config.example.toml` is committed.
- Orca injects pi extensions for status reporting, the terminal-title spinner, and editor
  prefill. They are managed by Orca; do not edit them by hand.

## 9. Task document template

```markdown
# Task: <name>

## Goal

<one paragraph>

## Scope

- ...

## Out of scope

- ...

## Files / areas

- ...

## Acceptance criteria

- [ ] ...

## How to verify

- <exact commands and expected result>

## Branch / base

- branch: <task>
- base: main-relay
```
