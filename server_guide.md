# Go Server 服务端架构设计

> 本文档**仅**聚焦服务端架构,与 `project_guide.md` 互为索引。
> `project_guide.md` 是顶层架构总纲(全局拓扑、阶段规划、Worker-Agent、模板系统、DB 表、配置中心);本文件按其 §5「Server 核心模块」做**细化与落地**,并按其 §7「数据库核心表设计」做服务端表结构补充。
> 凡与 `project_guide.md` 重叠之处,以**该文档为准**;本文件仅展开 server 实现细节。
>
> **核心定位**:server 是**协议翻译层 + 状态权威层**,不是无脑转发器。
> 对外 REST + SSE(契约见 `openapi-skeleton.md`),对内 WS 连 Worker-Agent(非 ComfyUI)。
> 单独部署到 Linux。

---

## 0. 与 `project_guide.md` 的索引关系

| project_guide § | 主题 | 本文件处理 |
|---|---|---|
| §1 全局拓扑 | 整体架构图 | 直接引用,不在此重复 |
| §2 开发阶段规划 | Phase 1 / Phase 2 | §6 落地顺序对齐 |
| §3 Worker-Agent 详细设计 | Agent 注册 / WS 协议 / 本地执行 | server 侧消费端(§3 agent_pool + §4 worker 端点) |
| §4 参数化管线模板 | `{{PARAM_NAME}}` 占位符 / 校验 / 版本 | §5.5 派发逻辑引用 |
| §5 Server 核心模块 | 模块清单与变更 | **本文件主章节** |
| §6 Client 实时进度 | client 端 SSE 封装 | 不属 server,跳过 |
| §7 数据库核心表设计 | MySQL workers / jobs / job_outputs / models | §7 引用并补充 server 视角字段 |
| §8 eye-trace-config 扩展 | 三类配置导出 | §5.6 / §7 引用 |
| §9 命名迁移 | 品牌重命名 | 跳过(非架构) |

---

## 1. 指导原则(必须遵守)

1. **分层单向依赖**:`handler → service → (agent_pool | repo)`。handler 只做 HTTP 编解码,**零业务逻辑**;service 持有业务规则;repo 只碰 DB。**严禁 handler 直连 agent_pool 或 SQL**。
2. **handler 薄**:每个 handler ≤ 30 行 —— 解析请求 → 调 service → 写响应。实现 `api.gen.go` 生成的接口。
3. **DB 是 job 权威源**:每个 Worker-Agent WS 事件先落 DB,再 fanout 到 SSE。SSE 断线不丢状态(client 可 `GET /workflow-runs/{id}` 补偿)。
4. **业务配置 ≠ App 配置**:
   - `pkg/config` 加载**编译期** vendor 进来的 `internal/config/generated/config_gen.go` 契约枚举(由 `eye-trace-config` 导出);
   - 自身 `config.yaml` / 环境变量:DB DSN、监听端口、worker auth token 密钥、对象存储凭证;
   - **管线模板从 DB 的 `pipeline_templates` 表加载**,不读 config 目录文件系统。
5. **ComfyUI 隔离边界**:server 永远不直连 ComfyUI。原 `internal/comfy/` 包的职责迁出 server,迁入 Worker-Agent 二进制(详见 §5.2)。

---

## 2. 目录结构

