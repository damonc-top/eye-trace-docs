# Eye Trace Server 部署指南

> 适用版本:本仓 `master` 分支当前快照,后端位于 `backend/`。
> 目标读者:本机/服务器/Windows 上手动部署一次的工程师。
> 文档结构:先讲整体流程(所有人都要看),再按 macOS / Linux / Windows 分平台步骤。

---

## 0. 一图看懂流程

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 装工具:Go 1.22+、MySQL 8.0+、可选:migrate CLI、oapi-codegen  │
│  2. 拿源码:git clone(或拷贝本仓)                                  │
│  3. 装 server 依赖:go mod tidy                                   │
│  4. 起 MySQL:docker run 或本机服务                                │
│  5. 改 config.yaml(或纯环境变量):DSN、Storage 路径、JWT_SECRET   │
│  6. 跑 migration + seed                                           │
│  7. 编 server:go build -o bin/server ./cmd/server                │
│  8. 跑 server:./bin/server                                        │
│  9. 自测:curl /healthz + 4 个公开端点                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. 准备:系统级依赖

无论哪个平台,先确认这几样:

| 依赖 | 最低版本 | 用途 | 必需 |
|---|---|---|:---:|
| **Go** | 1.22+(建议 1.24) | 编译 server | ✅ |
| **MySQL** | 8.0+ | 持久化 | ✅ |
| **git** | 任意 | 拉代码 | ✅ |
| **curl** | 任意 | smoke 测试 | ✅ |
| **oapi-codegen** | v2.7.1 | 重新生成 api.gen.go(仅改 OpenAPI 时) | 🟡 |
| **golang-migrate** | v4.17+ | 跑 migration | 🟡(可用 `go run` 替代) |

> **不要 MySQL 行不行?** 一期不行(代码直接连 MySQL)。最小测试可以用 `go test ./...`(用 in-memory fake,不需要 DB)。

---

## 2. 平台一:macOS(本机开发 / 装 Intel/M1 Mac)

### 2.1 装系统级工具

```bash
# Homebrew 一次性装好
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install go mysql git curl
# MySQL 起服务:
brew services start mysql
mysql_secure_installation  # 设 root 密码,后面 DSN 用

# Go 装完后,把 GOPATH/bin 加进 PATH(本机编译的工具都在这)
echo 'export PATH=$HOME/go/bin:$PATH' >> ~/.zshrc
source ~/.zshrc
```

> **如果用 Docker 跑 MySQL(更省心):**
> ```bash
> brew install --cask docker
> docker run -d --name eye-trace-mysql -p 3306:3306 \
>   -e MYSQL_ROOT_PASSWORD=root \
>   -e MYSQL_DATABASE=eye_trace \
>   -e MYSQL_USER=eye_trace \
>   -e MYSQL_PASSWORD=eye_trace \
>   mysql:8.0 --default-authentication-plugin=mysql_native_password
> ```

### 2.2 拿代码

```bash
cd ~/github  # 或你想放的位置
git clone <repo-url> eye-trace
cd eye-trace/eye-trace-server/backend
```

### 2.3 装 Go 工具(go install)

```bash
# oapi-codegen:改 OpenAPI 后需要跑 codegen 时用
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.7.1

# golang-migrate:跑 migration CLI
go install github.com/golang-migrate/migrate/v4/cmd/migrate@v4.17.1

# 验证
which oapi-codegen migrate
# 期望:两个都打印 $HOME/go/bin/ 路径
```

### 2.4 创建数据库

```bash
# 如果用本机 brew mysql:
mysql -uroot -p
> CREATE DATABASE eye_trace CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
> CREATE USER 'eye_trace'@'localhost' IDENTIFIED BY 'eye_trace';
> GRANT ALL ON eye_trace.* TO 'eye_trace'@'localhost';
> FLUSH PRIVILEGES;
> EXIT;

# 如果用 docker:上面 -e 已经建好,直接进下一步。
```

### 2.5 改 config / 准备环境变量

**两种方式二选一,推荐环境变量(12-factor):**

```bash
# 必须:JWT 签名密钥(≥ 32 字节随机,生产环境用密钥管理工具)
export JWT_SECRET="$(openssl rand -hex 32)"

# 必须:MySQL DSN(parseTime 让 driver 解析 time.Time)
export DB_DSN="eye_trace:eye_trace@tcp(127.0.0.1:3306)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4"

# 可选:监听地址(默认 :8080)
export SERVER_ADDR=":8080"

# 可选:本地资产根目录(默认 var/assets,seed 脚本会创建占位文件)
export STORAGE_LOCALROOT="var/assets"
```

