# Orca Development Workflow

How this repository is built with Orca: the branch model, how tasks are dispatched, and how
a project is initialized.

Adapted from the reference implementation `rio9466/saas-website`, with this project's own
areas and branch names.

## 1. Model

- One repo, `easy-admin-main`, with two areas: `server/` (Go API service) and `server/admin/`
  (Vue management console). **Areas are defined by your project; these two are this project's.**
- To adopt this workflow in a new project, see `docs/adopting-orca-workflow.md` (reference
  implementation: <https://github.com/rio9466/saas-website>).
- Orca manages git worktrees under `~/orca/workspaces/easy-admin-main/<branch>`; the repo root
  `~/orca/projects/easy-admin-main` stays on `main`.
- One worktree = one branch = one agent writing at a time.

### Branches

| Branch         | Role                                        | Base           | Merge target                     |
| -------------- | ------------------------------------------- | -------------- | -------------------------------- |
| `main`         | Frozen release branch                       | —              | only with explicit user approval |
| `main-relay`   | AI working / integration branch (managed by the conversation pi) | `main` | —                   |
| `<task>`       | Ephemeral branch, one per task              | `main-relay`   | `main-relay`                     |

Rules:

- Never commit directly to `main`; never merge into `main` without the user's explicit
  approval.
- All merges land on `main-relay`.
- New sub-branches (task branches) are always created from the AI working branch
  `main-relay`, never from `main` or another task branch.

## 2. Roles

Two pi roles. The former **orchestrator/planning pi is merged into the conversation pi** — there
is no separate relay agent. The long-lived branches stay: `main` (release baseline) and
`main-relay` (integration branch, managed by the conversation pi).

| Role                | Home branch            | Owns                                                                                                                       |
| ------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Conversation pi** | `main` (main checkout) | Dialogue, planning (PRD/contract/task docs), project setup, worktree dispatch, acceptance review, git merges and releases |
| **Executor pi**     | `<task>`               | Implementing exactly one task document; reporting evidence back. Never merges.                                             |

### Conversation pi (`main`, main checkout)

This is the agent the user talks to. It does not build the product; it keeps the process
honest.

Responsibilities:

- **Dialogue and decisions** — clarify requirements, resolve ambiguity, capture product and
  process decisions. Surface tradeoffs instead of guessing.
- **Planning** — own the PRD, the API contract, ADRs, and the task documents, and keep
  `docs/tasks/STATUS.md`. These are authored on `main-relay`.
- **Workflow guidance** — tell the user which branch to open, which task is ready (respecting
  dependencies), and hand over a ready-to-paste executor prompt.
- **Project setup** — initialize new projects and adopt existing projects into this workflow.
  Ask the user first whether it is a new or an existing project, then follow
  `docs/adopting-orca-workflow.md` (see *Project setup* below).
- **Worktree dispatch** — create each task's branch/worktree from `main-relay` with the Orca
  CLI (see *Worktree and Orca basics* below).
- **Acceptance review** — independently verify executor evidence (commands + results) against
  the task's acceptance criteria and the PRD. Approve or send back with concrete feedback; no
  rubber-stamping.
- **Git operations** — with the user's explicit permission it may perform any git operation
  across all branches: create/delete branches, merge, rebase, tag, and so on. Advancing
  `main` (including merging `main-relay` into it) always requires that explicit approval
  first.
- **Cross-cutting approvals** — approve changes to the PRD, ADRs, architecture, the contract,
  and process/rule files (authored on `main-relay`).
- **Persistent memory** — keep a durable memory of project facts, conventions, and the
  user's preferences and habits (an Obsidian note when Obsidian is installed, otherwise a
  git-ignored local file), and read it at the start of every session (see
  *Persistent memory and user preferences* below).

It must not:

- Write business code or implement tasks.
- Advance `main` without the user's explicit approval.
- Do an executor's work on a task branch.