```
eye-trace-server/backend/
├── cmd/
│   └── server/
│       └── main.go                  # 装配:读 config → 建 DB / agent_pool / pipeline cache → 挂路由 → 启动
├── internal/                        # 业务实现
│   ├── api/
│   │   ├── api.gen.go               # oapi-codegen 生成(types + chi-server 接口),勿手改
│   │   ├── handler/                 # 实现生成接口,薄 HTTP 层
│   │   │   ├── run.go               #   POST/GET/DELETE /workflow-runs(外部 tenant API)
│   │   │   ├── run_events.go        #   GET /workflow-runs/{id}/events  ← SSE
│   │   │   ├── model.go             #   /models
│   │   │   ├── asset.go             #   /assets
│   │   │   ├── content.go           #   /content/home
│   │   │   ├── auth.go              #   /auth, /me
│   │   │   └── worker.go            #   ★ POST /api/internal/workers/register + WS /api/internal/workers/stream
│   │   └── middleware/
│   │       ├── auth.go              # 校验 JWT/api-key → 注入 actor 到 ctx(外部 API)
│   │       ├── tenant.go            # 校验 X-Tenant-Id + 成员关系 → 注入 tenant 到 ctx
│   │       ├── rbac.go              # 角色 → 操作鉴权
│   │       ├── ratelimit.go         # 租户级限流/并发上限
│   │       ├── reqid.go             # X-Request-Id 注入 + 结构化日志
│   │       └── worker_token.go      # ★ Worker JWT 校验(内部端点专用,不进 tenant ctx)
│   ├── auth/
│   │   └── service.go               # 登录、签发/刷新 token、api-key 校验、worker JWT 签发
│   ├── tenant/
│   │   ├── service.go               # 租户、成员、角色 CRUD
│   │   └── repo.go
│   ├── agent_pool/                  # ★ 新增:Worker-Agent WS 连接池与调度器
│   │   ├── pool.go                  #   map[workerID]→wsConn,注册/注销/查询
│   │   ├── dispatch.go              #   Dispatch(job):capabilities 过滤 + least-jobs-first 分配
│   │   └── health.go                #   心跳追踪,3×interval 未响应 → offline + 触发 job 重调度
│   ├── job/
│   │   ├── service.go               # WorkflowRunService:提交/查询/取消;jobId↔worker 映射
│   │   ├── repo.go                  # jobs / job_outputs 表
│   │   ├── hub.go                   # ★ SSE fanout:jobId → 订阅者集合
│   │   └── dispatch.go              # ★ 调 agent_pool.Dispatch,管理 jobs 状态机(pending→running→...)
│   ├── pipeline/
│   │   ├── cache.go                 # 启动加载 + 热更新 pipeline_templates + 对应 schemas
│   │   ├── validate.go              # 按 schema 校验 params
│   │   └── render.go                # 占位符替换 strings.ReplaceAll + json.Valid 兜底
│   ├── asset/
│   │   ├── service.go               # 上传/列表/删除,owner 字段强制
│   │   └── repo.go
│   ├── model/
│   │   └── service.go               # 模型目录(读 DB / 缓存)
│   ├── content/
│   │   └── service.go               # 首页内容块(banner/recommend/tabs)
│   ├── quota/
│   │   └── service.go               # 配额:并发数、存储 TTL、GPU 秒计量
│   ├── audit/
│   │   └── service.go               # 关键操作审计日志
│   ├── config/
│   │   └── generated/
│   │       └── config_gen.go        # ★ 编译期 vendor 契约枚举(来自 eye-trace-config),勿手改
│   └── comfy/                       # ★ Phase 2 阶段:空包;Phase 3 多租户时再承载 binding 逻辑
│
├── pkg/                             # 可复用、无业务耦合的基础设施
│   ├── config/                      # 进程配置(yaml + env → struct)
│   ├── db/                          # 连接池 + migrations(golang-migrate)
│   ├── sse/                         # 通用 SSE writer(心跳、Last-Event-ID、flush)
│   ├── wspool/                      # ★ 通用 WS Hub(agent_pool 与 SSE fanout 复用):Subscribe/Publish/Close
│   ├── storage/                     # ★ S3 兼容客户端(MinIO / AWS S3 统一接口)
│   └── logger/                      # 结构化日志(log/slog)
│
├── schema/                          # → git submodule: eye-trace-config(契约真相源,运行时不再读)
├── migrations/                      # SQL 迁移文件(golang-migrate 格式)
├── Makefile                         # gen / build / test / migrate / seed
└── go.mod
```

> **空包 `internal/comfy/` 保留**:在 server 二进制中作为占位符,等待 Phase 3 多租户时承载 `binding.go`(tenant_comfy_bindings)。当前所有 ComfyUI 交互代码在 `cmd/worker-agent/internal/comfy/`,不在此 server 仓内。

---

## 3. `internal/agent_pool` — Worker WS 连接池与调度器(server 独有模块)

本模块是 Worker-Agent 架构下 server 的**唯一**对内通信组件。设计目标与断线容错由 `project_guide.md` §3.5 / §3.6 给出,本节做 server 实现细化。

### 3.1 内存结构

