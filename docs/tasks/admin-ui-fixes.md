# admin-ui-fixes — 无分页表格底部杂线 + 工作台恢复滚动与下边距

- **Branch**: `admin-ui-fixes`
- **Base**: `main-relay`
- **Area**: `server/admin`（Vue 管理控制台，唯一区域）
- **Dependencies**: `admin-ui-fullscreen`（已合并 `ec7f377`，是本次两个问题的来源）
- **Status**: see `docs/tasks/STATUS.md`

## 1. Goal

上一任务把页面改成"撑满视口 + 卡片内部滚动"后暴露两个问题，用户已确认要修：

1. **无分页器的表格底部多出一条 1px 杂线** —— 是 Element Plus 画的 `.el-table__inner-wrapper::before`（旧结构为 `.el-table::before`）底边线；表体现在撑满整张卡片，这条线落在卡片底部，看起来像一条多余横线。**只在没有分页器的表格卡片里移除**；有分页器的页面那条线夹在表格与分页之间，是正常分隔，保持现状。
2. **工作台（dashboard）是纯展示页，应该允许纵向滚动并保留下边距** —— 现在被做成"一屏放完"，内容超出时被压缩/贴到视口底边，底部没有留白。用户要求：保留下边距 + 允许滚动。

> 用户原话口径（不要自行扩大）：①「在没有分页器的表格底部会多出来一个线，看看好不好移除」；②「工作台页面应该保留下边距 并且让工作台页允许滚动 这种纯展示页面 允许滚动」。

## 2. 现状（main-relay 已合并的代码）

**问题 1 的现场**（`#/system/role`，暗色模式，devtools 已确认）：
- DOM：`div.el-table.el-table--fit.el-table--striped...`（内联 `style="height: 100%"`）→ `div.el-table__inner-wrapper` → 高亮伪元素 `::before`，尺寸 `1440 × 1`。
- 该列表页**没有分页器**；`src/views/system/role/index.vue:209` 与 `src/views/system/userLevel/index.vue` 的表格卡片同样无分页器。
- Element Plus 的规则大致是：

```css
.el-table__inner-wrapper::before,
.el-table::before {
  position: absolute;
  bottom: 0;
  left: 0;
  z-index: 1;
  width: 100%;
  height: 1px;
  content: "";
  background-color: var(--el-table-border-color);
}
```

**问题 2 的现场**（`src/views/dashboard/index.vue`）：
- 页根：`class="page-fill dashboard-fill"`，scoped 样式里有 `.dashboard-fill { overflow: hidden auto; }`；
- 三行：`.dash-row--stats { flex: 0 0 auto }`、`.dash-row--charts { flex: 0 0 460px; min-height: 440px }`、`.dash-row--bottom { flex: 1 1 auto; min-height: 140px }`（这三条 + `.el-card{height:100%}` + `.el-card__body` flex 就是"一屏放完/压缩"的来源）；
- 数据统计 / 最新动态两个 `<el-scrollbar>` 已经去掉了原来的 `max-height="504"`，改成 `flex: 1 1 auto` 自适应撑满卡片；
- scoped `.main-content { margin: 20px 20px 0 !important; }` —— **下边距是 0**，这就是"没有下边距"的来源。

**必须保持不回退的既有验收**（上一任务已通过，本次不得破坏）：
- 列表页 / 卡片页：视口高度 ≥ 900px 时页面无纵向滚动条，卡片撑满标签栏以下区域；表头固定、表体在卡片内滚动、分页贴卡片底；短视口降级为内部滚动且不裁切内容。

## 3. In scope

### R1 移除无分页表格底部的 1px 杂线

- 在 `server/admin/src/style/business.scss` 增加规则，去掉 `.table-scroll` 内 el-table 的底部伪元素线，伪元素两个选择器都覆盖（EP 版本不同）：