It may edit process/rule and planning documents on `main-relay` (this file, root `AGENTS.md`,
the PRD/contract, `docs/tasks/**`), but not business code.

#### Project setup (new or existing)

When the user asks to start or onboard a project, **first ask whether it is a new project or an
existing project being adopted into the workflow**, then follow
`docs/adopting-orca-workflow.md`.

- **New project** — scaffold the governance docs (`AGENTS.md`, `ORCA_WORKFLOW.md`, `docs/`
  skeleton), create the branches (`main`, `main-relay`), set up the memory note (the Obsidian
  vault if installed, otherwise a git-ignored `CONVERSATION_MEMORY.md`), then plan the first
  tasks.
- **Existing project** — audit the repo, add or adapt the governance docs, create `main-relay`
  from the current default branch, tag a baseline, and record facts in memory.

Ask the user for at least: project name / remote / default branch name, the areas and their
stacks, ports and environment conventions, and the starting version.

#### Worktree and Orca basics

One task = one Orca worktree cut from `main-relay`. Commands the conversation pi needs:

```bash
# create a task worktree (branch) from main-relay
orca worktree create --repo id:<repoId> --name <task> --base-branch main-relay --no-parent --json
orca worktree list --json
orca worktree rm --worktree branch:<task> --force --json
orca worktree set --worktree branch:<task> --comment "claimed <ID>" --workspace-status in-progress
# run and prompt an executor in that worktree
orca terminal create --worktree branch:<task> --command "pi" --json
orca terminal send --terminal <handle> --text "<task brief>" --enter --json
```

- Prefer `--json`; selectors: `id:<repoId>::<path>`, `branch:<name>`, `path:<abs>`, `active`.
- Orca auto-discovers external `git worktree`s. If `/usr/local/bin/orca` is a restricted
  symlink, invoke the real entrypoint:
  `ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca /Applications/Orca.app/Contents/Resources/app.asar.unpacked/out/cli/index.js …`