```go
type Pool struct {
    mu      sync.RWMutex
    workers map[string]*workerEntry   // workerID → entry
}

type workerEntry struct {
    conn          *websocket.Conn
    capabilities  []string             // 支持的 templateID
    gpuModel      string
    vramGB        int
    activeJobs    int                  // 已 accepted 尚未 complete 的 job 数
    lastHeartbeat time.Time
    status        workerStatus         // online | busy | offline
    sendCh        chan envelope        // 出站消息缓冲
}
```

- `workers` map 由 `Register` / `Unregister` 维护;`Dispatch` 加读锁遍历。
- 每个 workerEntry 持有独立 `sendCh` 协程,避免对单个慢 worker 的写阻塞整个 pool。

### 3.2 调度策略(`dispatch.go`)

`Dispatch(job *Job) error` 流程:

```
1. 读 lock,遍历 workers:
   a. status != online          → 跳过
   b. !capabilities 含 job.templateID → 跳过
   c. 收集 (entry, activeJobs, vramFreeGB) 候选
2. 候选按 activeJobs 升序排序;并列时按 vramFreeGB 降序
3. 取第一名,非阻塞尝试 send {type:"submit", jobID, templateID, params} 到 sendCh
4. 发送成功 → 返回 nil(server 后续等 accepted 消息再 DB status=running)
5. 无候选 → job 留 status=pending,等 health.go 触发 worker 上线/释放后重试
```

多 server 实例并存时,`jobs` 表用 `SELECT … FOR UPDATE SKIP LOCKED`(详见 §7.5)做横向扩展,无需额外分布式锁。

### 3.3 健康检查(`health.go`)

- 启动时按 `heartbeatIntervalSec`(worker 注册时回传)为每个 online worker 启动独立 ticker。
- 每 tick 向 `sendCh` 推 `{type:"ping"}`。
- Worker 回 `{type:"pong", vramFreeGB}` → 更新 `lastHeartbeat`。
- `3 × heartbeatIntervalSec` 内未收到 pong → 状态置 offline,从 map 移除;对所有 `status=running` 且 `worker_id=此worker` 的 job 重新调 `Dispatch` 走重调度。

### 3.4 重连幂等

- Worker 断线重连后再次 POST `/api/internal/workers/register` → server 拿到相同 `workerID` → 复用旧 entry,仅刷新 conn 与 sendCh。
- 重连期间 agent 本地继续执行的 job,complete/failed 消息会带 `jobID` 重新上报;server 端按 `jobID` upsert(状态机不接受回退:running → succeeded/failed OK,running → pending 拒绝)。

---

## 4. `internal/api/handler/worker.go` — 内部端点(不走 tenant middleware)

### 4.1 注册

```
POST /api/internal/workers/register
Headers: 无需 Authorization;由 worker_token.go 校验初次握手签名(Phase 2 简化:用预共享 secret)
Body:
{
  "workerID":     "<hostname>-<uuid>",
  "capabilities": ["txt2img_sdxl_v1", ...],
  "gpuModel":     "NVIDIA RTX 4090",
  "vramGB":       24
}

→ 200:
{
  "token":                "<worker-jwt>",
  "heartbeatIntervalSec": 30
}
```

副作用:UPSERT `workers` 表,`status='online'`,`last_heartbeat=NOW()`。

### 4.2 长连接

```
WS /api/internal/workers/stream?token=<worker-jwt>
```

升级后由 `worker_token.go` 校验 token → 注入 `workerID` 到 ctx → 移交 `agent_pool.Register(workerID, conn, capabilities, …)` 接管生命周期。

**Server → Agent 出站消息**:见 `project_guide.md` §3.3 表格,本模块只做 envelope 编码,业务载荷由调用方传入。

**Agent → Server 入站消息路由**(本 handler 内):

| type | 处理 |
|---|---|
| `accepted`  | `job/dispatch.go`:jobs.status pending→running,记 started_at |
| `progress`  | `job/repo.go` 写 progress(可选,事件可选持久化) + `job/hub.go` 广播 SSE `progress` 事件 |
| `preview`   | `job/hub.go` 广播 SSE `preview` 事件(不落 DB,体积大) |
| `complete`  | `job/dispatch.go`:写 `job_outputs`,jobs.status running→succeeded,记 finished_at → 广播 `succeeded` |
| `failed`    | jobs.status running→failed,error_msg 落库 → 广播 `failed` |
| `pong`      | 更新 `last_heartbeat`、`vramFreeGB` |

---

## 5. 关键流程