**或者写 `config.yaml`**(会被 `os.Getenv` 覆盖):

```yaml
# backend/config.yaml
server:
  addr: ":8080"
  readTimeout: 30s
  writeTimeout: 30s
  shutdownTimeout: 10s
db:
  dsn: "eye_trace:eye_trace@tcp(127.0.0.1:3306)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4"
  maxOpenConns: 20
  maxIdleConns: 5
  connMaxLifetime: 30m
assets:
  baseUrl: "/api/v1/assets"
storage:
  backend: "local"
  localRoot: "var/assets"
```

### 2.6 装 Go 依赖 + 跑 migration + seed

```bash
cd ~/github/eye-trace/eye-trace-server/backend

# 1) 同步 go.sum
go mod tidy

# 2) 跑 migration(创建 6 张表)
migrate -path migrations -database "mysql://$DB_DSN" up
# 期望输出:migrations/000007, 000008, 000009, 000010, 000011 ... all up

# 3) 灌 demo 数据(6 个 slot + 30 个 model + 100+ 个 asset)
make seed
# 等价于:
#   ./scripts/gen_placeholder_assets.sh var/assets
#   mysql $DB_DSN < migrations/seed_assets.sql
#   mysql $DB_DSN < migrations/seed_models.sql
#   mysql $DB_DSN < migrations/seed_home_content_slots.sql
```

### 2.7 编 + 启动 server

```bash
# 编译
go build -o bin/server ./cmd/server

# 启动(前台跑,看日志)
./bin/server
# 期望日志:
#   {"time":"...","level":"INFO","msg":"db ready"}
#   {"time":"...","level":"INFO","msg":"http listening","addr":":8080"}

# 或后台跑
nohup ./bin/server > server.log 2>&1 &
echo $! > server.pid
# 停止:kill $(cat server.pid)
```

### 2.8 Smoke 自测

```bash
# 1) 健康检查
curl -i http://localhost:8080/healthz
# 期望:200,body "ok"

# 2) 首页内容
ETAG=$(curl -sI http://localhost:8080/api/v1/content/home | grep -i etag | tr -d '\r' | awk '{print $2}')
echo "ETag: $ETAG"
# 期望有 ETag,Cache-Control 含 max-age=60

# 3) 模型列表
curl -s "http://localhost:8080/api/v1/models?limit=3" | head -c 500
# 期望:items 数组,variant 含 single/preview-deck/gallery,nextCursor 非空

# 4) 资产下载
curl -i http://localhost:8080/api/v1/assets/1/content | head -5
# 期望:200,Content-Type: image/png,Content-Length: 67

# 5) 304 测试(用首页 ETag)
curl -i -H "If-None-Match: $ETAG" http://localhost:8080/api/v1/content/home | head -1
# 期望:304

# 6) 登录(需要先有 users 表数据,见下)
# INSERT 一个 admin(密码 admin123 的 bcrypt hash):
HASH=$(go run - <<'EOF'
package main
import ("fmt"; "golang.org/x/crypto/bcrypt")
func main() {
  b, _ := bcrypt.GenerateFromPassword([]byte("admin123"), 12)
  fmt.Println(string(b))
}
EOF
)
mysql $DB_DSN -e "INSERT INTO users (email, password_hash, role, display_name) VALUES ('admin@local', '$HASH', 'admin', 'Admin');"

curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@local","password":"admin123"}' | head -c 300
# 期望:accessToken + refreshToken + expiresIn: 1800
```

### 2.9 验证(可选)跑测试

```bash
# 全套测试,36 个,不需要 MySQL(用 in-memory fake)
go test ./...
# 期望:"N passed in M packages"
```

---

## 3. 平台二:Linux(Ubuntu 22.04 / Debian 12 / CentOS 8 等)

### 3.1 装系统级工具

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y build-essential curl git mysql-server

# 装 Go 1.24(用官方 tarball,apt 的版本太老)
wget https://go.dev/dl/go1.24.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.24.3.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
go version  # 期望 go1.24

# MySQL 起服务
sudo systemctl start mysql
sudo mysql_secure_installation