- **Two traps verified in this repo (2026-09-12)** — the commands above do not work as written
  without these:
  - Orca's `settings.branchPrefix` defaults to `git-username`, so `orca worktree create --name
    <task>` creates `<git-owner>/<task>` (e.g. `rio9466/be-01`) instead of `<task>`, and every
    `branch:<task>` selector above then fails to match. Set Orca's branch prefix to `none`, or
    create the worktree in the Orca GUI with an explicit branch name, or `git checkout <task>`
    inside the new worktree and delete the leftover prefixed branch.
  - A worktree created with plain `git worktree add` is *external*, and this repo has
    `externalWorktreeVisibility: "hide"`, so it never appears in Orca's sidebar. Create task
    worktrees through Orca. Always assert the result — in `git worktree list` the bracket must
    read exactly `[<task>]`.

#### Persistent memory and user preferences

The conversation pi keeps durable memory across sessions. **If Obsidian is installed on the
user's machine, the memory lives in an Obsidian note; otherwise it falls back to a local,
git-ignored file.**

Detection and storage:

1. Detect Obsidian — macOS: `/Applications/Obsidian.app` and/or
   `~/Library/Application Support/obsidian/obsidian.json` (lists vaults; the entry with
   `open: true` is the user's active vault).
2. In the active vault, find `CONVERSATION_MEMORY.md` — **create it if missing; merge into the
   matching section if present** (never overwrite other sections). Organize it with Obsidian
   `[[wikilinks]]` so it connects in the **graph view**: a hub note linking section notes such
   as `[[用户偏好]]`, `[[项目事实]]`, `[[约定与坑]]`, `[[轻量决策]]`, each linking back to the hub.
3. All future memory writes go to that note. It is the source of truth.
4. One vault can hold several projects. Keep this project's notes together in a folder named
   after the project (`easy-admin-main/`): the hub plus its section notes, linked with bare
   `[[…]]` links (Obsidian resolves them to the folder siblings). Never write this project's
   facts into another project's notes — merge only the sections that are genuinely user-wide.

Fallback (no Obsidian): keep the git-ignored `CONVERSATION_MEMORY.md` at the repo root (listed
in `.gitignore`, never committed).

What to keep there:

- **User preferences and habits** — communication language, desired brevity, whether to
  propose a plan before acting, when merges/releases need explicit approval, naming and port
  conventions, review style, and any correction the user repeats.
- **Project facts** — branch model, release baseline and tags, service ports, datastores and
  connection details, external dependencies, environment quirks.
- **Conventions and pitfalls** — e.g. pnpm 12 native-binary install, the root `.gitignore`
  `docs/` trap that hid `server/docs/`, `auth.trusted_origins` and the `30001` forbidden error,
  Orca's `branchPrefix` creating `<owner>/<task>` branches, external (non-Orca) worktrees being
  hidden in the sidebar.
- **Lightweight decisions and rationale** — decisions too small for an ADR.

When to read and write it:

- Read it at the start of a session to restore context and preferences.
- Update it the moment the user states or corrects a preference.
- Update it at checkpoints (decisions, releases, task close-out).
- Keep it concise: deduplicate, merge, and date entries — it is memory, not a log.

Boundaries:

- The memory (the Obsidian note, or the fallback file) is the conversation pi's own; executors
  do not read or write it.
- It is not a product document: the PRD holds product requirements, `docs/tasks/STATUS.md`
  holds the task ledger, and this memory holds the assistant's durable context and the user
  profile.

### Executor pi (`<task>`)

Implements exactly one task document on its task branch, verifies with the task's commands,
reports command + result, and never edits task docs, the contract, or other areas.

**It must never merge** — not into `main-relay`, not into `main`. It reports evidence and lets
the conversation pi review and merge.

## 3. Task dispatch

1. The conversation pi writes a task document, e.g. `docs/tasks/<task>.md`, with goal, scope,
   out-of-scope, files, acceptance criteria, and verification steps.
2. The conversation pi creates the task's branch/worktree from `main-relay` with the Orca CLI
   (see *Worktree and Orca basics* above) and adds its row to `docs/tasks/STATUS.md` as `todo`.
   The user does not run git or Orca commands by hand for this.
3. The conversation pi starts an executor terminal in that worktree and hands pi the task
   document (or its path).
4. The executor implements it, runs verification, and reports command + result.
5. The conversation pi reviews and merges into `main-relay`.

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

A task is **assigned** when the conversation pi creates its dedicated branch/worktree from
`main-relay`. It is **claimed** when the executor starts work on that branch. Status is
recorded in two places:

- **Durable ledger**: `docs/tasks/STATUS.md` on `main-relay`, maintained by the conversation
  pi. Executors never edit it.
- **Branch evidence**: the task branch's commit history. The first commit is
  `chore(<ID>): claim task`; implementation commits use `feat(<ID>): ...` / `fix(<ID>): ...`.

Status values: `todo` (assigned, unclaimed), `in-progress` (claimed), `in-review`
(executor done, awaiting review), `done` (merged into `main-relay`), `blocked`.

Claiming steps:

1. Conversation pi: create the task worktree from `main-relay` and add its row to
   `docs/tasks/STATUS.md` as `todo`.
2. Executor: read the task doc, restate scope / assumptions / plan, and make the first commit
   `chore(<ID>): claim task`.
3. Executor: implement, verify with the task doc's commands, report command + result, and ask
   for review.
4. Conversation pi: set the ledger row to `in-review`, review, merge `--no-ff` into
   `main-relay`, then set it to `done` with the merge commit as evidence.

Rules:

- Only one branch works a task ID. If a task is already `in-progress`, do not start it again.
- Executors must not edit `docs/tasks/**` (task docs and the ledger); they report status and
  the conversation pi records it.
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
- The conversation pi writes task documents on `main-relay`. An executor reads its assigned
  task doc and must not rewrite it without the conversation pi's approval.

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