### 5.1 提交生图

```
POST /api/v1/jobs        (handler/run.go)
  → middleware/auth+tenant 注入 actor/tenant 到 ctx
  → job/service.go          Submit(ctx, modelID, params)
       ├── pipeline/validate.go  按 schema 校验 params
       ├── jobs 表 INSERT(status='pending', template_snapshot=当前 DB 中模板内容)
       └── job/dispatch.go       agent_pool.Dispatch(job)
            ├── 有候选 → 发 submit → 等 accepted(协程)
            └── 无候选 → job 留 pending,等 health.go 触发重试
  → 201 { jobId }
```

### 5.2 状态机(所有转移必须经过 DB)

```
pending → running    (agent 发 accepted)
running → succeeded  (agent 发 complete,outputs 写 job_outputs)
running → failed     (agent 发 failed)
pending/running → cancelled  (client 发 DELETE /workflow-runs/{id},server 发 cancel 给 agent)
```

非法转移(例如 running → pending)由 `job/dispatch.go` 的转换函数拒绝,记录到 audit log。

### 5.3 取消生图

```
DELETE /api/v1/jobs/{jobId}  (handler/run.go)
  → job/service.go:查 jobs 拿 worker_id
  → agent_pool.Send(worker_id, {type:"cancel", jobID})
       ↓ agent 收到后:
       ├── 未提交 ComfyUI → 直接回 cancelled
       └── 已提交 → POST localhost:8188/interrupt {prompt_id: jobID}(定向中断,不调全局)
```

**绝不**调无参数的全局 `POST /api/interrupt`(会杀掉同 worker 上所有租户的任务)—— 此约束由 agent 二进制保证;server 仅做消息转发,不感知 ComfyUI 细节。

### 5.4 SSE fanout 链路(对齐 `project_guide.md` §5.2)

```
Agent WS {type:"progress"/"preview"/"complete"/"failed", jobID, ...}
  → handler/worker.go      : 解析消息类型
  → job/service.go         : 先写 DB(状态权威落库)
  → job/hub.go             : 按 jobID fanout 给所有 SSE 订阅者
  → handler/run_events.go  : 写 text/event-stream 给 client
```

`job/hub.go` 结构与原 `go-server-structure.md` §2.2 一致,只是 WS 来源从 ComfyUI 换成 Worker-Agent:

```go
// 内存 hub(单实例 MVP)
type Hub struct {
    mu   sync.RWMutex
    subs map[int64][]chan Event  // jobId → 订阅者 channel 列表
}

// 多实例部署:把 map 换成 Redis Pub/Sub,接口保持不变
type Hub interface {
    Subscribe(jobId int64) (<-chan Event, unsubscribe func())
    Publish(jobId int64, e Event)
}
```

SSE 写器封装在 `pkg/sse/`,提供 `WriteEvent(event)` / `Heartbeat()` / `Last-Event-ID` 续传。

### 5.5 模板渲染与版本快照(对齐 `project_guide.md` §4.5 / §4.6)

`job/dispatch.go` 在 Submit 阶段把当时 DB 中的模板内容 snapshot 进 `jobs.template_snapshot`,并记 `template_id` + `template_version`(语义版本或 content hash)。worker 收到的 `params` 已**不包含**模板;agent 端拿 `templateID + templateVersion` 从**自身缓存**取模板(由 server 提交时附带),做占位符替换后 POST ComfyUI。

历史复现:读 `jobs.template_snapshot` 即可重放,不依赖 config 目录当前状态。

### 5.6 配置加载

- **契约枚举**:`pkg/config` 启动时读 `internal/config/generated/config_gen.go`(vendor 自 `eye-trace-config/dist/server/config_gen.go`)。运行时不可改;改 = 重新 build。
- **管线模板**:`internal/pipeline/cache.go` 启动时 `SELECT * FROM pipeline_templates` 加载到内存;`pipeline_templates` 表由 `make seed` 从 `eye-trace-config/dist/seed/pipelines.json` 灌入。Admin 改 DB 后调内部 admin 端点触发 cache reload,无需重新部署。
- **运行时配置**:`pkg/config.Config` 由 `config.yaml` + 环境变量合成(DB DSN / 监听端口 / worker auth token 签名密钥 / 对象存储凭证)。

