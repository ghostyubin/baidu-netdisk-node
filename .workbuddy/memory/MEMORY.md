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
  - 新增 GHA 构建缓存（cache-from/cache-to type=gha）加速后续构建
  - 触发方式：手动 workflow_dispatch + push tag（`srv-v*` / `xth-v*`）
