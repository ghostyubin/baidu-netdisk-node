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
RUN pnpm install --frozen-lockfile --registry=${NPM_REGISTRY}

# ── 再复制源码 ──
COPY . .

# 构建
RUN pnpm build xth

# ────────────────────────────────────────────
# Stage 2: 生产镜像（只保留运行时必要文件）
# ────────────────────────────────────────────
FROM node:22.18.0-alpine

ARG NPM_REGISTRY=https://registry.npmjs.org

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk add --no-cache tzdata dumb-init && \
    npm install -g pnpm@^10 --registry=${NPM_REGISTRY}

WORKDIR /baidu-netdisk-xth

# 从构建阶段拷贝产物
COPY --from=builder /app/packages/xth/dist ./

# 只安装生产依赖
RUN pnpm install --prod --registry=${NPM_REGISTRY}

CMD ["dumb-init", "pnpm", "run", "start"]
