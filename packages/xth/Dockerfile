# ────────────────────────────────────────────
# Stage 1: 安装依赖 + 构建
# ────────────────────────────────────────────
FROM node:22.18.0-alpine AS builder

ARG NPM_REGISTRY=https://registry.npmjs.org

WORKDIR /app

# 换源 & 安装系统工具
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk add --no-cache tzdata && \
    npm install -g pnpm@^10 --registry=${NPM_REGISTRY}

# ── 先只复制 lockfile + 所有 package.json，让依赖层可以被缓存 ──
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/api/package.json   ./packages/api/
COPY packages/sdk/package.json   ./packages/sdk/
COPY packages/srv/package.json   ./packages/srv/
COPY packages/web/package.json   ./packages/web/
COPY packages/xth/package.json   ./packages/xth/

# 安装全量依赖（只要 lockfile/package.json 不变，这层直接命中缓存）
# --ignore-scripts 跳过 postinstall (simple-git-hooks) 和 preinstall (only-allow) 避免报错
RUN pnpm install --frozen-lockfile --ignore-scripts --registry=${NPM_REGISTRY}

# ── 再复制源码 ──
COPY . .

# 构建
RUN pnpm build xth

# 用 pnpm deploy 把 xth 的生产依赖（含 workspace 包）打包到独立目录
# 这会自动将 workspace:* 解析为实际包，无需保留整个 monorepo
RUN pnpm --filter baidu-netdisk-xth deploy --prod /deploy/xth

# 把编译产物覆盖进 deploy 目录（deploy 命令只含 node_modules，不含 dist）
RUN cp -r /app/packages/xth/dist/. /deploy/xth/

# ────────────────────────────────────────────
# Stage 2: 生产镜像（只保留运行时必要文件）
# ────────────────────────────────────────────
FROM node:22.18.0-alpine

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk add --no-cache tzdata dumb-init

WORKDIR /baidu-netdisk-xth

# 从 builder 拷贝打包好的完整目录（源码 + node_modules，无 workspace 问题）
COPY --from=builder /deploy/xth ./

CMD ["dumb-init", "node", "./src/app.js"]