# 创建 DB + 用户
sudo mysql <<EOF
CREATE DATABASE eye_trace CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'eye_trace'@'localhost' IDENTIFIED BY 'eye_trace';
GRANT ALL ON eye_trace.* TO 'eye_trace'@'localhost';
FLUSH PRIVILEGES;
EOF
```

> **或者用 Docker:**
> ```bash
> sudo apt install -y docker.io
> sudo systemctl start docker
> sudo docker run -d --name eye-trace-mysql --restart=unless-stopped -p 127.0.0.1:3306:3306 \
>   -e MYSQL_ROOT_PASSWORD=root \
>   -e MYSQL_DATABASE=eye_trace \
>   -e MYSQL_USER=eye_trace \
>   -e MYSQL_PASSWORD=eye_trace \
>   mysql:8.0 --default-authentication-plugin=mysql_native_password
> ```

### 3.2 拿代码 + 装 Go 工具

```bash
cd /opt  # 或你想放的位置
sudo git clone <repo-url> eye-trace
sudo chown -R $USER:$USER eye-trace
cd eye-trace/eye-trace-server/backend

# 装 oapi-codegen + migrate
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.7.1
go install github.com/golang-migrate/migrate/v4/cmd/migrate@v4.17.1
which oapi-codegen migrate
```

### 3.3 环境变量 + migration + seed + 启动

```bash
cd /opt/eye-trace/eye-trace-server/backend

# 环境变量(生产用密钥管理,这里只是本机)
export JWT_SECRET="$(openssl rand -hex 32)"
export DB_DSN="eye_trace:eye_trace@tcp(127.0.0.1:3306)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4"
export SERVER_ADDR=":8080"

# 同步 + migrate + seed + build
go mod tidy
migrate -path migrations -database "mysql://$DB_DSN" up
make seed
go build -o bin/server ./cmd/server
```

### 3.4 用 systemd 托管(生产推荐)

```bash
# 1) 创建 /etc/eye-trace-server/env(避免 shell history 漏 JWT_SECRET)
sudo tee /etc/eye-trace-server/env > /dev/null <<'EOF'
JWT_SECRET=<从上面 export 复制过来>
DB_DSN=eye_trace:eye_trace@tcp(127.0.0.1:3306)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4
SERVER_ADDR=:8080
STORAGE_LOCALROOT=/var/lib/eye-trace/assets
EOF
sudo chmod 600 /etc/eye-trace-server/env

# 2) 资产目录
sudo mkdir -p /var/lib/eye-trace/assets
sudo chown -R eye-trace:eye-trace /var/lib/eye-trace  # 见下创建用户

# 3) 创建专用用户
sudo useradd -r -s /bin/false -d /opt/eye-trace eye-trace

# 4) systemd unit
sudo tee /etc/systemd/system/eye-trace-server.service > /dev/null <<'EOF'
[Unit]
Description=Eye Trace API Server
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=eye-trace
Group=eye-trace
WorkingDirectory=/opt/eye-trace/eye-trace-server/backend
EnvironmentFile=/etc/eye-trace-server/env
ExecStart=/opt/eye-trace/eye-trace-server/backend/bin/server
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# 5) 启动
sudo systemctl daemon-reload
sudo systemctl enable --now eye-trace-server
sudo systemctl status eye-trace-server

# 6) 验证
curl -i http://localhost:8080/healthz
```

### 3.5 看日志

```bash
sudo journalctl -u eye-trace-server -f        # 跟踪
sudo journalctl -u eye-trace-server --since "5 min ago"
```

---

## 4. 平台三:Windows 10 / Windows 11

> 警告:Windows 部署是开发/演示用,生产建议 Linux。路径分隔符、信号处理、systemd 都没 Linux 顺。

### 4.1 装系统级工具

1. **Go 1.24+**:到 https://go.dev/dl/ 下载 `go1.24.3.windows-amd64.msi`,双击安装。
2. **MySQL 8.0+**:到 https://dev.mysql.com/downloads/installer/ 下载 `mysql-installer-community-8.0.x.msi`,选 "Server only" 安装,记下 root 密码。
3. **Git for Windows**:到 https://git-scm.com/download/win 下载安装。
4. **可选 — Docker Desktop**:到 https://www.docker.com/products/docker-desktop/ 安装,用 Docker 跑 MySQL 更省心。

### 4.2 准备 PowerShell 环境

用 **PowerShell**(不是 cmd):

```powershell
# 1) 创建工作目录
New-Item -ItemType Directory -Path "$env:USERPROFILE\github" -Force
cd $env:USERPROFILE\github

# 2) 拉代码
git clone <repo-url> eye-trace
cd eye-trace\eye-trace-server\backend

# 3) 装 Go 工具
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.7.1
go install github.com/golang-migrate/migrate/v4/cmd/migrate@v4.17.1

