# admin-ui-fullscreen — 页面撑满视口 / 禁止页面滚动 / 移除页脚

- **Branch**: `admin-ui-fullscreen`
- **Base**: `main-relay`
- **Area**: `server/admin`（Vue 管理控制台，唯一区域）
- **Dependencies**: none
- **Status**: see `docs/tasks/STATUS.md`

## 1. Goal

三项界面统一性改造，一次交付：

1. **移除登录后的页脚**：`Copyright © 2026 easy-admin` 彻底删除（组件、渲染位、设置开关、配置项一起删）。
2. **页面撑满视口**：列表页、卡片页的高度由"布局撑满"决定，**不再由内容撑开**；页面本身不出现纵向滚动条。
3. **滚动内聚**：表格卡片里表头与分页固定，**表体在卡片内部滚动**；工作台保持一屏，卡片内区域各自滚动。

> 用户已确认的口径（不要自行更改）：
> - Q1：允许"卡片内部滚动"（这是"撑满 + 页面不滚"唯一可落地的组合，表格数据不能被裁掉）。
> - Q2：工作台选 **方案 a** —— 整页不滚动，卡片格子固定、图表自适应，右下"数据统计 / 最新动态"在各自卡片内部滚动。
> - Q3：页脚**彻底删除**，不是"默认隐藏"。

## 2. 现状（已实测，照这个理解改）

视口 1843×1299 时的实测链路（`#/system/user`）：

```
section.app-main           h=1299  height:100vh  overflow-x:hidden（=> 计算后 overflow-y:auto，可滚）
 └ div.w-full.h-full       h=1218  （.app-main 有 padding-top:81px 给标签栏）
    └ div.el-scrollbar     h=1218  overflow:hidden
       └ ...__wrap         h=1218  overflow:auto      ← 页面滚动的来源
          └ ...__view      h=1218  display:flex;flex-direction:column;flex:1 1 auto
             └ div.grow    h=1189  flex:1 1 auto（Tailwind .grow）display:block ← 没有把高度传下去
                └ 页面根节点 `div.flex.flex-col.gap-4`  h=441  ← 内容高度，没撑满
                   ├ .panel-container（筛选）      h=64
                   └ .panel-container（表格）      h=357
                      └ .panel-body.table-body    h=300  min-height:300px
                         └ .el-table              h=100（无 height，随行数长高 → 页面滚）
             └ LayFooter（页脚）  ≈29px
```

关键点：

- 页面高度断链在 `div.grow`（`display:block`）与页面根节点（内容高度）之间。
- `.main-content`（由 `lay-content` 加到路由组件根节点上）当前只有 `margin: 24px`。
- 页面滚动来自 `el-scrollbar__wrap { overflow:auto }`，`.app-main` 因 `overflow-x:hidden` 计算成了 `overflow-y:auto`（第二个滚动容器）。
- 列表页统一结构：`根 > 筛选 .panel-container > .panel-container > .panel-header + .panel-body.table-body > el-table + el-pagination(套在 .flex.justify-end.pt-3 里)`。
- 所有业务页都在自己的 `<style scoped>` 里 `@import url("@/style/business.scss")`（scoped 导入，规则只作用于本组件）。
- 工作台 `#/dashboard` 结构：`根 div > el-row > re-col(value=6 ×4 统计卡) / re-col(18 分析概览 + 6 解决概率) / re-col(18 数据统计 + 6 最新动态)`；两个列表已用 `<el-scrollbar max-height="504">` 包裹。工作台另有 scoped `.main-content { margin: 20px 20px 0 !important }`。
- `ChartBar` 内部固定 `height: 365px`；`ChartLine` / `ChartRound` 固定 `height: 60px`。
- 页脚相关引用：`lay-footer/index.vue`、`lay-content/index.vue`（两处 `<LayFooter>` + import + `hideFooter` computed）、`lay-setting/index.vue`（"隐藏页脚"开关 + `hideFooterChange`）、`useLayout.ts`、`utils/responsive.ts`、`public/platform-config.json` 的 `HideFooter`。`layout-footer` 除了自身组件外无其他引用。
- `public/platform-config.json` 的 `FixedHeader: true`、`Stretch: false` **不要改**（`Stretch` 打开会加 `max-width:1440px`，与"撑满"冲突）。