```scss
/* 表体撑满卡片后，EP 画的 1px 底边线会变成卡片底部的一条杂线 */
.table-scroll .el-table__inner-wrapper::before,
.table-scroll .el-table::before {
  display: none;
}
```

- **仅对没有分页器的表格卡片生效**。两种实现都可，任选其一并说明：
  - (a) css 选择器：`.panel-body.table-body:not(:has(.el-pagination)) .table-scroll …`（或 `:not(:has(> div > .el-pagination))`，注意分页被套在 `div.flex.justify-end.pt-3` 里）；
  - (b) 给两个无分页页面（`system/role`、`system/userLevel`）的表格容器加一个 class（例如 `table-scroll--no-pagination`），css 只针对该 class。
- **有分页器的页面（`system/user`、`system/administrator`、`system/audit`）必须保持现状**：表格与分页之间那条线不要动。
- 注意：**只删这一条伪元素线**，不要动 `el-table` 的其它边框（表头下边线、行分隔线、列边框、固定列阴影都要保留）。

### R2 工作台：保留下边距 + 允许纵向滚动

- `src/views/dashboard/index.vue`：
  - 去掉 `.dashboard-fill { overflow: hidden auto; }`（不要让页面根部自己成为滚动容器，交给 layout 的 `el-scrollbar`）；
  - 去掉"一屏放完"的高度约束：`.dash-row--charts` 的固定/最小高度、`.dash-row--bottom` 的 `flex: 1 1 auto; min-height` 等，让三行按内容高度自然排布；
  - 数据统计 / 最新动态两个 `<el-scrollbar>` **恢复有界高度**（重新加 `max-height="504"`，或等价 CSS `max-height`），避免这两个列表随内容无限升高；
  - scoped `.main-content` 恢复下边距：`margin: 20px !important;`（即上/右/下/左都保留 20px，滚动到底时最后一张卡片下方要有留白）。
- 结果：视口够高时工作台自然铺开且底部有 20px 留白；视口不够高时**整页可以纵向滚动**（用 layout 自带的 `el-scrollbar`，不要出现两层滚动条、不要用原生滚动条）。
- 卡片仍然撑满所在列宽（行内两张卡等高可保留现状），图表（`ChartBar` 365px / `ChartLine` / `ChartRound`）、卡片内容、数据结构、`v-motion` 动画都不要改。
- 工作台的 `el-scrollbar` 滚动条被既有样式隐藏（`:deep(.el-card) .el-scrollbar__bar { display: none }`）——**本任务不动它**。

## 4. Out of scope

- 不改其它页面的"撑满 + 页面不滚"行为（列表页、个人中心、系统设置都必须保持上一任务的验收结果）。
- 不改任何 Go 代码、`server/docs/openapi.yaml`、生成的 `types/api.generated.ts`、接口与鉴权。
- 不改 `platform-config.json`（`FixedHeader` / `Stretch` / `HideTabs` 保持）、不改侧边栏/标签栏/导航栏。
- 不改动各页面的弹窗、表格列定义、业务字段、接口调用。
- 不重构 `.page-fill` / `.panel-fill` / `.table-scroll` 骨架，不做新的视觉设计。

## 5. Files / areas

| 文件 | 动作 |
| ---- | ---- |
| `src/style/business.scss` | R1：新增去杂线的规则（仅无分页表格） |
| `src/views/dashboard/index.vue` | R2：去一屏约束 + 恢复列表有界高度 + 恢复下边距 |
| `src/views/system/role/index.vue` | 仅当选择实现 (b) 时：给表格容器加 class |
| `src/views/system/userLevel/index.vue` | 仅当选择实现 (b) 时：给表格容器加 class |

## 6. Acceptance criteria