# 4) GOPATH/bin 加 PATH(本会话有效,新开窗口要重设)
$env:Path = "$env:USERPROFILE\go\bin;$env:Path"
where.exe migrate   # 期望:显示 $env:USERPROFILE\go\bin\migrate.exe
```

**永久加 PATH**(图形界面):
- Win+R → `sysdm.cpl` → 高级 → 环境变量 → 用户 Path → 编辑 → 新建 → `%USERPROFILE%\go\bin` → 确定 → 重开 PowerShell

### 4.3 创建数据库(用本机 MySQL Installer 装的)

打开 **MySQL 8.0 Command Line Client**(开始菜单找),输 root 密码:

```sql
CREATE DATABASE eye_trace CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'eye_trace'@'localhost' IDENTIFIED BY 'eye_trace';
GRANT ALL ON eye_trace.* TO 'eye_trace'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

或者用 MySQL Workbench(图形)同样能建。

### 4.4 环境变量

```powershell
# 临时(当前 PowerShell 会话)
$env:JWT_SECRET = -join ((1..32) | ForEach-Object { [char](Get-Random -Minimum 48 -Maximum 122) })
$env:DB_DSN = "eye_trace:eye_trace@tcp(127.0.0.1:3306)/eye_trace?parseTime=true&loc=Local&charset=utf8mb4"
$env:SERVER_ADDR = ":8080"

# 查看
Get-ChildItem Env: | Where-Object { $_.Name -in "JWT_SECRET","DB_DSN","SERVER_ADDR" }
```

**永久**:`sysdm.cpl` → 高级 → 环境变量 → 新建系统变量(或用户变量)。

### 4.5 跑 migration + seed

```powershell
cd $env:USERPROFILE\github\eye-trace\eye-trace-server\backend

# 同步
go mod tidy

# Migration(migrate CLI)
migrate -path migrations -database "mysql://$env:DB_DSN" up

# Seed
.\scripts\gen_placeholder_assets.sh var\assets    # 可能要 bash(Git Bash / WSL)
# 或者用 PowerShell 改写
New-Item -ItemType Directory -Path "var\assets\cover","var\assets\avatar","var\assets\gallery" -Force
# 占位文件:gen_placeholder_assets.sh 是 bash,Windows 用 Git Bash 运行最方便

# 跑 SQL seed
Get-Content migrations\seed_assets.sql,migrations\seed_models.sql,migrations\seed_home_content_slots.sql |
  mysql eye_trace -ueye_trace -peye_trace
# 或者用 mysqldump-style:`<file.sql mysql -ueye_trace -peye_trace eye_trace`
```

> **WSL 推荐**:Windows 上跑 Linux 二进制最省心(免去路径问题)。`wsl --install` 进 Ubuntu,然后按"平台二:Linux"做。

### 4.6 编 + 启动

```powershell
# 编
go build -o bin\server.exe .\cmd\server

# 启动(前台)
.\bin\server.exe

# 后台跑(简单方式:start-job)
Start-Process -FilePath ".\bin\server.exe" -WorkingDirectory $PWD -RedirectStandardOutput "server.log" -RedirectStandardError "server.err.log"
# 停止:Get-Process server | Stop-Process -Force
```

### 4.7 用 NSSM 注册成 Windows 服务(可选,生产化)

```powershell
# 1) 下载 NSSM(https://nssm.cc/download)
# 2) 解压到 C:\nssm
# 3) 注册服务
cd C:\nssm\win64
.\nssm.exe install EyeTraceServer "C:\Users\<you>\github\eye-trace\eye-trace-server\backend\bin\server.exe"
.\nssm.exe set EyeTraceServer AppDirectory "C:\Users\<you>\github\eye-trace\eye-trace-server\backend"
.\nssm.exe set EyeTraceServer AppEnvironmentExtra "JWT_SECRET=<your>^DB_DSN=eye_trace:..."   # 多个用 ^
.\nssm.exe set EyeTraceServer DisplayName "Eye Trace Server"
.\nssm.exe set EyeTraceServer Start SERVICE_AUTO_START

# 4) 启动
.\nssm.exe start EyeTraceServer
.\nssm.exe status EyeTraceServer
```

---

## 5. 通用:跑测试(不需 MySQL)

任意平台:

```bash
cd <repo>/eye-trace/eye-trace-server/backend
go test ./...
# 期望:36 passed in 14 packages
```

只跑某一类:

```bash
go test ./internal/auth/...     # 8 个
go test ./internal/model/...    # 10 个
go test -v -run TestHTTP ./internal/api/handler/...   # 8 个 HTTP 集成
```

---

## 6. 通用:常见问题

### 6.1 `JWT_SECRET required; refusing to start without it`

服务拒绝裸奔。设环境变量 `JWT_SECRET=$(openssl rand -hex 32)`(≥ 32 字节随机串)。