对象存储:`pkg/storage` 封装 S3 兼容客户端,凭 `storageEndpoint / storageAccessKey / storageSecretKey / storageBucket / storageRegion / storageUseSSL` 接入。开发 MinIO、上线 AWS S3,只换 endpoint / 凭证,业务代码不动。

---

## 6. 落地顺序(对齐 `project_guide.md` §2)

1. **空壳服务跑通**:`cmd/server` + `pkg/{config,db,logger,wspool,sse,storage}` + `migrations/` 建表,服务能启动、能连 DB、能热加载 `pipeline_templates`。
2. **★ Phase 2 单链路打通**(固定单 worker、单模板,无租户):
   `internal/{agent_pool, job, pipeline, handler/worker, handler/run, handler/run_events}` → 端到端跑通 txt2img_sdxl_v1。
3. **client 接上**:`eye-trace-client/src/api/sse.ts` 切真实实现,FSM 消费真实事件。
4. **补多租户**:`auth/service` + `middleware/{auth,tenant,rbac}` + `tenant/{service,repo}` + `internal/comfy/binding`。
5. **补其余模块**:`asset` / `model` / `content` / `quota` / `audit`。

---

## 7. 数据库核心表(对齐 `project_guide.md` §7,server 视角补充)

SQL 真相源在 `migrations/` 下,golang-migrate 升序执行。本节只补充 server 实现所需的**索引 / 约束细节**与**典型查询**。

### 7.1 workers(状态权威 + 调度依据)

