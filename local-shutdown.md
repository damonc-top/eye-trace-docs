# EyeTraceAI 本地停止步骤

> 目标:按 client -> admin-schema-viewer -> server -> MySQL -> MinIO 的反向顺序停止本地开发栈。
> 以下命令按 Codex/RTK 环境写出;普通终端可去掉 `rtk` 前缀。

## 1. 停止 client

如果是在前台终端启动的 `npm run dev`,在该终端按 `Ctrl+C`。

如果找不到终端,按端口查 PID 后停止:

```bash
rtk lsof -nP -iTCP:1420 -sTCP:LISTEN
rtk kill <PID>
```

## 2. 停止 server

如果是在前台终端启动的 `go run ./cmd/server`,在该终端按 `Ctrl+C`。

如果找不到终端,按端口查 PID 后停止:

```bash
rtk lsof -nP -iTCP:8080 -sTCP:LISTEN
rtk kill <PID>
```

## 2b. 停止 admin-schema-viewer

admin-schema-viewer(`make admin` / `make admin-run`,监听 `127.0.0.1:8091`)是本地开发栈标准组件,
跟随 startup 必起,故此处必停。前台启动的在后端终端按 `Ctrl+C`;dev 模式下前端 Vite(`:5173`)同理。

如果找不到终端,按端口查 PID 后停止:

```bash
rtk lsof -nP -iTCP:8091 -sTCP:LISTEN
rtk kill <PID>
rtk lsof -nP -iTCP:5173 -sTCP:LISTEN   # 若用了 Vite dev 前端
rtk kill <PID>
```

## 3. 停止 MySQL

本地开发默认 MySQL 是项目内实例,数据目录为 `eye-trace-db/mysql-data`,监听 `127.0.0.1:3307`。
停止该实例:

```bash
cd /Users/mac/github/eye-trace-ai
rtk ./eye-trace-db/eyetrace-mysql.sh stop
```

如果只是要确认状态:

```bash
rtk ./eye-trace-db/eyetrace-mysql.sh status
rtk lsof -nP -iTCP:3307 -sTCP:LISTEN
```

不要在 EyeTraceAI shutdown 流程里停止 Homebrew 全局 MySQL(`brew services stop mysql`),
除非你明确回滚到旧的 `/usr/local/var/mysql` 数据源并确认没有其他项目使用它。

## 4. 停止 MinIO

数据目录(从仓库根):`eye-trace-res/minio-data`(仓库内,与 MySQL 的 `eye-trace-db/mysql-data/` 同模式)。

如果是在前台终端启动的 `minio server`,在该终端按 `Ctrl+C`。

如果找不到终端,按端口查 PID 后停止:

```bash
rtk lsof -nP -iTCP:9000 -sTCP:LISTEN
rtk kill <PID>
```

## 5. 验证停止结果

```bash
rtk lsof -nP -iTCP:1420 -sTCP:LISTEN
rtk lsof -nP -iTCP:8080 -sTCP:LISTEN
rtk lsof -nP -iTCP:8091 -sTCP:LISTEN   # admin-schema-viewer 后端
rtk lsof -nP -iTCP:5173 -sTCP:LISTEN   # 若用了 Vite dev 前端
rtk lsof -nP -iTCP:3307 -sTCP:LISTEN
rtk lsof -nP -iTCP:9000 -sTCP:LISTEN
```

上述命令没有输出时,client、server、admin-schema-viewer、项目内 MySQL、MinIO 已停止。
