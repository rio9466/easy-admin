# Task status ledger

Maintained by the conversation pi (on `main-relay`). **Executors must not edit this file**; claiming and
progress are reported through the task branch commits (`chore(<ID>): claim task`,
`feat(<ID>): ...`) and the completion report, and recorded here by the conversation pi.

Status values:

- `todo` — assigned (branch/worktree created), unclaimed
- `in-progress` — claimed, in progress
- `in-review` — implementation done, awaiting conversation pi review
- `done` — merged into `main-relay`
- `blocked` — blocked (write the reason under "Evidence / notes")

| ID  | Task | Branch | Base | Dependencies | Status | Evidence / notes |
| --- | ---- | ------ | ---- | ------------ | ------ | ---------------- |
| admin-brand-ux | 图标替换 + 修改密码弹窗 + 登录页文案 | `admin-brand-ux` | `main-relay` | — | done | 合并提交 `f798990`（--no-ff）；conversation pi 复核通过（typecheck/test/build 绿 + 浏览器实测） |
| admin-ui-fullscreen | 移除页脚 + 页面撑满视口 + 禁止页面滚动（表格/卡片/工作台） | `admin-ui-fullscreen` | `main-relay` | — | todo | 任务文档 `docs/tasks/admin-ui-fullscreen.md` |

## Update rules

- Conversation pi: set `todo` after creating a task branch; `in-progress` on claim receipt;
  `in-review` on completion receipt; `done` after merging into `main-relay`, with the merge
  commit as evidence.
- Only one branch works a task ID; a task already `in-progress` must not be started again.
- Delete a task branch only after its task is `done`.
