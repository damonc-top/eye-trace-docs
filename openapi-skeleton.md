# API 契约骨架 (OpenAPI 3.1) — 架构指导

> 本文是 **client ↔ server 的唯一契约真相源**。次级模型据此补全完整 `openapi.yaml`,
> 再双向生成代码:Go 用 `oapi-codegen`,client 用 `openapi-typescript` + 现有 `zod`。
> **不要**手写 client/server 的请求/响应类型 —— 全部从本契约生成。

---

## 0. 不可违背的设计决策

1. **传输层**:常规操作 = REST + JSON;生图进度/结果 = **SSE**;文件 = multipart。**不用 Protobuf/gRPC**。
2. **版本**:所有路径前缀 `/api/v1`。破坏性变更进 `/api/v2`,旧版并存。
3. **鉴权**:`Authorization: Bearer <jwt|api-key>` + `X-Tenant-Id: <tenant_id>`(除 `/auth/*`)。
4. **租户隔离**:server 端 `comfy-user` **永远从后端绑定表派生**,契约里**不存在**任何让 client 传 `comfy-user`/`client_id`/`comfy_user_id` 的字段。谁加谁就是 IDOR 漏洞。
5. **job 状态权威源 = server 业务 DB**,不是 ComfyUI history。契约只暴露 server 的 `jobId`,**绝不**透出 ComfyUI 的 `prompt_id`。
6. **幂等**:所有写操作(尤其 `POST /workflow-runs`)接受 `Idempotency-Key` header。

---

## 1. 与 client 现有代码的对齐约束(硬性)

契约必须和 client `src/features/generate/fsm.ts` 的状态机咬合,否则前端要写适配层。

### 1.1 JobStatus ↔ FSM 状态映射

client FSM 状态:`idle → selectingModel → configuring → generating → success → error`。
其中 `generating/success/error` 是**真实有后端交互**的状态。契约的 `JobStatus` 必须能驱动这三态:

| API `JobStatus` | 触发 client FSM                 | FSM 落点状态  |
|-----------------|--------------------------------|--------------|
| `pending`       | (SSE `queued`) 排队提示,不转移   | `generating` |
| `running`       | `PROGRESS{percent}`            | `generating` |
| `succeeded`     | `SUCCESS{resultUrl}`           | `success`    |
| `failed`        | `ERROR{error}`                 | `error`      |
| `cancelled`     | `ERROR{error:"CANCELLED"}`     | `error`      |

> 命名用 `succeeded`/`failed`(过去式,REST 惯例),**不要**用 FSM 内部状态名 `success`/`error`。client 适配层做 1:1 翻译。

### 1.2 SSE 事件 ↔ FSM 事件映射(关键)

FSM 在 `generating` 态只接收三种事件:`PROGRESS{percent}` / `SUCCESS{resultUrl}` / `ERROR{error}`。
SSE 事件**必须**能无损翻译成它们:

| SSE `event:` | SSE `data` 关键字段           | → FSM 事件               |
|--------------|-------------------------------|--------------------------|
| `queued`     | `position` (排队位次)           | (仅 UI,无 FSM 转移)       |
| `progress`   | `percent` (0.0–1.0)           | `PROGRESS{percent}`      |
| `succeeded`  | `resultUrl`, `thumbnailUrl?`  | `SUCCESS{resultUrl}`     |
| `failed`     | `code`, `message`             | `ERROR{error: message}`  |

> `percent` 用 **0.0–1.0 浮点**(对齐 FSM `ctx.progress`,`SUCCESS` 时置 1)。**不要**用 0–100 整数。

### 1.3 内容数据形状对齐

`GET /api/v1/content/home` 响应**直接复用** client `src/config/static/types.ts` 已有的形状,
让前端把"硬编码 YAML"无缝换成"API 返回",不改渲染组件。
次级模型应把这些 TS interface 原样翻译成 OpenAPI schema。

**BannerCarousel 替代(2026-06-24)**:老 `BannerCarouselConfig`(carousel + sideCards + overlayLinks)
已废弃,改用判别联合 `BannerSlot`(见 §3)。
client 端 `types.ts` 删除旧 `BannerCarouselConfig` / `CarouselSlideConfig` / `StaticCardConfig` /
`OverlayLinkConfig`,替换为 `BannerSlot` / `CarouselSlide` / `SingleBanner` / `ActionButton` /
`RouteTarget`。
旧 `BannerCarouselConfig.src` 改名为 `imageUrl`,旧 `href` 改名为 `target.viewId + params`
(走内部路由,不暴露外链字段;外部链接由客户端包成 webview viewId)。
`home_content_slots` 表同步重命名为 `home_banner_slots`,并新增 `home_banner_targets` 表
存跳转目标(viewId + params)。

---

## 2. 端点清单(MVP 范围)

按落地优先级排序。打 ★ 的是打通单链路的最小集合,先实现这些。

### 认证 `/auth`

