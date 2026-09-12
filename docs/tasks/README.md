# Task index

Generated and maintained by `main-relay` from the PRD. The status ledger is
[`STATUS.md`](STATUS.md); executors must not edit this directory.

## Contract first

The API contract is `server/docs/openapi.yaml`. Executors **must not** change
request/response shapes on their own; if the contract is missing something, stop and report to
`main-relay` so it can update the contract.

## Execution order

| ID | Document | Branch | Base | Dependencies |
| -- | -------- | ------ | ---- | ------------ |
| admin-brand-ux | [`admin-brand-ux.md`](admin-brand-ux.md) | `admin-brand-ux` | `main-relay` | — |

## Common acceptance requirements (all tasks)

- Change only the directories listed in the task's "Files / areas"; report cross-area changes
  before making them.
- Service: `cd server && make check` (fmt/vet/test/build); endpoint changes update
  `server/docs/openapi.yaml`.
- Console: `cd server/admin && pnpm build` passes (`pnpm lint`, `pnpm typecheck`, `pnpm test`
  where relevant).
- Deliver "command + result" evidence; "should work" is not accepted.