## 3. In scope

### R1 移除页脚（彻底）

- 删除 `server/admin/src/layout/components/lay-footer/index.vue`。
- `server/admin/src/layout/components/lay-content/index.vue`：
  - 删除两处 `<LayFooter v-if="…" />`（fixedHeader 分支内、fixedHeader 之外各一处）与 `LayFooter` import；
  - 删除 `hideFooter` computed（删掉后若 `$storage` 不再使用，连带清理 import，别留未使用变量）。
- `server/admin/src/layout/components/lay-setting/index.vue`：删除"隐藏页脚"`<li>` 整块、`hideFooterChange()` 及其调用点（`settings.hideFooter && hideFooterChange()`）、`settings.hideFooter` 字段。
- `server/admin/src/layout/hooks/useLayout.ts`、`server/admin/src/utils/responsive.ts`：删除 `hideFooter` 初始化项。
- `server/admin/public/platform-config.json`：删除 `"HideFooter": false`（注意保持 JSON 合法、逗号正确）。
- 改完 `grep -rn "hideFooter\|HideFooter\|LayFooter\|layout-footer" server/admin/src server/admin/public` 应为空（`ReDialog` 的 `hideFooter` 是弹窗自己的选项，**不属于本次范围，不要动**）。

### R2 布局基础：撑满 + 页面不滚

- `server/admin/src/layout/components/lay-content/index.vue` 的 scoped 样式：
  - `.app-main` 增加 `overflow: hidden`（保持 `height: 100vh`），让页面滚动的来源只剩内部滚动区；
  - `div.grow` 增加 `display: flex; flex-direction: column; min-height: 0;`（把高度传下去；Tailwind 的 `.grow` 只有 `flex-grow:1`）；
  - `.main-content` 增加 `flex: 1 1 auto; min-height: 0;`（保留 `margin: 24px`）。
- `server/admin/src/style/business.scss` 增加可复用的撑满/滚动骨架类（各业务页已 scoped 导入该文件）：
  - `.page-fill`：页面根节点用，`display:flex; flex-direction:column; gap:16px; flex:1 1 auto; min-height:0;`
  - `.panel-fill`：需要撑满剩余高度的面板，`display:flex; flex-direction:column; flex:1 1 auto; min-height:0;`
  - `.table-scroll`：表格滚动容器，`flex:1 1 auto; min-height:0;`
  - `.panel-body.table-body` 改为 flex 列：`display:flex; flex-direction:column; gap:12px; flex:1 1 auto; min-height:0;`（保留原有 `min-height` 语义时改成合理的最小值，例如 `min-height: 240px`，不要再用固定 `300px` 把页面顶高）。
- **降级要求**：短视口（例如高度 < 900px）不允许出现内容被裁掉；此时允许内部滚动区出现滚动条（不要用 `overflow:hidden` 硬裁页面内容）。

### R3 列表页（5 个）

`system/user`、`system/administrator`、`system/role`、`system/audit`、`system/userLevel` 的 `index.vue`：

- 根节点 `class="flex flex-col gap-4"` → `class="page-fill"`（间距由 `.page-fill` 提供，保持 16px，与原 `gap-4` 视觉一致）。
- 第二个（表格）`.panel-container` 增加 `panel-fill`。
- `.panel-body.table-body` 内的 `<el-table>` 外面包一层 `<div class="table-scroll">`，并给 `el-table` 加 `height="100%"`：表头固定、表体在卡片内部滚动。
- `el-pagination` 保持在该面板底部（现有 `div.flex.justify-end.pt-3` 保留即可，会被 flex 排在表格下方）。
- **只改布局类与包裹层，不要动列定义、接口调用、分页逻辑、弹窗逻辑。**
- 各页内已有的弹窗（例如用户详情弹窗里的 `<el-table max-height="460">`）**保持原样**，弹窗不属于本次范围。

### R4 卡片页

