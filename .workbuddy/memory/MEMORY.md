# 项目长期记忆

## baidu-netdisk-node 项目概况

- monorepo 结构，使用 pnpm workspace，包含 api/sdk/srv/web/xth 五个子包
- 技术栈：Node.js ≥20，pnpm ^10，TypeScript，Vue3+Vite（web 包）
- 构建命令：`pnpm build srv web`（sync 服务）、`pnpm build xth`（auth 服务）

## Docker 镜像

- Dockerfile 位于 `packages/srv/Dockerfile` 和 `packages/xth/Dockerfile`
- 镜像分两个：`baidu-netdisk-sync`（srv+web）和 `baidu-netdisk-auth`（xth）
- 2026-03-31：已将 GitHub Actions 工作流从 DockerHub 改为推送到 ghcr.io
  - 使用 `secrets.GITHUB_TOKEN` 鉴权，无需额外配置 secrets
  - 镜像地址：`ghcr.io/<owner>/baidu-netdisk-sync` 和 `ghcr.io/<owner>/baidu-netdisk-auth`
  - 触发方式：手动 workflow_dispatch + push tag（`srv-v*` / `xth-v*`）
- 2026-03-31：大幅优化构建速度
  - **Dockerfile** 改为多阶段构建：先 COPY `package.json`+lockfile 安装依赖（可缓存），再 COPY 源码构建，最后产物拷入纯净 runtime 镜像
  - **Workflow** 改为矩阵并行策略：amd64 用 `ubuntu-latest`（原生），arm64 用 `ubuntu-24.04-arm`（GHA 原生 arm64，免费），armv7 在 arm64 runner 上跑 QEMU（比 amd64 模拟快很多）；三平台并行后 `merge` job 用 `imagetools create` 合并 manifest
  - 镜像层缓存改为 `type=registry`，每个平台独立缓存 tag（`cache-amd64`/`cache-arm64`/`cache-armv7`），跨 run 命中率更高

## UI 现代化改版预览

- 预览文件：`brain/<conv-id>/ui-preview.html`（standalone HTML，无需构建）
- 设计方向：左侧边栏布局、深色渐变卡片头、CSS 变量设计 token、accent #4f6ef7

## UI 改动已落地到实际 Vue 组件（2026-03-31）

已修改文件列表：
- `packages/web/assets/app.css`：注入 CSS 设计 token `:root` 变量（--accent、--bg、--surface、--border、--radius-*、--shadow-*、--sidebar-width、--bottom-nav-h 等），添加 `.wb-sidebar`、`.wb-sidebar-overlay`、`.wb-bottom-nav`、`.wb-hamburger`、`.card-dark-hd`、`.btn-accent` 等全局辅助类，以及响应式断点（≤768px 显示汉堡按钮，≤600px 侧边栏变抽屉 + 底部 Tab 显示）
- `packages/web/src/pages/workbench.vue`：顶部 Tab 导航改为左侧边栏布局，移动端支持抽屉（translateX + overlay）+ 底部 Tab 导航（同步/网盘/传输/设置）
- `packages/web/src/pages/login.vue`：磨砂玻璃登录卡片（`min(360px, 90vw)`），accent 色登录按钮，响应式 clamp 字号
- `packages/web/src/pages/workbench/sync.vue`：卡片头从 bg-gray-800 改为 `linear-gradient(135deg, var(--card-hd-from), var(--card-hd-to))`，卡片加 border-radius + shadow，空状态改为 icon+文字
- `packages/web/src/pages/workbench/disk/list.vue`：工具栏使用 surface/border CSS 变量，面包屑升级为 breadcrumb-seg 样式
- `packages/web/src/ui-components/progress.vue`：支持 `type` prop（upload=橙色，download/storage=accent 蓝），进度条高度从 h-8 改为 h-6
- `packages/web/src/components/widget-task.vue`：卡片边框/背景改用 CSS 变量，Progress 传入 type prop
- `packages/web/src/components/modal-mod-config.vue`：桌面端表单改为双列 grid 布局（config-form-grid）

