# Task status ledger

Maintained by `main-relay` (orchestrator). **Executors must not edit this file**; claiming and
progress are reported through the task branch commits (`chore(<ID>): claim task`,
`feat(<ID>): ...`) and the completion report, and recorded here by the orchestrator.

Status values:

- `todo` — assigned (branch/worktree created), unclaimed
- `in-progress` — claimed, in progress
- `in-review` — implementation done, awaiting orchestrator review
- `done` — merged into `main-relay`
- `blocked` — blocked (write the reason under "Evidence / notes")

| ID  | Task | Branch | Base | Dependencies | Status | Evidence / notes |
| --- | ---- | ------ | ---- | ------------ | ------ | ---------------- |
| —   | —    | —      | —    | —            | —      | —                |

## Update rules

- Orchestrator: set `todo` after creating a task branch; `in-progress` on claim receipt;
  `in-review` on completion receipt; `done` after merging into `main-relay`, with the merge
  commit as evidence.
- Only one branch works a task ID; a task already `in-progress` must not be started again.
- Delete a task branch only after its task is `done`.
