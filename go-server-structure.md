# Go Server 目录结构与架构 — 架构指导

> server 定位:**协议翻译层 + 状态权威层**,不是无脑转发器。
> 对外 REST+SSE(契约见 `openapi-skeleton.md`),对内 WebSocket 连 ComfyUI worker。
> 单独部署到 Linux。

---

## 0. 指导原则(次级模型必须遵守)

1. **分层单向依赖**:`handler → service → (comfy-client | repo)`。handler 只做 HTTP 编解码,**零业务逻辑**;service 持有业务规则;repo 只碰 DB。严禁 handler 直接调 comfy-client 或 SQL。
2. **handler 薄**:每个 handler ≤ 30 行 —— 解析请求 → 调 service → 写响应。实现 `api.gen.go` 生成的接口。
3. **comfy-user 边界封死**:只有 `comfy` 包能注入 `comfy-user` header,且值只来自 `BindingService`。其它任何包都拿不到、传不进这个 header。
4. **DB 是 job 权威源**:每次 ComfyUI WS 事件都先写 DB,再 fanout 到 SSE。SSE 断了不丢状态(client 可 `GET /workflow-runs/{id}` 补偿)。
5. **业务配置 ≠ App 配置**:`pkg/config` 只放进程配置(端口、DB DSN、worker 地址池);模型目录/首页内容是**数据**,进 DB,走 service。

---

## 1. 目录结构

```
eye-trace-server/backend/
├── cmd/
│   └── server/
│       └── main.go                  # 装配:读 config → 建 DB/worker 池 → 挂路由 → 启动
├── internal/                        # 不对外暴露的业务实现
│   ├── api/
│   │   ├── api.gen.go               # oapi-codegen 生成(types + chi-server 接口),勿手改
│   │   ├── handler/                 # 实现生成接口,薄 HTTP 层
│   │   │   ├── run.go               #   POST/GET/DELETE /workflow-runs
│   │   │   ├── run_events.go        #   GET /workflow-runs/{id}/events  ← SSE
│   │   │   ├── model.go             #   /models
│   │   │   ├── asset.go             #   /assets
│   │   │   ├── content.go           #   /content/home
│   │   │   └── auth.go              #   /auth, /me
│   │   └── middleware/
│   │       ├── auth.go              # 校验 JWT/api-key → 注入 actor 到 ctx
│   │       ├── tenant.go            # 校验 X-Tenant-Id + 成员关系 → 注入 tenant 到 ctx
│   │       ├── rbac.go              # 角色 → 操作鉴权
│   │       ├── ratelimit.go         # 租户级限流/并发上限
│   │       └── reqid.go             # X-Request-Id 注入 + 结构化日志
│   ├── auth/
│   │   └── service.go               # 登录、签发/刷新 token、api-key 校验
│   ├── tenant/
│   │   ├── service.go               # 租户、成员、角色 CRUD
│   │   └── repo.go
│   ├── comfy/                       # ★ 与 ComfyUI worker 交互的全部逻辑(隔离边界)
│   │   ├── client.go                # TenantAwareComfyClient:出站调用自动注入 comfy-user
│   │   ├── binding.go               # BindingService:lazy POST /api/users,存 tenant_comfy_bindings
│   │   ├── wspool.go                # ★ 每 client_id 一条常驻 WS,消费 status/progress/executing
│   │   ├── dispatch.go              # WS 事件 → 按 prompt_id 路由 → 写 DB → 推 hub
│   │   └── pool.go                  # worker 实例池(多 ComfyUI 实例的健康检查/调度)
│   ├── job/
│   │   ├── service.go               # WorkflowRunService:提交/查询/取消;prompt_id↔jobId 映射
│   │   ├── repo.go                  # comfy_jobs 表(强制 tenant_id + actor_user_id)
│   │   └── hub.go                   # ★ SSE fanout:jobId → 订阅者集合,事件广播
│   ├── asset/
│   │   ├── service.go               # TenantAssetService:上传/列表/删除,owner 字段强制
│   │   └── repo.go                  # comfy_assets 表
│   ├── model/
│   │   └── service.go               # 模型目录(读 DB / 缓存 ComfyUI /api/models 结果)
│   ├── content/
│   │   └── service.go               # 首页内容块(banner/recommend/tabs),形状对齐 client types.ts
│   ├── quota/
│   │   └── service.go               # 配额:并发数、存储 TTL、GPU 秒计量
│   └── audit/
│       └── service.go               # 关键操作审计日志
├── pkg/                             # 可复用、无业务耦合的基础设施
│   ├── config/                      # 进程配置(env/yaml → struct)
│   ├── db/                          # 连接池 + migrations(golang-migrate)
│   ├── sse/                         # 通用 SSE writer(心跳、Last-Event-ID、flush)
│   └── logger/                      # 结构化日志(log/slog)
├── schema/                          # → git submodule: eye-trace-config(契约真相源)
├── migrations/                      # SQL 迁移文件(golang-migrate 格式)
├── Makefile                         # gen / build / test / migrate
└── go.mod
```

---

## 2. 关键架构决策(展开)

### 2.1 ComfyUI WS 连接池(`comfy/wspool.go`)—— 最易踩坑点

**反模式**:每个生图请求开一条到 ComfyUI 的 WebSocket → 量上来后文件句柄爆炸。

**正确做法(类比游戏长连接 session)**:
- **每个 `client_id` 一条常驻 WS**。`client_id` 是 server 为每个活跃用户/会话生成的 UUID,存 DB,**不暴露给 client**。
- 该用户所有 prompt 复用这条 WS;ComfyUI 事件里带 `prompt_id`,`dispatch.go` 据此路由回正确的 job。
- WS 断线自动重连;重连期间 job 状态从 DB 兜底(`GET /api/history/{prompt_id}` 补拉)。
- 提交生图时 `POST /api/prompt` 的 body 带上该 `client_id`,确保进度事件回流到这条 WS。