### 6.2 `failed to connect to MySQL`

- 检查 DSN:`mysql -u eye_trace -p -h 127.0.0.1 eye_trace` 能不能登
- 检查端口:`nc -z 127.0.0.1 3306`(macOS/Linux)或 `Test-NetConnection 127.0.0.1 -Port 3306`(PowerShell)
- 防火墙:本地部署一般无碍;云服务器要开 3306 入站
- Docker 用户:`127.0.0.1:3306` 还是 `host.docker.internal:3306`?

### 6.3 `bearerAuthContextKey undefined`

`make gen` 后没跑 patch。看 `scripts/patch-gen.sh` 是否被 Makefile 串接。手工修法:在 `internal/api/api.gen.go` imports 块后加:

```go
type bearerAuthContextKey string

const (
    BearerAuthScopes bearerAuthContextKey = "bearerAuth.Scopes"
)
```

### 6.4 `tls: failed to verify certificate` (Go 工具链下载)

sandbox / 公司内网常见。设 `GOPROXY=https://goproxy.cn,direct` + `GOSUMDB=off`(只在开发机;生产 build 仍走 sum 校验)。

### 6.5 seed 后 `/api/v1/auth/login` 仍 401

种子只灌 `models` / `assets` / `home_content_slots`,**没灌 users**。手动插一条:

```bash
# 计算 bcrypt hash(Go 一行)
go run - <<'EOF'
package main
import ("fmt"; "golang.org/x/crypto/bcrypt")
func main() {
  b, _ := bcrypt.GenerateFromPassword([]byte("admin123"), 12)
  fmt.Println(string(b))
}
EOF

# 拿到 hash 后插库
mysql $DB_DSN -e "INSERT INTO users (email, password_hash, role) VALUES ('admin@local', '\$2a\$12\$...', 'admin');"
```

### 6.6 `/api/v1/assets/{id}/content` 404

seed 脚本(`scripts/gen_placeholder_assets.sh`)没跑,或 storage.localRoot 路径错了。检查:

```bash
ls var/assets/cover/1.png
# 期望:67 字节 PNG

# 或检查 config.yaml 的 storage.localRoot
```

### 6.7 启动后 502 / 502 Bad Gateway(Nginx 后面)

加反代配置:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 90s;     # SSE 兼容
}
```

---

## 7. 升级 / 回滚

```bash
# 升级
cd <repo>/eye-trace/eye-trace-server
git pull
cd backend
go mod tidy
go build -o bin/server ./cmd/server
# 重启(对应平台的命令)
sudo systemctl restart eye-trace-server   # Linux
# 或 stop + start bin\server.exe          # Windows

# 跑新 migration(若有)
migrate -path migrations -database "mysql://$DB_DSN" up

# 回滚 1 个 migration
migrate -path migrations -database "mysql://$DB_DSN" down 1
```

---

## 8. 性能 / 监控(可选,生产化)

| 项 | 工具 | 备注 |
|---|---|---|
| 进程监控 | systemd / supervisor / pm2 | 进程挂了自动拉起 |
| 日志聚合 | journalctl + journalbeat → ES | 已有 slog JSON 输出 |
| 指标暴露 | 加 `github.com/prometheus/client_golang` + `/metrics` | 未实现,需求来再加 |
| 链路追踪 | OpenTelemetry SDK | 未实现,先放 |
| 限流 | 已有 IP token bucket | 多实例换 Redis Lua |
| 数据库 | MySQL 8.0,`FOR UPDATE SKIP LOCKED` 已支持 | 横向扩直接用 |

---

## 9. 端口与资源

| 资源 | 默认 | 配置 |
|---|---|---|
| HTTP 监听 | `:8080` | `SERVER_ADDR` |
| MySQL | `3306` | `DB_DSN` |
| 资产文件 | `var/assets/` | `STORAGE_LOCALROOT` |
| 限流 | 120 req/min/IP | `internal/api/middleware/middleware.go:RateLimit(120, time.Minute, nil)` |
| 内存 | < 50MB(单实例,无负载) | 取决于 job hub 大小 |

---

## 10. 链接到其他文档

- `SESSION_RECAP.md` — 整个项目的历史决策与文件清单
- `docs/oapi-codegen.md` — 改 OpenAPI 后怎么重新 codegen
- `eye-trace-docs/server_guide.md` — server 架构总纲(为什么这么设计)
- `eye-trace-docs/client_data_split.md` — client/server 数据拆分依据

---

**部署完跑通 smoke,贴回 `/healthz` + `/api/v1/content/home` 的 ETag,确认无误。**
