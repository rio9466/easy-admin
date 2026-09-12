# admin-brand-ux — 控制台品牌图标与账号交互整理

- **Branch**: `admin-brand-ux`
- **Base**: `main-relay`
- **Area**: `server/admin`（另含：删除仓库根目录 `icon.png`，已获用户授权）
- **Dependencies**: none
- **Status**: see `docs/tasks/STATUS.md`

## 1. Goal

三项控制台界面整理，一次交付：

1. **替换默认图标**：删除旧的占位图标，把仓库根目录 `icon.png` 去掉纯白背景（转透明）后放到
   `server/admin/public/` 的 favicon 位置。
2. **修改密码改为弹窗**：把"修改密码"从个人中心页面剥离成全局对话框，由导航栏用户菜单触发。
3. **登录页精简**：删除登录页的"管理后台登录"标题文字，只保留 `easy-admin` 品牌文字。

## 2. In scope

### R1 图标替换

- 用 `ffmpeg` 把根目录 `icon.png`（600×600，白底）抠成透明背景，输出到
  `server/admin/public/favicon.png`。
- 更新 `server/admin/index.html` 的 favicon 引用：`/favicon.svg` → `/favicon.png`。
- 删除旧占位图标：`server/admin/public/favicon.svg`、`server/admin/public/logo.svg`
  （两者均为模板占位 SVG，`logo.svg` 无任何代码引用）。
- 删除仓库根目录 `icon.png`（源文件已移到 `public/`，用户授权跨目录删除）。

> 决策：用户确认按"默认方案 A"——位置保持 `server/admin/public/`，文件名用 `favicon.png`
> （位图不能直接命名成 `.svg`，否则浏览器解析失败）。不采用 base64 内嵌 SVG 的写法。

**抠图命令（已验证可用，本机 ffmpeg）：**

```sh
cd <task-worktree-root>
ffmpeg -y -i icon.png \
  -vf "format=rgba,geq=r='r(X,Y)':g='g(X,Y)':b='b(X,Y)':a='255-min(r(X,Y),min(g(X,Y),b(X,Y)))'" \
  -pix_fmt rgba server/admin/public/favicon.png
```

背景接近纯白（alpha≈0）、线条颜色保留、抗锯齿边缘得到连续 alpha，比 `colorkey` 效果好得多。
验证：`sips -g hasAlpha server/admin/public/favicon.png` 应输出 `hasAlpha: yes`，
且用深色背景合成查看时原图线条完整、无白底。

若 `ffmpeg` 不可用：**停止并报告 blocker**，不要提交带白底的图标。

### R2 修改密码弹窗

- 新增全局组件 `server/admin/src/components/ChangePasswordDialog/index.vue`。
- 把 `server/admin/src/views/profile/index.vue` 中现有的修改密码逻辑**整体迁移**进去：
  - `passwordForm` / `passwordRules` / `passwordFormRef` / `passwordLoading`；
  - `submitPassword()`（成功后 `resetAuthState()` → 清 tags → `resetRouter()` → 跳 `/login`）；
  - 相关 imports（`message`、`getErrorMessage`、`useUserStoreHook`、`createPasswordByteValidator`、
    `PASSWORD_PLACEHOLDER`、`resetRouter`、`useMultiTagsStoreHook`、`routerArrays`）。
- 表单字段与校验**保持不变**：当前密码、新密码（字节校验）、确认新密码（一致性校验）、
  顶部"修改成功后会话吊销需重新登录"提示。
- 弹窗需支持：打开时重置表单与校验状态、`destroy-on-close` 或等价处理、关闭时清空输入。
- `server/admin/src/layout/components/lay-navbar/index.vue`：
  - "修改密码"下拉项改为**打开弹窗**（不再走 `goProfile()`）；
  - "个人中心"下拉项保持 `goProfile()` 不变。