| Method | Path            | 说明 |
|--------|-----------------|------|
| POST   | `/auth/login`   | 账号密码 → `{accessToken, refreshToken, expiresIn}` |
| POST   | `/auth/refresh` | refreshToken → 新 accessToken |
| POST   | `/auth/logout`  | 吊销当前 token |
| GET    | `/me`           | 当前用户 + 所属 tenant 列表 + 角色 |

### 生图任务 `/workflow-runs` ★

| Method    | Path                              | 说明 |
|-----------|-----------------------------------|------|
| ★ POST    | `/workflow-runs`                  | 提交生图。body=`CreateRunRequest`,返回 `WorkflowRun{jobId,status}`(201) |
| ★ GET     | `/workflow-runs/{jobId}`          | 查单个 job 当前状态(SSE 断线后补偿拉取) |
| ★ GET     | `/workflow-runs/{jobId}/events`   | **SSE 流**,`Accept: text/event-stream` |
| ★ DELETE  | `/workflow-runs/{jobId}`          | 取消。server 内部转 ComfyUI 定向 `interrupt{prompt_id}` |
| GET       | `/workflow-runs`                  | 分页列出本租户历史 job(`status`/`limit`/`offset` 过滤) |

### 模型 `/models` ★

| Method | Path               | 说明 |
|--------|--------------------|------|
| ★ GET  | `/models`          | 分页模型目录,返回 `ModelSummary[]`(替代硬编码 modelItems) |
| GET    | `/models/{modelId}`| 模型详情:可调参数 schema、预览图、默认值 |

### 资产 `/assets`(用户生成结果的隔离存储)

| Method | Path                       | 说明 |
|--------|----------------------------|------|
| GET    | `/assets`                  | 分页本租户资产(`tag`/`mediaType` 过滤) |
| POST   | `/assets`                  | multipart 上传(参考图/垫图) |
| GET    | `/assets/{assetId}`        | 元数据 |
| GET    | `/assets/{assetId}/content`| 二进制下载(server 代理 ComfyUI `/api/view`) |
| DELETE | `/assets/{assetId}`        | 删除(用户资产隔离的删除入口) |

### 内容 `/content`

| Method | Path             | 说明 |
|--------|------------------|------|
| GET    | `/content/home`  | 首页内容块,形状复用 client types.ts(banner/recommend/modelItems/tabs) |

> **绝不出现在契约里**的 ComfyUI 端点:`/api/history`、`/api/queue`、`/api/interrupt`、
> `/api/free`、`/api/users`、`/api/object_info`。这些是 server 内部对 worker 的调用,不对 client 暴露。

---

## 3. 核心 Schema 骨架(YAML)

次级模型直接拷进 `components.schemas`,补全字段约束。