- `views/profile/index.vue`：根节点 → `page-fill`；唯一的 `.panel-container` 增加 `panel-fill`（卡片本身撑满视口高度，内容仍顶部对齐、不拉伸表单）。
- `views/system/settings/index.vue`：根节点 → `page-fill`；`.panel-container` 增加 `panel-fill`；`.panel-body.settings-layout` 撑满（`flex:1 1 auto; min-height:0`）；`.settings-form` 设为 `flex:1 1 auto; min-height:0; overflow:auto`（右侧表单在卡片内部滚动）；左侧 `.settings-nav` 在窄屏/内容超高时同样允许内部滚动。窄屏（≤768px）规则保持可用，不要破坏。

### R5 工作台 `#/dashboard`（方案 a：一屏放完）

- 根节点增加 `page-fill`（保留现有 scoped `.main-content { margin: 20px 20px 0 !important }`），并让 `el-row` 成为撑满的纵向容器。
- 三行高度分配（建议实现，数值可微调，但必须满足验收）：
  - 统计卡行：内容高度（`flex: 0 0 auto`）；
  - 图表行（分析概览 + 解决概率）：`flex: 0 0 460px` 左右（要容下 `ChartBar` 的 365px + 卡片头内边距），最小高度不小于 440px；
  - 底部行（数据统计 + 最新动态）：`flex: 1 1 auto; min-height: 320px`。
- 每个 `el-card` 撑满所在单元格（`height:100%` + `.el-card__body` 为 `display:flex; flex-direction:column; min-height:0`），卡片头固定，内容区 `flex:1 1 auto; min-height:0`。
- 底部两卡片的 `<el-scrollbar max-height="504">` 改为自适应撑满（去掉固定 `max-height`，让 `el-scrollbar` 占满卡片剩余高度并内部滚动）。
- 图表组件本身（`ChartBar` / `ChartLine` / `ChartRound`）**不要改**；若卡片高度不足导致遮挡，只调整行高/内边距，不要改图表尺寸或删卡片。
- 页面根节点不出现纵向滚动条（视口高度 ≥ 900px）；短视口允许降级为内部滚动（见 R2 降级要求）。

## 4. Out of scope

- 不改任何 Go 代码、`server/docs/openapi.yaml`、生成的 `types/api.generated.ts`、接口与鉴权逻辑。
- 不改登录页、403/404/500 页面（错误页保持居中）。
- 不改侧边栏、标签栏、导航栏、搜索、设置抽屉的其余项（Logo、深色模式等保持原样）。
- 不新增功能、不加动画、不换主题、不改文案、不改表格列与业务字段。
- 不改 `platform-config.json` 的 `FixedHeader`、`Stretch`、`HideTabs`。
- 不动各页面里的弹窗内部布局（含弹窗内 `el-table max-height`）。

## 5. Files / areas

| 文件 | 动作 |
| ---- | ---- |
| `src/layout/components/lay-footer/index.vue` | 删除 |
| `src/layout/components/lay-content/index.vue` | 删页脚渲染/import/`hideFooter`；加撑满基础样式与 `.app-main{overflow:hidden}` |
| `src/layout/components/lay-setting/index.vue` | 删"隐藏页脚"开关及相关逻辑 |
| `src/layout/hooks/useLayout.ts` | 删 `hideFooter` 初始化 |
| `src/utils/responsive.ts` | 删 `hideFooter` 初始化 |
| `public/platform-config.json` | 删 `HideFooter` |
| `src/style/business.scss` | 新增 `.page-fill` / `.panel-fill` / `.table-scroll`，调整 `.panel-body.table-body` |
| `src/views/system/user/index.vue` | R3 |
| `src/views/system/administrator/index.vue` | R3 |
| `src/views/system/role/index.vue` | R3 |
| `src/views/system/audit/index.vue` | R3 |
| `src/views/system/userLevel/index.vue` | R3 |
| `src/views/profile/index.vue` | R4 |
| `src/views/system/settings/index.vue` | R4 |
| `src/views/dashboard/index.vue` | R5 |

## 6. Acceptance criteria