1. `#/system/role`、`#/system/userLevel`（**无分页器**）：表格卡片底部不再出现那条 1px 横线（浅色 / 深色都验）；用 DOM/样式证据说明伪元素已不生效，不要只说"看不见了"。
2. `#/system/user`、`#/system/administrator`、`#/system/audit`（**有分页器**）：表格与分页之间的分隔线保持现状（未被误删），分页仍在卡片底部。
3. `#/dashboard`：滚动到底时最后一张卡片下方有 ≥16px 留白（`.main-content` 下边距生效）；内容高于视口时**可以纵向滚动到底**（能看到完整的底部两张卡片），只有一层滚动条；数据统计 / 最新动态在卡片内部有界滚动（滚不到无限长）。
4. 回归：`#/system/user`、`#/system/administrator`、`#/system/role`、`#/system/audit`、`#/system/userLevel`、`#/profile/index`、`#/system/settings` 在 1843×1299 下**仍然**无页面纵向滚动条、卡片仍撑满、表头固定、表体内部滚动、分页贴卡片底。
5. `pnpm lint`、`pnpm typecheck`、`pnpm test`、`pnpm build` 全部通过。
6. 深色模式下 R1/R2 同样成立。

## 7. Verification（executor 必须给出 command + result）

```sh
cd <task-worktree>/server/admin
pnpm lint
pnpm typecheck
pnpm test
pnpm build
```

浏览器验证（必须做；dev server 用 3000 端口，8848 被 main-relay 占用，别动）：

```sh
cd <task-worktree>/server/admin && pnpm dev --port 3000
ORCA=/Applications/Orca.app/Contents/Resources/app.asar.unpacked/out/cli/index.js
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA tab create --url "http://localhost:3000/#/login" --json
# 用 result.browserPageId 作为 --page： snapshot → fill(admin/admin123) → click 登录
# 注意：截图前先 orca tab switch --page <pageId>，否则截图可能是空/发白的一帧；整页 goto 到 SPA 路由后要轮询等路由组件渲染
```

需要给出的**数字证据**（`orca eval --page <pageId> --expression "…"`）：

```js
// R1：无分页表格的伪元素线已移除（角色页 / 用户等级页）
getComputedStyle(document.querySelector('.table-scroll .el-table__inner-wrapper'), '::before').display   // 期望 "none"
// 有分页页面：伪元素线仍在（业务用户 / 管理员 / 审计）
getComputedStyle(document.querySelector('.table-scroll .el-table__inner-wrapper'), '::before').display   // 期望不是 "none"
// R2：工作台可滚 + 有下边距
(() => { const am = document.querySelector('.app-main'), mc = document.querySelector('.main-content');
  return { appMain: [am.clientHeight, am.scrollHeight], mainContent: [mc.clientHeight, mc.scrollHeight],
           marginBottom: getComputedStyle(mc).marginBottom, lastCardBottom: Math.round([...document.querySelectorAll('.el-card')].pop().getBoundingClientRect().bottom) }; })()
// R2：两个列表仍有界
[...document.querySelectorAll('.el-card__body > .el-scrollbar')].map(e => { const w = e.querySelector('.el-scrollbar__wrap'); return [e.clientHeight, w.scrollHeight]; })
```

还要给出：浅色 + 深色各一张 `#/system/role` 截图（杂线消失）、一张工作台滚到底的截图（有下边距）。

## 8. Notes for the executor

- 首个提交必须是 `chore(admin-ui-fixes): claim task`；实现提交用 `feat(admin-ui-fixes): ...`。
- **环境**：`server/admin/.env`、`.env.development` 已放好，`node_modules` 已安装；dev server 用 `pnpm dev --port 3000`，**不要动 main-relay 的 8848**；后端 `:8080` 在跑，登录 `admin / admin123`。
- 只改 §5 列出的文件；不要动 `docs/tasks/**`、PRD/ADR、契约、其他区域。
- 改动要小：这是两个针对性修复，不要顺手重构布局骨架或改其他页面。
- 完成后报告：改动摘要 + 每条验证命令与数字结果 + 截图路径 + 偏离说明；缺证据要明说。
- 完成后不要 merge、不要 push、不要改台账，等 review。