```yaml
openapi: 3.1.0
info:
  title: Eye Trace API
  version: 0.1.0
servers:
  - url: /api/v1

security:
  - bearerAuth: []

components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }

  parameters:
    TenantId:
      name: X-Tenant-Id
      in: header
      required: true
      schema: { type: string, format: uuid }
    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: false
      schema: { type: string }

  schemas:
    # ── 共享枚举(同时进 eye-trace-config/shared,client+server 共读)──────────
    JobStatus:
      type: string
      enum: [pending, running, succeeded, failed, cancelled]

    MediaType:
      type: string
      enum: [image, video]

    ErrorCode:
      type: string
      enum:
        - VALIDATION_FAILED
        - UNAUTHORIZED
        - FORBIDDEN
        - NOT_FOUND
        - QUOTA_EXCEEDED
        - WORKER_UNAVAILABLE
        - GENERATION_FAILED
        - CANCELLED
        - RATE_LIMITED
        - INTERNAL

    # ── 统一错误信封(所有 4xx/5xx 用它)─────────────────────────────────────
    Error:
      type: object
      required: [code, message, requestId]
      properties:
        code:      { $ref: '#/components/schemas/ErrorCode' }
        message:   { type: string }
        requestId: { type: string }
        details:   { type: object, additionalProperties: true }

    # ── 提交生图 ─────────────────────────────────────────────────────────────
    CreateRunRequest:
      type: object
      required: [modelId, params]
      properties:
        modelId:
          type: string
        # params 是模型相关的自由 KV,对齐 FSM ctx.params: Record<string, unknown>
        params:
          type: object
          additionalProperties: true
        # 可选:垫图/参考图引用已上传的 asset
        inputAssetIds:
          type: array
          items: { type: string }

    WorkflowRun:
      type: object
      required: [jobId, status, createdAt]
      properties:
        jobId:        { type: string, format: uuid }
        status:       { $ref: '#/components/schemas/JobStatus' }
        modelId:      { type: string }
        progress:     { type: number, minimum: 0, maximum: 1 }  # 0.0–1.0
        resultUrl:    { type: string, nullable: true }
        thumbnailUrl: { type: string, nullable: true }
        error:        { type: string, nullable: true }
        createdAt:    { type: string, format: date-time }
        finishedAt:   { type: string, format: date-time, nullable: true }

    ModelSummary:
      type: object
      required: [modelId, title, mediaType, cover]
      properties:
        modelId:   { type: string }
        title:     { type: string }
        author:    { type: string }
        cover:     { type: string }
        typeTag:   { type: string }
        mediaType: { $ref: '#/components/schemas/MediaType' }
        # 渲染相关字段镜像 client ModelItemCardBase(previews/badge/exclusive/...)
        # 次级模型补全,对照 eye-trace-client/src/config/static/types.ts

    # ── 首页 banner slot(替代 BannerCarouselConfig)────────────────────
    # 三种变体:carousel(左 36.42%)/ static(单图) / staticActions(单图+按钮条)。
    # 数组长度 1..3;每个 slot 独立类型。
    BannerSlot:
      oneOf:
        - $ref: '#/components/schemas/BannerSlotCarousel'
        - $ref: '#/components/schemas/BannerSlotStatic'
        - $ref: '#/components/schemas/BannerSlotStaticActions'
      discriminator:
        propertyName: kind
        mapping:
          carousel:      '#/components/schemas/BannerSlotCarousel'
          static:        '#/components/schemas/BannerSlotStatic'
          staticActions: '#/components/schemas/BannerSlotStaticActions'

    BannerSlotCarousel:
      type: object
      required: [kind, slides]
      properties:
        kind: { type: string, enum: [carousel] }
        intervalMs: { type: integer, default: 5000, minimum: 1000 }
        slides:
          type: array
          minItems: 1
          items: { $ref: '#/components/schemas/CarouselSlide' }

    BannerSlotStatic:
      type: object
      required: [kind, banner]
      properties:
        kind: { type: string, enum: [static] }
        banner: { $ref: '#/components/schemas/SingleBanner' }

    BannerSlotStaticActions:
      type: object
      required: [kind, banner, actions]
      properties:
        kind: { type: string, enum: [staticActions] }
        banner: { $ref: '#/components/schemas/SingleBanner' }
        actions:
          type: array
          minItems: 1
          maxItems: 8
          items: { $ref: '#/components/schemas/ActionButton' }

    SingleBanner:
      type: object
      required: [imageUrl, target]
      properties:
        imageUrl: { type: string, format: uri }
        alt:      { type: string }
        target:   { $ref: '#/components/schemas/RouteTarget' }

    CarouselSlide:
      type: object
      required: [imageUrl, target]
      properties:
        imageUrl: { type: string, format: uri }
        alt:      { type: string }
        target:   { $ref: '#/components/schemas/RouteTarget' }

    ActionButton:
      type: object
      required: [label, target]
      properties:
        label:
          type: string
          maxLength: 8
        iconUrl: { type: string, format: uri }
        target:  { $ref: '#/components/schemas/RouteTarget' }

    # 跳转目标:只支持内部路由 viewId + params。
    # 外部链接由 client 包成 webview viewId,契约不暴露外链字段。
    RouteTarget:
      type: object
      required: [viewId]
      properties:
        viewId: { type: string }
        params:
          type: object
          additionalProperties: true
```

---

## 4. SSE 事件信封(text/event-stream)

`GET /workflow-runs/{jobId}/events` 的 wire 格式。次级模型在 Go 端按此写 SSE writer。

```
id: 1
event: queued
data: {"jobId":"...","status":"pending","position":3}

id: 2
event: progress
data: {"jobId":"...","status":"running","percent":0.42}

id: 3
event: succeeded
data: {"jobId":"...","status":"succeeded","resultUrl":"/api/v1/assets/<id>/content","thumbnailUrl":"..."}

id: 3
event: failed
data: {"jobId":"...","status":"failed","code":"GENERATION_FAILED","message":"..."}
```

**SSE 实现约束**:
- 每条 `data` 是单行 JSON(SSE 不允许裸换行;统一压成单行最省事)。
- 终态事件(`succeeded`/`failed`)发送后 server **主动关闭流**。
- 支持断线重连:client 带 `Last-Event-ID` header,server 从该序号之后重放未发事件。`id:` 字段写 job 内单调递增序号。
- 心跳:空闲时每 15s 发 `: keepalive` 注释行,防代理超时断连。

---

## 5. 代码生成落地

完整 OpenAPI 契约见同目录 `openapi.yaml`(本骨架的展开版本,含本节追加的 `BannerSlot` 等)。

```bash
# Go server(放 server 的 Makefile)
oapi-codegen -generate types,chi-server,spec \
  -package api openapi.yaml > internal/api/api.gen.go

# Client(放 client package.json scripts)
openapi-typescript openapi.yaml -o src/api/schema.gen.ts
```

client 侧:生成的 TS 类型 + 现有 `zod` 做运行时边界校验。
SSE `data` 解析后过 zod schema,防 server 契约漂移在运行时静默出错。