1. 登录后任意页面都不再出现 `Copyright` / 页脚；设置抽屉里不再有"隐藏页脚"开关；配置里无 `HideFooter`。
2. **列表页**（用户/管理员/角色/审计/等级）在视口高度 ≥ 900px 时：卡片撑满标签栏以下区域，**页面无纵向滚动条**；表格行多时表头仍固定在卡片顶部、分页仍在卡片底部、表体在卡片内部滚动。
3. **卡片页**：个人中心的"个人资料"卡片、系统设置的卡片撑满视口高度（不靠内容撑）；内容超高时在卡片内部滚动，页面不滚。
4. **工作台**：视口高度 ≥ 900px 时不出现页面纵向滚动条；统计卡、图表行、底部两卡片都在视口内可见；"数据统计"与"最新动态"在各自卡片内部滚动；图表未被裁切/遮挡。
5. 短视口（例如 1280×800）降级为内部滚动，**不出现内容被裁掉看不见**的情况。
6. `pnpm lint`、`pnpm typecheck`、`pnpm test`、`pnpm build` 全部通过；登录页 / 403 / 404 视觉无回归。
7. 深色模式（设置抽屉切换）下上述页面无布局回归（至少列表页与个人中心）。

## 7. Verification（executor 必须给出 command + result）

```sh
cd <task-worktree>/server/admin
pnpm lint
pnpm typecheck
pnpm test
pnpm build
```

浏览器验证（必须做，不能只靠静态分析；本机可用 Orca 自带浏览器 CLI）：

```sh
# 起任务分支自己的 dev server（8848 被 main-relay 占用，用 3000 端口，后端 trusted_origins 已包含 3000）
cd <task-worktree>/server/admin && pnpm dev --port 3000

ORCA=/Applications/Orca.app/Contents/Resources/app.asar.unpacked/out/cli/index.js
# 登录页 → 填账号密码 → 进系统
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA tab create --url "http://localhost:3000/#/login" --json
# 用返回的 result.browserPageId 作为 --page，配合 snapshot / click / fill / eval / screenshot
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA snapshot --page <pageId>
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA fill --page <pageId> --element e3 --value admin
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA fill --page <pageId> --element e4 --value admin123
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA click --page <pageId> --element e2
ELECTRON_RUN_AS_NODE=1 /Applications/Orca.app/Contents/MacOS/Orca $ORCA screenshot --page <pageId> --format png --json   # result.data 是 base64
```

每个页面都要用 `eval` 给出**可判定的数字证据**（不要只说"看起来没问题"），例如：

```js
// 页面是否可滚：scrollHeight 必须等于 clientHeight
document.querySelector('.app-main').scrollHeight === document.querySelector('.app-main').clientHeight
// 卡片是否撑满：面板高度应接近内容区高度
document.querySelector('.main-content').clientHeight
document.querySelector('.panel-fill').clientHeight
// 表体内部滚动：表体容器可滚、表头不动
const w = document.querySelector('.table-scroll .el-table__body-wrapper');
({ clientH: w.clientHeight, scrollH: w.scrollHeight, scrollable: w.scrollHeight > w.clientHeight })
// 页脚已消失
!document.querySelector('.layout-footer')
```

至少覆盖：`#/system/user`（行少 + 行多两种）、`#/system/administrator`、`#/system/role`、`#/system/audit`、`#/system/userLevel`、`#/profile/index`、`#/system/settings`、`#/dashboard`，以及 1280×800 与 1843×1299 两个视口尺寸（可用 `orca exec --command` 或直接改窗口大小；若无法改视口，至少报告实际视口尺寸并说明未覆盖尺寸）。

## 8. Notes for the executor

- 首个提交必须是 `chore(admin-ui-fullscreen): claim task`；实现提交用 `feat(admin-ui-fullscreen): ...`。
- **环境**：worktree 里 git-ignored 的 `server/admin/.env`、`.env.development` 已由 conversation pi 放好；首次需要先 `pnpm install`（store 已预热，很快）。**不要动 main-relay 上 8848 的 dev server**，用 3000。
- 只改本文件列出的文件；不要动 `docs/tasks/**`、PRD/ADR、契约、其他区域。不要为了让 lint 通过删除与本任务无关的既有代码。
- 纯布局改动，不要顺手重构业务逻辑；`el-table` 只加 `height="100%"` 与外层 `.table-scroll`。
- 完成后报告：改动摘要 + 每条验证命令及其真实输出（含浏览器 `eval` 的数字结果与截图路径）+ 任何偏离本任务的地方。
- 完成后不要 merge、不要 push、不要改台账，等 review。