- `server/admin/src/views/profile/index.vue`：删除"修改密码"面板及其全部脚本逻辑，仅保留
  "个人资料 / 显示名称"面板；清理由此产生的未使用 import。

### R3 登录页文案

- `server/admin/src/views/login/index.vue`：删除 `<h2 class="outline-hidden">管理后台登录</h2>`
  所在的整个 `<Motion>` 块；保留其上方 `<p class="login-brand">easy-admin</p>`。
- `server/admin/src/style/login.css`：删除仅服务于该标题的死样式
  `.login-form h2`、`.dark-mode .login-form h2`，以及 1180px 媒体查询里的
  `.login-form h2 { font-size... }` 规则。
- `server/admin/index.html` 的 `<title>easy-admin 管理后台</title>` **不改**。

## 3. Out of scope

- 不修改任何 `server/` 下的 Go 代码、`server/docs/openapi.yaml` 或生成的 `types/api.generated.ts`。
- 不改后端接口、鉴权流程、token 存储方式。
- 不新增登录页/侧边栏的图形 logo（当前品牌区刻意只保留文字）。
- 不改登录页 `<title>` 之外的任何页面文案。
- 不做设计重构、不加动画/主题等额外功能。

## 4. Files / areas

| 文件 | 动作 |
| ---- | ---- |
| `icon.png`（仓库根） | 删除（源文件已移入 public，目标为 `favicon.png`） |
| `server/admin/public/favicon.png` | 新增（透明背景） |
| `server/admin/public/favicon.svg` | 删除 |
| `server/admin/public/logo.svg` | 删除 |
| `server/admin/index.html` | favicon 引用改为 `/favicon.png` |
| `server/admin/src/components/ChangePasswordDialog/index.vue` | 新增 |
| `server/admin/src/views/profile/index.vue` | 删除修改密码面板及逻辑 |
| `server/admin/src/layout/components/lay-navbar/index.vue` | "修改密码"改为开弹窗 |
| `server/admin/src/views/login/index.vue` | 删除"管理后台登录"标题 |
| `server/admin/src/style/login.css` | 删除标题死样式 |

## 5. Acceptance criteria

1. `server/admin/public/` 下存在 `favicon.png` 且 `hasAlpha: yes`；`favicon.svg`、`logo.svg` 不存在；
   仓库根目录 `icon.png` 不存在。
2. `index.html` 引用 `/favicon.png`；`pnpm build` 产物中 favicon 可从 public 复制（无 404）。
3. 导航栏点"修改密码"弹出对话框；提交成功后提示、登出、跳转 `/login`，与旧行为一致。
4. 个人中心页不再出现修改密码表单，仅剩个人资料面板。
5. 登录页不显示"管理后台登录"，仍显示 `easy-admin`。
6. `pnpm lint`、`pnpm typecheck`、`pnpm test`、`pnpm build` 全部通过。

## 6. Verification（executor 必须给出 command + result）

```sh
cd server/admin
pnpm lint
pnpm typecheck
pnpm test
pnpm build

# R1 产物
sips -g hasAlpha public/favicon.png          # 期望 hasAlpha: yes
ls public/favicon.svg public/logo.svg        # 期望 No such file
ls ../../icon.png                            # 期望 No such file
grep -n "favicon" index.html                 # 期望 /favicon.png
```

浏览器/人工检查（可截图为证）：

- 登录页只有 `easy-admin` 文字，无"管理后台登录"。
- 登录后导航栏"修改密码"打开弹窗，个人中心无修改密码面板。
- 标签页图标为新图标，背景透明。

## 7. Notes for the executor

- 首个提交必须是 `chore(admin-brand-ux): claim task`；实现提交用 `feat(admin-brand-ux): ...`。
- 只改本文件列出的文件；不要动 `docs/tasks/**`、PRD、ADR、契约、其他区域。
- 不要为了让 lint 通过而删除与本任务无关的既有代码。
- 完成后报告：改动摘要 + 每条验证命令及其结果 + 任何偏离本任务的地方。
