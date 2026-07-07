# EyeTraceAI 本地启动步骤

> 目标:按顺序启动 MinIO -> MySQL -> server -> admin-schema-viewer -> client,最后用 Chrome 打开 client。
> 以下命令按 Codex/RTK 环境写出;普通终端可去掉 `rtk` 前缀。

## 1. 启动 MinIO

在第一个终端执行并保持运行:

```bash
rtk mkdir -p eye-trace-res/minio-data
cd /Users/mac/github/eye-trace-ai
MINIO_ROOT_USER=eye-trace-admin \
MINIO_ROOT_PASSWORD=eye-trace-secret-32bytes-1234567890 \
rtk minio server ./eye-trace-res/minio-data \
  --address 127.0.0.1:9000 \
  --console-address 127.0.0.1:9001
```

> 数据目录约定:MinIO 数据放在仓库内 `eye-trace-res/minio-data/`(与 MySQL 的 `eye-trace-db/mysql-data/` 同模式),根 `.gitignore` 已排除。若是从旧 `~/.local/minio-data` 迁来的,见 `eye-trace-res/README.md`。

初始化 alias 和 bucket:

```bash
rtk mc alias set local http://127.0.0.1:9000 eye-trace-admin eye-trace-secret-32bytes-1234567890
rtk mc mb --ignore-existing local/eye-trace-assets
rtk mc anonymous set download local/eye-trace-assets
```

## 2. 启动 MySQL

本地开发默认使用项目内 MySQL 数据源:

- 数据目录:`/Users/mac/github/eye-trace-ai/eye-trace-db/mysql-data`
- 运行目录:`/Users/mac/github/eye-trace-ai/eye-trace-db/run`
- 监听端口:`127.0.0.1:3307`

启动:

```bash
cd /Users/mac/github/eye-trace-ai
rtk ./eye-trace-db/eyetrace-mysql.sh start
```

验证:

```bash
rtk lsof -nP -iTCP:3307 -sTCP:LISTEN
rtk ./eye-trace-db/eyetrace-mysql.sh ping
```

不要用 `brew services start mysql` 作为 EyeTraceAI 默认数据源;Homebrew 的 `/usr/local/var/mysql`
是全局 MySQL 服务,可能被其他项目共享。只有要回滚旧数据源时才手动启动它。

如果库或账号不存在,按 `eye-trace-docs/DEPLOY.md` 的 MySQL 初始化段处理。

## 3. 启动 server

在第二个终端执行并保持运行:

```bash
cd /Users/mac/github/eye-trace-ai/eye-trace-server/backend
JWT_SECRET=dev-jwt-secret-please-change-32bytes \
DB_DSN='eye_trace:eye_trace@tcp(127.0.0.1:3307)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4' \
SERVER_ADDR=':8080' \
STORAGE_BACKEND=s3 \
STORAGE_S3_ENDPOINT=127.0.0.1:9000 \
STORAGE_S3_BUCKET=eye-trace-assets \
STORAGE_S3_ACCESS_KEY=eye-trace-admin \
STORAGE_S3_SECRET_KEY=eye-trace-secret-32bytes-1234567890 \
STORAGE_S3_REGION=us-east-1 \
STORAGE_S3_USE_SSL=false \
STORAGE_S3_PUBLIC_URL=http://127.0.0.1:9000/eye-trace-assets \
ASSETS_PUBLIC_BASE=http://127.0.0.1:8080 \
rtk go run ./cmd/server
```

验证:

```bash
rtk curl -fsS http://127.0.0.1:8080/healthz
```

期望输出 `ok`。

## 4. 启动 admin-schema-viewer

内部只读 DB 结构浏览器,复用同一个 MySQL(3307),独立端口 `127.0.0.1:8091`。
用途是核对表结构/样本/migration 是否符合设计预期,见 `eye-trace-docs/admin-schema-viewer.md`。

**后端**(必起,第四个终端,后台保持运行):

```bash
cd /Users/mac/github/eye-trace-ai/eye-trace-server/backend
DB_DSN='eye_trace:eye_trace@tcp(127.0.0.1:3307)/eye_trace?parseTime=true&loc=UTC&charset=utf8mb4' \
ADMIN_ADDR=127.0.0.1:8091 \
rtk go run ./cmd/admin-server
```

验证:

```bash
rtk curl -fsS http://127.0.0.1:8091/api/schema/tables | rtk head -c 80
```

期望返回 JSON `{"tables":[{"name":"..."}...]}`。

**前端**(二选一):

- dev 模式(热更新):第五个终端 `cd eye-trace-server/admin && rtk npm run dev`,浏览器开 `http://localhost:5173`(Vite 自动 proxy `/api` → `:8091`)。首次需 `rtk npm install`。
- 单进程托管(无需 Vite,推荐日常用):`make admin-build` 构建 `../admin/dist`,再 `make admin-run`,浏览器开 `http://127.0.0.1:8091`。

> admin 后端是本地开发栈的标准组件,与 client/server 同级必起;仅前端 dev/server 形态可选。

## 5. 启动 client

在第三个终端执行并保持运行:

```bash
cd /Users/mac/github/eye-trace-ai/eye-trace-client
rtk npm run dev
```

验证:

```bash
rtk curl -fsS -I http://127.0.0.1:1420/
rtk curl -fsS http://127.0.0.1:1420/api/v1/content/home
```

## 6. 用 Chrome 打开 client

```bash
rtk open -a "Google Chrome" http://127.0.0.1:1420/
```

验收点:页面可见,首页状态显示 `home: server`,并能加载来自 `127.0.0.1:9000` 或 `127.0.0.1:8080/api/v1/assets/...` 的图片。
