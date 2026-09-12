# API contract

The API source of truth is [`server/docs/openapi.yaml`](../../server/docs/openapi.yaml),
owned by the main line (`main-relay`).

Rules:

- Update `server/docs/openapi.yaml` whenever an endpoint or payload changes, in the same task.
- The management console regenerates `server/admin/types/api.generated.ts` with
  `pnpm generate:api`; never edit that file by hand — add handwritten adapters in
  `server/admin/src/api/contract.ts`.
- Executors must not change request/response shapes on their own. If the contract is missing
  something, stop and report to `main-relay`.

Frontend ↔ service contract notes (envelope, auth, error codes) can be added here as
`*.md` documents when they are cross-cutting.