```sql
CREATE TABLE workers (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  worker_id     VARCHAR(64)  NOT NULL UNIQUE,
  hostname      VARCHAR(255) NOT NULL,
  status        ENUM('online','offline','busy') NOT NULL DEFAULT 'offline',
  gpu_model     VARCHAR(128),
  vram_gb       INT,
  vram_free_gb  INT,                          -- ★ 由 pong 消息持续更新
  capabilities  JSON,                          -- 支持的 templateID 数组
  last_heartbeat TIMESTAMP NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_status (status),
  INDEX idx_last_heartbeat (last_heartbeat)   -- ★ health.go 离线判定扫描用
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

典型查询:
- 离线判定:`SELECT worker_id FROM workers WHERE status='online' AND last_heartbeat < NOW() - INTERVAL ? SECOND`
- 调度候选:`SELECT worker_id, vram_free_gb, active_jobs FROM workers WHERE status='online' AND JSON_CONTAINS(capabilities, JSON_QUOTE(?))`

### 7.2 jobs(任务主表)

```sql
CREATE TABLE jobs (
  id                BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id           BIGINT NULL,                -- Phase 3 多租户前可空
  tenant_id         BIGINT NULL,                -- ★ Phase 3 启用,Phase 2 可空但保留字段
  pipeline_id       VARCHAR(64)  NOT NULL,      -- templateID
  template_version  VARCHAR(40)  NOT NULL,      -- 语义版本或 content hash
  template_snapshot LONGTEXT     NULL,          -- 派发时的模板 JSON 快照
  params            JSON         NOT NULL,
  status            ENUM('pending','running','succeeded','failed','cancelled')
                      NOT NULL DEFAULT 'pending',
  progress          DECIMAL(4,3) NULL,          -- 0.000–1.000,由 progress 事件写入
  worker_id         VARCHAR(64)  NULL,          -- 分配到的 worker
  error_msg         TEXT         NULL,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  started_at        TIMESTAMP NULL,
  finished_at       TIMESTAMP NULL,
  INDEX idx_status_created (status, created_at),
  INDEX idx_user (user_id),
  INDEX idx_worker_status (worker_id, status)   -- ★ 重调度扫描用
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

典型查询:
- 用户拉取 job 列表:`SELECT … WHERE user_id=? ORDER BY created_at DESC LIMIT ?`
- SSE 断线补偿:`GET /workflow-runs/{id}` → `SELECT * FROM jobs WHERE id=?`
- 重调度扫描:`SELECT id, pipeline_id, params FROM jobs WHERE status='pending' ORDER BY created_at ASC FOR UPDATE SKIP LOCKED LIMIT 1`

### 7.3 job_outputs(生成产物索引)

```sql
CREATE TABLE job_outputs (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  job_id          BIGINT       NOT NULL,
  filename        VARCHAR(255) NOT NULL,
  storage_url     VARCHAR(512) NOT NULL,        -- 对象存储 key 或完整 URL
  media_type      ENUM('image','video') NOT NULL,
  width           INT NULL,
  height          INT NULL,
  file_size_bytes BIGINT NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_job (job_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**SSE `succeeded` 事件载荷**:`resultUrl` 取 `job_outputs` 中第一条 `image` 类型记录的 `storage_url`(相对路径 `/api/v1/assets/{id}/content`,由 server 解析为可访问 URL)。

### 7.4 models(模型目录)

```sql
CREATE TABLE models (
  id                BIGINT AUTO_INCREMENT PRIMARY KEY,
  name              VARCHAR(255) NOT NULL,
  type              ENUM('checkpoint','lora','template','workflow','video') NOT NULL,
  base_model        VARCHAR(64) NULL,            -- sd15 / sdxl / flux / ...
  civitai_id        VARCHAR(64) NULL,
  preview_image_url VARCHAR(512) NULL,
  tags              JSON,
  meta              JSON,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_type (type),
  FULLTEXT idx_name_ft (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 7.5 多调度器并发抢 job:SELECT … FOR UPDATE SKIP LOCKED

```sql
START TRANSACTION;

SELECT id, pipeline_id, params, template_version
FROM jobs
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;                          -- 已被其它事务锁定的行直接跳过

UPDATE jobs SET status='running', worker_id=?, started_at=NOW() WHERE id=?;

COMMIT;
```

MySQL 8.0+ 支持 `SKIP LOCKED`。该模式让 dispatcher 天然支持横向扩展,无需额外分布式锁。Phase 2 单实例下可选使用,保持语义一致以便直接扩。

### 7.6 pipeline_templates(模板真相源,运行时从 DB 加载)

不在 `project_guide.md` §7 列出,作为 server 实现补充表:

```sql
CREATE TABLE pipeline_templates (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  template_id   VARCHAR(64)  NOT NULL,           -- 如 txt2img_sdxl_v1
  version       VARCHAR(40)  NOT NULL,           -- 语义版本或 content hash
  graph_json    LONGTEXT     NOT NULL,           -- ComfyUI API-format graph,含 {{PARAM}}
  schema_yaml   LONGTEXT     NOT NULL,           -- 参数 schema
  is_active     TINYINT(1)   NOT NULL DEFAULT 1,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_template_version (template_id, version)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

由 `make seed` 从 `eye-trace-config/dist/seed/pipelines.json` 灌入;admin 可改后调内部端点触发 `pipeline/cache.go` reload。

---

## 8. 技术选型

| 关切点 | 选型 | 理由 |
|---|---|---|
| HTTP 路由 | `go-chi/chi` | oapi-codegen 原生支持 chi-server 模板,轻量无反射 |
| WebSocket(连 Worker-Agent) | `coder/websocket` | context 友好,cancellation 干净 |
| DB | **MySQL 8.0+** + `go-sql-driver/mysql` | `FOR UPDATE SKIP LOCKED` 横向扩展;与 worker-agent / 对象存储栈统一 |
| DB 迁移 | `golang-migrate` | 与 `migrations/` 目录配套,CLI + Go 双调用 |
| 日志 | 标准库 `log/slog` | 结构化,零依赖,Go 1.21+ 内置 |
| 配置 | `caarlos0/env` + yaml | 12-factor:环境变量优先,yaml 做本地 dev override |
| JWT | `golang-jwt/jwt/v5` | 用户 token + worker token 共用一套 |
| 对象存储 | `aws-sdk-go-v2` S3 client | MinIO / AWS S3 同协议,只换 endpoint / 凭证 |
| OpenAPI 客户端生成 | `oapi-codegen` | 产出 `api.gen.go`(types + chi-server 接口) |

---

## 9. 不在本文件范围(避免重复)

- **全局拓扑 / 阶段规划**:见 `project_guide.md` §1 / §2
- **Worker-Agent 二进制**(`cmd/worker-agent`)的内部结构:见 `project_guide.md` §3,以及未来 `worker_agent_guide.md`(本文件不重复)
- **管线模板 YAML 格式 / 占位符清单 / Schema 字段**:见 `project_guide.md` §4
- **client 侧 SSE 封装 / FSM 接入**:见 `project_guide.md` §6
- **eye-trace-config 目录结构 / 导出器**:见 `project_guide.md` §8