### 2.2 WS → SSE 中继链路

```
ComfyUI WS event (status/progress/executing/executed)
   → comfy/dispatch.go        : 解析 + 按 prompt_id 查 jobId
   → job/repo.go              : 写 comfy_jobs(状态权威落库,先于 fanout)
   → job/hub.go               : 按 jobId fanout 给所有 SSE 订阅者
   → api/handler/run_events.go: 写 text/event-stream 给 client
```

`hub.go` 是内存中的 `map[jobId][]chan Event`。SSE handler 建连时注册 channel,断开时注销,终态事件后关闭并清理。
**多实例部署**时 hub 需换成 Redis Pub/Sub(MVP 单实例先用内存,但接口要抽象好,便于后续替换)。

### 2.3 comfy-user 派生(`comfy/binding.go`)—— 安全红线

- client 请求**从不**携带任何 comfy-user/comfy_user_id 字段。
- service 调 comfy-client 前,由 `BindingService.EnsureBinding(ctx, tenantId, serverId)` 返回 `comfy_user_id`:
  - 查 `tenant_comfy_bindings` 有无该 `(tenant_id, server_id)` 绑定;
  - 无 → lazy `POST /api/users` 拿 ID → 存表;
  - 有但状态 `broken` → 暂停调度该 tenant 到该 server,触发告警修复。
- `TenantAwareComfyClient` 是**唯一**注入 `comfy-user` header 的地方,值只接受 `BindingService` 的输出,绝不接受外部传入。

### 2.4 取消生图

```
DELETE /workflow-runs/{jobId}
  → job/service.go     : 查 comfy_jobs 拿 prompt_id
  → comfy/client.go    : POST /api/interrupt {"prompt_id": "<prompt_id>"}  ← 定向中断
```

**绝不**调无参数的全局 `POST /api/interrupt`(会杀掉同 worker 上所有租户的任务)。

---

## 3. 核心数据表(MVP)

每张业务表**强制** `tenant_id`,所有查询带 `WHERE tenant_id = $1`(防 IDOR)。

```sql
-- 租户与成员
tenants(id uuid PK, name text, created_at timestamptz)
tenant_memberships(tenant_id uuid, user_id uuid, role text, status text)
  -- role: owner | admin | creator | viewer | service_account

-- ComfyUI 绑定(tenant_shared 模式:每个 tenant × server 一个 comfy_user_id)
tenant_comfy_bindings(tenant_id uuid, server_id text, comfy_user_id text, status text)
  -- status: active | broken

-- 生图任务(★ 核心表)
comfy_jobs(
  id uuid PK,
  tenant_id uuid NOT NULL,      -- 强制,防 IDOR
  actor_user_id uuid NOT NULL,  -- 发起人
  model_id text,
  prompt_id text,               -- ComfyUI 内部 ID,永不暴露给 client
  client_id text,               -- ComfyUI WS client_id,永不暴露给 client
  status text,                  -- pending|running|succeeded|failed|cancelled
  progress numeric(4,3),        -- 0.000–1.000
  result_asset_id uuid,
  error text,
  created_at timestamptz,
  finished_at timestamptz
)

-- 资产(用户生成结果 + 上传的参考图)
comfy_assets(
  id uuid PK,
  tenant_id uuid NOT NULL,
  owner_user_id uuid NOT NULL,
  media_type text,              -- image | video
  comfy_filename text,          -- ComfyUI 侧文件名,用于 /api/view 代理
  hash text,
  size bigint,
  tags text[],
  created_at timestamptz
)

-- 审计
tenant_audit_logs(id uuid PK, tenant_id uuid, actor_user_id uuid, action text, target text, ts timestamptz)
```

> `comfy_jobs.prompt_id` 和 `client_id` 是 ComfyUI 内部映射,**永不**出现在任何 API 响应体里。

---

## 4. 技术选型建议

| 关切点 | 建议 | 理由 |
|--------|------|------|
| HTTP 路由 | `go-chi/chi` | oapi-codegen 原生支持 chi-server 模板,轻量无反射 |
| WebSocket(连 ComfyUI) | `coder/websocket` | context 友好,cancellation 干净 |
| DB | PostgreSQL + `jackc/pgx` | 多租户 + JSONB 存 params + Row-Level Security 可选 |
| DB 迁移 | `golang-migrate` | 与 `migrations/` 目录配套,CLI + Go 双调用 |
| 日志 | 标准库 `log/slog` | 结构化,零依赖,Go 1.21+ 内置 |
| 配置 | `caarlos0/env` + yaml | 12-factor:环境变量优先,yaml 做本地 dev override |
| JWT | `golang-jwt/jwt/v5` | 标准,维护活跃 |

---

## 5. 落地顺序

依赖关系决定的正确顺序,不要跳步:

1. **空壳服务跑通**:`cmd/server` + `pkg/{config,db,logger}` + `migrations/` 建表,服务能启动、能连 DB。
2. **★ 单链路打通**(先不做租户,固定单 worker 和固定 comfy-user):
   `comfy/{client,wspool,dispatch}` + `job/{service,repo,hub}` → `handler/{run,run_events}`。
   目标:client 提交生图 → ComfyUI 执行 → SSE 进度推回 → FSM 跑通 `generating → success`。
3. **client 接上**:填充 `eye-trace-client/src/api/`,基于生成的 TS 类型写 fetch client + EventSource,接 Generate FSM。
4. **补多租户**:`auth/service` + `middleware/{auth,tenant,rbac}` + `tenant/{service,repo}` + `comfy/binding`。
5. **补其余模块**:`asset` / `model` / `content` / `quota` / `audit`。
