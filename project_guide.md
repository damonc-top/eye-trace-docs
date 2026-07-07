# EyeTraceAI 系统架构指导 v1

> 版本: 1.0 | 日期: 2026-06-12 | 状态: 规范

本文是 EyeTraceAI 的顶层架构总纲,统合并扩展以下文档(原列出的 `go-server-structure.md` 已并入本文 §5,`client-data-split-guide.md` 已改名为 [`client_data_split.md`](./client_data_split.md)):
[`openapi-skeleton.md`](./openapi-skeleton.md)、[`client_data_split.md`](./client_data_split.md)、
[`liblib-client-function-map.md`](./liblib-client-function-map.md)、[`client-architecture.md`](./client-architecture.md)。本文只做**扩展与补全**,不推翻上述文档的任何既有决策。
凡与既有文档重叠处,以既有文档为准;本文新增的部分(Worker-Agent、参数化管线、MySQL 表)为唯一新增真相源。

## 目录

1. 全局拓扑
2. 开发阶段规划
3. Worker-Agent 详细设计
4. 参数化管线模板系统
5. Server 核心模块(原 `go-server-structure.md` 内容已并入,无独立分卷)
6. Client 侧实时进度接入
7. 数据库核心表设计(MySQL)
8. eye-trace-config 扩展清单
9. 命名迁移任务
10. 文档索引与子文档

---

## 10. 文档索引与子文档

本文是大纲,所有"展开即可施工"的细节下沉到子文档,避免单文件膨胀。本节给出**唯一的索引关系**,子文档之间不得互相复制:

| 关注点 | 主文档 | 子文档(展开施工) |
|---|---|---|
| Client 架构与 Phase 1 实施 | 本文 §2 Phase 1 | [`client-architecture.md`](./client-architecture.md) — 14 路由外壳清单、AppShell 形态选择、UI 原语、平台桥、设计 token、命名迁移落点、Phase 1 完成判据、PR 拆分 |
| Server 模块 | 本文 §5(原 `server_guide.md` 已并入,无独立分卷) | — |
| 数据契约 / 拆分 | 本文 §7 + [`client_data_split.md`](./client_data_split.md) | (沿用,不分流) |
| liblib 仿站功能面 | [`liblib-client-function-map.md`](./liblib-client-function-map.md) | `client-architecture.md` §5(路由→page→yaml 段映射) + §10(仿站原语登记) |
| Client 前端技术栈 | `client/docs/client-frontend-roadmap.md` | `client-architecture.md` §3 §4 §8(消费其结论,不复述) |
| 客户端实时进度 | 本文 §6 | `client-architecture.md` §7 `api/sse.ts` 接入位 + 命名迁移落点 |

**核心契约**:

- `client-architecture.md` 是 Phase 1 实施蓝本,凡涉及新增路由、改动 AppShell、设计 token、平台桥命名,**先改子文档再改代码**。
- 本文是大纲,只列"是什么 / 为什么 / 与其他文档关系",不列"目录结构 / 函数签名 / 字段定义"。
- 子文档之间不互相覆盖:同一事实只在唯一一处展开,其他位置用"详见 `<子文档> §<X>`"链回。

---

## 1. 全局拓扑

EyeTraceAI 是 server 中心化架构:**Server 是系统唯一入口**,所有外部流量(浏览器/桌面端)
只与 Server 通信;GPU 机器处于隔离网络,只有 Server 能触达;ComfyUI **绝不**对公网暴露。

```
        ┌──────────────────────────────┐
        │   Browser (web)              │
        │   Desktop (React + Tauri2)   │
        └──────────────┬───────────────┘
                       │  REST (/api/v1) + SSE
                       │  Authorization: Bearer + X-Tenant-Id
                       ▼
        ┌──────────────────────────────────────────────┐
        │                Go Server                       │
        │  对外: REST + SSE   对内: WS 长连接(worker)      │
        │  状态权威层 (job 生命周期 / 租户 / 内容)           │
        └───┬───────────────┬──────────────────┬─────────┘
            │               │                  │
   ┌────────▼──────┐  ┌─────▼──────┐    WS 长连接(内网,server→agent 下发)
   │   MySQL       │  │ Object      │           │
   │ jobs/workers/ │  │ Storage     │           │
   │ models/...    │  │ (生成结果)   │           │
   └───────────────┘  └─────────────┘           │
                                    ┌────────────┼────────────┐
                                    ▼            ▼            ▼
                            ┌───────────┐ ┌───────────┐ ┌───────────┐
                            │Worker-Agent│ │Worker-Agent│ │Worker-Agent│
                            │   -1 (Go)  │ │   -2 (Go)  │ │   -N (Go)  │
                            └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                                  │ localhost   │ localhost   │ localhost
                                  ▼ :8188       ▼ :8188       ▼ :8188
                            ┌───────────┐ ┌───────────┐ ┌───────────┐
                            │ ComfyUI-1 │ │ ComfyUI-2 │ │ ComfyUI-N │
                            └───────────┘ └───────────┘ └───────────┘
```

要点:

- **Server 单入口**:client 永远只连 server。worker/ComfyUI 的地址、prompt_id、client_id 等内部细节,契约里一律不暴露(沿用 `openapi-skeleton.md` §0.4/0.5)。
- **GPU 机器隔离**:Worker-Agent 与 ComfyUI 部署在同一台 GPU 机器,二者通过 `localhost:8188` 通信。GPU 机器**不监听公网**,只主动向 server 发起出站 WS 长连接。防火墙只需放行 agent→server 的出站连接。
- **ComfyUI 不公开**:`:8188` 只对本机 agent 可见。任何对 ComfyUI 的调用(`/prompt`、`/ws`、`/history`、`/view`)都只发生在 agent 进程内。
- **MySQL = job 状态权威源**:每个 worker 上报的进度/结果先落 DB,再 fanout 到 SSE。SSE 断线不丢状态(client 可 `GET /workflow-runs/{id}` 补偿)。
- **Object Storage**:生成结果由 agent 从 ComfyUI 取出后上传对象存储,server 仅持有 storageURL,不中转大文件二进制(资产下载走 server 代理或直链,按部署定)。

**对象存储**:开发阶段用 **MinIO**(自建,S3 兼容),正式上线切 **AWS S3**。
两者均兼容 S3 协议,server 与 worker-agent 统一用一套 S3 SDK,仅按环境替换
`endpoint` / `accessKey` / `secretKey` / `bucket`。存储凭证属于按环境变化的基础设施配置,
归 server 运行时配置(`pkg/config` / `config.yaml` + 环境变量),不进 eye-trace-config。

---

## 2. 开发阶段规划

两阶段推进。**Phase 1 先把 client 仿站做完**(纯 YAML/mock 驱动,零后端依赖),
**Phase 2 再做一条管线的垂直打通**(文生图端到端,贯穿全部四层)。

### Phase 1 — Client 仿站完工(YAML / mock 驱动)

目标:产出一个**完整可导航的产品外壳**,所有运营内容由 YAML 驱动,除 `/content/home`
(已可工作)外不发任何真实后端请求。完整路由清单见 `liblib-client-function-map.md` §2,
14 条路由摘要如下,每条均为 YAML/mock 驱动:

| # | 路由 | 页面 | 本阶段交付物(UI only) |
|---|---|---|---|
| 1 | `/` 或 `/inspiration` | 灵感首页 | Banner 轮播、推荐模型条、类型页签、分类 chip、瀑布流卡片(数据走 `/content/home`,已工作) |
| 2 | `/search?keyword=` | 搜索结果 | 搜索框回填、资源类型页签、过滤入口、结果卡片(mock) |
| 3 | `/ai-tool/image-generator` | 图片生成 | 模式页签、prompt 文本域、模型选择器、风格/上传 chip、比例/数量、生成按钮(禁用态)、"做同款"卡片 |
| 4 | `/ai-tool/video-generator` | 视频生成 | 同工具框架 + 视频/图片/音频上传、视频模型、参考模式、比例/分辨率/时长/配音参数 |
| 5 | `/sd` | WebUI | 受保护工作区;未登录登录闸;登录后展示 模型/参数/任务 面板(mock) |
| 6 | `/comfy` | ComfyUI | 受保护节点工作区;未登录登录闸;登录后展示 工作流/操作/任务 面板(mock) |
| 7 | `/pretrain` | 训练 LoRA | 受保护训练工作区;数据集上传、底模、触发词、训练进度(mock) |
| 8 | `/lib3` | AI 应用 | 应用分类网格、应用运行器、上传控件、滑块/输入、一键生成、结果/空态 |
| 9 | `/asset` | 资产 | 产品页签、资产类型选择、日期范围、收藏勾选、空态/列表态(mock) |
| 10 | `/modelinfo/:id` | 模型详情 | 画廊、标题/统计、版本、描述、作者卡、CTA、收藏/关注、版本参数、许可、评论、showcase |
| 11 | `/imageinfo/:id` | 作品详情 | 作品媒体、作者/关注、点赞/分享、日期/工具、标签、prompt/参数、关联模型、评论、做同款 |
| 12 | `/teaching` + `/teaching/:id` | 教学 | 教学分类页签、最新文章、视频教程、付费课程入口;详情页文章模板 |
| 13 | `/apis` | API 落地 | Hero CTA、能力分区、对比页签、定价方案、FAQ |
| 14 | `/userpage/me/publish` | 用户中心 | 作品/资产/收藏/任务外壳(鉴权后置) |

> 数据策略遵循 `liblib-client-function-map.md` §4 与 `client-data-split-guide.md`:
> 运营内容全部可由 `src/config/static/default.yaml` 配置,server 建好后降级为离线兜底。

本阶段 client 关键剩余缺口(必须补齐):

1. **标识符重命名**:全量 `Game AI Asset Studio → EyeTraceAI`、deeplink `gameai:// → eyetrace://`(详见 §9)。
2. **接 generate FSM 到 UI**:把 `src/features/generate/fsm.ts`(`idle→selectingModel→configuring→generating→success→error`)接到 6 个生成页的实际按钮上,configuring→generating 由"生成"按钮触发。
3. **SSE 事件源模块(stub)**:新建 `src/api/sse.ts`,本阶段**返回 mock 进度**(定时器模拟 `progress→succeeded`),让 FSM 能跑通 `generating→success`,真实 SSE 在 Phase 2 接入。
4. **6 个生成工作区页面**:`/ai-tool/image-generator`、`/ai-tool/video-generator`、`/sd`、`/comfy`、`/pretrain`、`/lib3` 的 UI 外壳。
5. **保留既有规则**:Page Registry / ViewStack / KeepAlive 不动(见 `liblib-client-function-map.md` §5)。

Phase 1 "完成"判据:14 条路由均可导航,运营内容由 YAML 驱动,生成按钮点击后看到 mock 进度并落到 success 卡片,无任何真实生成。

**Mock 数据来源**:Phase 1 的模型卡、封面图、生成参数、推荐词等直接取自 liblib 真实数据
(部分已抓取于 `eye-trace-docs/html_sourcecode/`),作为仿站 mock fixtures 铺满 14 个路由的交互。
**限制**:模型权重文件无法从 liblib 获取——Phase 1 不需要真权重,纯交互/仿站用 mock 驱动即可。
权重在 Phase 2 验证流程时从 **HuggingFace** 下载部署到 worker,再跑真实生成打通链路。

### Phase 2 — 单管线垂直打通(文生图,端到端)

目标:打通**唯一一条**管线(文生图)的全链路,贯穿 client → server → worker-agent → ComfyUI。
先不做多租户、不做多管线、不做配额,固定单 worker、单模板,验证整条数据流。

各层必须交付的最小集合:

| 层 | 必做项 |
|---|---|
| **Client** | `POST /api/v1/jobs` 提交真实任务;`new EventSource(/api/v1/jobs/{id}/events)` 订阅进度;把 §6 的真实 `sse.ts` 替换 Phase 1 的 mock;FSM `generating` 态消费真实 SSE 事件 |
| **Server** | job 端点(创建 job 落 DB,status=pending);dispatcher(`internal/job` 调 `agent_pool.Dispatch`);worker 分配(least-jobs-first);SSE fanout(agent 进度 → 订阅该 jobID 的 SSE 连接);job 状态机(pending→running→succeeded/failed/cancelled) |
| **Worker-Agent** | 启动注册(`POST /api/internal/workers/register`);WS 长连接接收 submit;查模板 + 注入参数 + `POST localhost:8188/prompt`;连 ComfyUI WS 转发进度;`execution_success` 后取结果、传对象存储、上报 complete |
| **Template** | 落地首个模板:`txt2img_sdxl_v1.json`(或 `txt2img_sd15_v1.json`)+ 对应 `*.schema.yaml`(见 §4) |

Phase 2 "完成"判据(端到端验收):

> 用户在图片生成页**输入 prompt → 点击「生成」→ 看到实时进度(步数 + 预览帧)→ 最终生成图片显示在结果卡片**。
> 全程经过真实 ComfyUI 执行,job 状态在 MySQL 中完整流转,SSE 实时回推。

前置条件:**真实模型权重(从 HuggingFace 下载,如 SDXL base + 对应 LoRA)须提前部署到 worker 的 ComfyUI `models/` 目录**;否则 `{{CHECKPOINT_NAME}}` 无对应权重,ComfyUI 将返回 `GENERATION_FAILED`。

---

## 3. Worker-Agent 详细设计

### 3.1 架构定位

每台 GPU 机器上跑一个 Go 二进制 `eyetrace-worker-agent`,与 ComfyUI 部署在同一机器。

规则:

- Agent 是**唯一**能调 `localhost:8188` 的进程。Server 永远不直接联系 ComfyUI。
- Agent 拥有 ComfyUI 进程的生命周期管理权,可在 ComfyUI 崩溃后重启它。
- ComfyUI 单线程单进程,每个实例同时只处理 1 个 prompt。Agent 不并发提交。
- Agent 主动向 Server 发起出站 WS 连接(server 不主动连 agent),防火墙只需允许 agent 的出站连接。

### 3.2 注册流程

Agent 启动后立即注册:

```
Agent → POST /api/internal/workers/register
Body:
{
  "workerID": "<hostname>-<uuid>",   // 唯一,重启不变(持久化到本地)
  "capabilities": ["txt2img_sdxl_v1", "txt2vid_wan_v1", ...],  // 支持的 templateID 列表
  "gpuModel": "NVIDIA RTX 4090",
  "vramGB": 24
}

Server 响应:
{
  "token": "<worker-jwt>",           // 后续 WS 认证用
  "heartbeatIntervalSec": 30
}
```

注册成功后 worker 记录写入 `workers` 表,status=online。

### 3.3 长连接协议

注册后建立 WebSocket 长连接:

```
Agent → ws://server/api/internal/workers/stream?token=<token>
```

**Server → Agent 消息(JSON)**:

| type | 字段 | 说明 |
|---|---|---|
| `submit` | `jobID`, `templateID`, `params{}`, `priority` | 下发任务 |
| `cancel` | `jobID` | 取消任务(agent 若已提交到 ComfyUI 则发 interrupt) |
| `ping` | — | 心跳探测 |

**Agent → Server 消息(JSON)**:

| type | 字段 | 说明 |
|---|---|---|
| `accepted` | `jobID` | 已接单,server 将 DB status pending→running |
| `progress` | `jobID`, `step`, `totalSteps` | 执行进度(来自 ComfyUI WS `progress` 事件) |
| `preview` | `jobID`, `dataB64`, `format:"jpeg"` | 中间预览帧(来自 ComfyUI WS 二进制 type-1 frame,base64 编码) |
| `complete` | `jobID`, `outputs:[{storageURL}]` | 完成,outputs 已上传对象存储 |
| `failed` | `jobID`, `error` | 失败原因 |
| `pong` | `vramFreeGB` | 响应 ping,带 GPU 空闲显存 |

### 3.4 Agent 本地执行流程

```
1. 收到 {type:"submit", jobID, templateID, params}
2. 发送 {type:"accepted", jobID}
3. 从本地缓存读取 templateID 对应的 JSON 模板
4. 深拷贝模板,注入 params(见 §4 占位符替换规则)
5. POST localhost:8188/prompt
   body: { "prompt": <graph>, "client_id": jobID, "prompt_id": jobID }
6. 连接 localhost:8188/ws?clientId=jobID
7. 消费 WS 事件:
   - type=progress → 发送 {type:"progress", jobID, step, totalSteps}
   - 二进制 type-1 帧 → base64 编码 → 发送 {type:"preview", jobID, dataB64, format:"jpeg"}
   - executing{node:null} 或 execution_success → 进入步骤 8
   - execution_error → 发送 {type:"failed", jobID, error} → 结束
8. GET localhost:8188/history/jobID → 提取 output 文件列表
9. 对每个 output: GET localhost:8188/view?filename=&subfolder=&type=output → 上传到 S3 兼容对象存储(开发 MinIO / 上线 AWS S3),得到 object key/URL → 随 `complete` 消息回传 server → 落 `job_outputs.storage_url`
10. 发送 {type:"complete", jobID, outputs:[{storageURL}]}
```

> `client_id` 与 `prompt_id` 均填 jobID。ComfyUI 内部 prompt_id 不再是独立字段,
> server 永远不拿到 ComfyUI 的 prompt_id(隔离边界)。

### 3.5 断线重连与容错

- 断线后按指数退避重连:`1s → 2s → 4s → 8s → … → 60s(上限)`。
- 重连期间 agent 本地继续执行已接单的 job,结果积压到本地队列,重连成功后批量上报。
- Server 端:job 超过 `2 × heartbeatIntervalSec` 未收到 `accepted`,自动重入 pending 队列。
- Worker 连续 3 次心跳超时 → server 将其标记 offline,已分配给该 worker 的 pending job 重新调度。

### 3.6 Server 侧 Worker 池(`internal/agent_pool`)

- 维护 `map[workerID]→wsConn` 内存连接表。
- 调度策略:least-jobs-first(空闲 worker 优先,空闲相同时按 vramFreeGB 降序)。
- 能力过滤:先按 `capabilities` 字段筛选支持目标 templateID 的 worker,再按负载排序。
- 健康检查:每 `heartbeatIntervalSec` 发一次 `ping`,未在 `3×interval` 内收到 `pong` → 标记 offline。

---

## 4. 参数化管线模板系统

把 ComfyUI 的 API-format JSON graph 做成**带占位符的参数化模板**,真相源存于
`eye-trace-config/source/pipelines/`，经导出器灌入 server DB（`pipeline_templates` 表）。
运行时 server 从 DB 取模板，做占位符替换 + 校验后提交给 worker-agent。

### 4.1 目录结构

```
eye-trace-config/
└── source/
    ├── pipelines/
    │   └── {templateID}.json          # ComfyUI API-format graph,含 {{PARAM_NAME}} 占位符
    └── pipeline-schemas/
        └── {templateID}.schema.yaml   # 参数 schema:名称/类型/默认值/是否必填/校验规则
```

这是真相源;经 `to-seed.ts` 导出灌入 server DB,server 不直接读此目录(详见 §8)。

### 4.2 首批模板 ID

| templateID | 用途 |
|---|---|
| `txt2img_sdxl_v1` | 文生图 SDXL(Phase 2 首个落地) |
| `txt2img_sd15_v1` | 文生图 SD1.5 |
| `img2img_sdxl_v1` | 图生图 SDXL |
| `txt2vid_wan_v1` | 文生视频 WanVideo |
| `img2vid_wan_v1` | 图生视频 WanVideo |

> Worker 已预装 WanVideoWrapper、LTXVideo、AnimateDiff、EasyAnimate、ControlNet、
> IPAdapter、RMBG,上述图像/视频管线节点依赖均已就绪。

### 4.3 占位符约定

模板 JSON 内统一使用 `{{PARAM_NAME}}` 标记:

```
{{POSITIVE_PROMPT}}   {{NEGATIVE_PROMPT}}   {{CHECKPOINT_NAME}}
{{LORA_NAME}}         {{LORA_STRENGTH}}     {{WIDTH}}
{{HEIGHT}}            {{SEED}}              {{STEPS}}
{{CFG_SCALE}}         {{SAMPLER_NAME}}      {{SCHEDULER}}
{{BATCH_SIZE}}        {{VIDEO_FRAMES}}      {{MOTION_BUCKET_ID}}
```

文生图用前 13 个;视频管线追加 `{{VIDEO_FRAMES}}`、`{{MOTION_BUCKET_ID}}`。

### 4.4 Schema 示例(txt2img_sdxl_v1.schema.yaml)

```yaml
templateID: txt2img_sdxl_v1
version: 1

params:
  POSITIVE_PROMPT:
    type: string
    required: true

  NEGATIVE_PROMPT:
    type: string
    required: false
    default: ""

  CHECKPOINT_NAME:
    type: string
    required: true

  LORA_NAME:
    type: string
    required: false
    default: ""

  LORA_STRENGTH:
    type: float
    required: false
    default: 0.8
    constraints: { min: 0.0, max: 2.0 }

  WIDTH:
    type: int
    required: false
    default: 1024
    constraints: { min: 512, max: 2048, step: 64 }

  HEIGHT:
    type: int
    required: false
    default: 1024
    constraints: { min: 512, max: 2048, step: 64 }

  SEED:
    type: int
    required: false
    default: -1
    constraints: { min: -1, max: 2147483647 }

  STEPS:
    type: int
    required: false
    default: 20
    constraints: { min: 1, max: 150 }

  CFG_SCALE:
    type: float
    required: false
    default: 7.0
    constraints: { min: 1.0, max: 30.0 }

  SAMPLER_NAME:
    type: enum
    required: false
    default: euler
    constraints:
      values: [euler, dpm_2, dpmpp_2m, dpmpp_sde, ddim, uni_pc]

  SCHEDULER:
    type: enum
    required: false
    default: karras
    constraints:
      values: [karras, normal, sgm_uniform, simple, ddim_uniform]

  BATCH_SIZE:
    type: int
    required: false
    default: 1
    constraints: { min: 1, max: 4 }
```

### 4.5 注入与校验逻辑(Go server / agent)

```
1. 模板和 schema 已在 DB 中（`pipeline_templates` 表，由 seed 导入）；server 启动时查询并缓存，无需读 config 目录
2. 缓存可热更新：admin 修改 DB 中的模板后触发 server reload，无需重新部署
3. 每个 job 到来时:
   a. 按 templateID 查校验规则,对 params 做类型/范围/枚举校验 → 失败返回 VALIDATION_FAILED
   b. 深拷贝模板 JSON 字符串
   c. 对每个 {{PARAM_NAME}}: strings.ReplaceAll(json, "{{PARAM_NAME}}", quote(value))
   d. json.Valid(result) 确认结果是合法 JSON → 失败属于模板 bug,返回 INTERNAL
   e. 将最终 graph 提交给 ComfyUI
```

> 占位符替换用字符串替换而非 JSON 操作,前提是 JSON 模板中对应占位符两侧已有合法的 JSON
> 界定符(字符串值用 `"{{PARAM}}"`,数值型占位符不加引号)。模板作者须保证语义正确。

### 4.6 版本管理

- 模板真相源 git-versioned 于 `eye-trace-config/source/pipelines/`。
- 经 `exporters/to-seed.ts` 导出,由 server migrate/seed 命令灌入 `pipeline_templates` 表。
- job 派发时把当时 DB 中的模板内容 snapshot 进 `jobs.template_snapshot`(JSON 字段),
  并记 `template_id` + `template_version`(语义版本或 content hash)。
- 历史复现:直接用 `jobs.template_snapshot` 重放,不依赖 config 当前状态,也不需要 git checkout。

---

## 5. Server 核心模块(原 `go-server-structure.md` 内容已并入,无独立分卷)

本节只列**新增项与变更项**。现有目录结构、分层原则(handler→service→repo 单向依赖)、
handler ≤ 30 行、comfy-user 安全边界等规则见 `go-server-structure.md`,全部保持有效。

### 5.1 新增模块

**`internal/agent_pool/`** — Worker WS 连接池与调度器

```
internal/agent_pool/
├── pool.go       # map[workerID]→conn,注册/注销/查询
├── dispatch.go   # Dispatch(job) → 筛 capabilities + least-jobs-first 分配
└── health.go     # 心跳追踪,3×interval 未响应 → offline,触发 job 重调度
```

**`pkg/wspool/`** — 通用 WS Hub(agent_pool 与 SSE fanout 复用)

```
pkg/wspool/
└── hub.go        # 通用 hub:map[topic]→[]conn; Subscribe/Publish/Close 接口
```

### 5.2 现有模块变更

**`internal/comfy/`** — 此包**不在 server 中**。

与 `go-server-structure.md` 的一个关键差异:在 Worker-Agent 架构下,
server 永远不直接调用 ComfyUI。`internal/comfy/` 是**agent 二进制的包**,不是 server 的包。
server 的 `internal/comfy/` 原有职责重新分配:

| 原职责 | 新归属 |
|---|---|
| `client.go`(调 ComfyUI REST) | agent binary 的 `internal/comfy/` |
| `wspool.go`(连 ComfyUI WS) | agent binary 的 `internal/comfy/` |
| `dispatch.go`(ComfyUI 事件路由) | agent binary 的 `internal/comfy/` |
| `pool.go`(多 ComfyUI 实例调度) | 退化为单实例,由 agent 本地管理 |
| `binding.go`(tenant_comfy_bindings) | 保留 server,Phase 3 多租户时启用 |

> server `internal/comfy/` 在 Phase 2 可为空包,Phase 3 多租户时再引入 binding 逻辑。

**`internal/job/` dispatch 逻辑补充**

```
job created → status=pending → DB INSERT
  ↓
agent_pool.Dispatch(job)
  ├── 有空闲 worker → 发送 WS {type:"submit",...} → 等 accepted → DB status=running
  └── 无空闲 worker → job 留 pending,当 worker reconnect/complete 时触发重新 Dispatch
```

job 状态机(所有转移必须经过 DB):

```
pending → running    (agent 发 accepted)
running → succeeded  (agent 发 complete,outputs 写 job_outputs 表)
running → failed     (agent 发 failed)
pending/running → cancelled  (client 发 DELETE /workflow-runs/{id})
```

**`internal/api/handler/`** — 新增 worker 内部端点

```
handler/worker.go   # POST /api/internal/workers/register
                    # WS  /api/internal/workers/stream?token=...
```

内部端点用独立 router group,不走 tenant middleware,改走 worker-token 鉴权。

**`pkg/config/` 配置扩展**

`pkg/config` 负责以下配置来源:

- `pkg/config` 加载 `internal/config/generated/config_gen.go` 中的编译期枚举常量（由 `eye-trace-config/dist/server/config_gen.go` vendor 进来），不在运行时读 config 目录
- pipeline-schemas 从 DB（`pipeline_templates` 表）加载，不从 config 目录文件系统读取
- 自身 `config.yaml` / 环境变量:DB DSN、监听端口、worker auth token 密钥
- 对象存储:storageEndpoint / storageAccessKey / storageSecretKey / storageBucket / storageRegion / storageUseSSL(开发 MinIO,上线 AWS S3;同一套 S3 SDK 切换)

**SSE fanout 链路(补充 go-server-structure.md §2.2)**

worker 上报进度的 fanout 路径:

```
Agent WS {type:"progress"/"preview"/"complete"/"failed", jobID, ...}
  → handler/worker.go      : 解析消息类型
  → job/service.go         : 先写 DB(状态权威落库)
  → job/hub.go             : 按 jobID fanout 给所有 SSE 订阅者
  → handler/run_events.go  : 写 text/event-stream 给 client
```

此链路与 `go-server-structure.md` §2.2 描述的 WS→SSE 中继逻辑一致,
只是 WS 的来源从 ComfyUI 换成了 Worker-Agent。`job/hub.go` 结构不变。

---

## 6. Client 侧实时进度接入

### 6.1 新建 `src/api/sse.ts`

对 `EventSource` 的类型化封装,事件形状对齐 `openapi-skeleton.md` §4 的 SSE 信封:

```ts
// SSE 事件类型(对齐 openapi-skeleton.md SSE 信封)
export type SseEvent =
  | { event: "queued";    data: { jobId: string; status: "pending"; position: number } }
  | { event: "progress";  data: { jobId: string; status: "running"; percent: number } }  // 0.0–1.0
  | { event: "succeeded"; data: { jobId: string; status: "succeeded"; resultUrl: string; thumbnailUrl?: string } }
  | { event: "failed";    data: { jobId: string; status: "failed"; code: string; message: string } }
  | { event: "preview";   data: { jobId: string; dataURL: string } }  // 中间帧

export interface SseSubscription {
  unsubscribe(): void;
}

export function subscribeJobEvents(
  jobId: string,
  callbacks: { [K in SseEvent["event"]]?: (data: Extract<SseEvent, { event: K }>["data"]) => void },
  options?: { backoffMs?: number }
): SseSubscription;
```

实现要点:

- 断线后按退避重连(默认 `1s → 2s → 4s → 30s max`)。
- 携带 `Last-Event-ID` header 让 server 从断点续发。
- 终态事件(`succeeded`/`failed`)收到后自动 `close()`。
- **Phase 1**:实现为定时器 mock,无 `EventSource`。**Phase 2**:换成真实实现,接口不变。

### 6.2 接入 FSM(`src/features/generate/fsm.ts`)

```
FSM: configuring → generating
  action: POST /api/v1/jobs {modelId, params}
          → 拿到 jobId
          → subscribeJobEvents(jobId, {
              progress: ({percent}) => send({type:"PROGRESS", percent}),
              preview:  ({dataURL}) => updatePreview(dataURL),
              succeeded:({resultUrl})=> send({type:"SUCCESS", resultUrl}),
              failed:   ({message}) => send({type:"ERROR", error: message}),
            })

generating → success   (收到 SUCCESS 事件)
generating → error     (收到 ERROR 事件)
```

FSM 的 `PROGRESS` / `SUCCESS` / `ERROR` 事件名与 `openapi-skeleton.md` §1.2 的映射表完全对齐,
client 无需额外适配层。

### 6.3 提交流程

```ts
// FSM action:configuring → generating
async function submitJob(ctx: FsmContext) {
  const { jobId } = await apiClient.post("/api/v1/jobs", {
    modelId: ctx.selectedModelId,
    params:  ctx.params,
  });
  return { jobId };   // 存入 FSM context,供 cancel / 结果展示使用
}
```

### 6.4 Auth header TODO

`src/api/client.ts` 预留注入点:

```ts
// TODO-Auth: 从 auth store 取 accessToken 后注入
// headers["Authorization"] = `Bearer ${authStore.token}`;
```

Phase 1/2 不实现鉴权,此行注释保留,Phase 3 多租户时解注。

### 6.5 结果图片展示

生成完成后,`resultUrl` 是 `/api/v1/assets/{id}/content` 的相对路径。
client 调用现有 `openImagePreview`(来自 `src/app/layer/manager.ts`)展示完成图:

```ts
// on SUCCESS event
openImagePreview({ src: apiClient.resolveUrl(resultUrl), jobId });
```

---

## 7. 数据库核心表设计(MySQL)

MVP 核心表。所有表 `utf8mb4` + InnoDB。

### 7.1 workers

```sql
CREATE TABLE workers (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  worker_id     VARCHAR(64)  NOT NULL UNIQUE,
  hostname      VARCHAR(255) NOT NULL,
  status        ENUM('online','offline','busy') NOT NULL DEFAULT 'offline',
  gpu_model     VARCHAR(128),
  vram_gb       INT,
  capabilities  JSON,                       -- 支持的 templateID 数组
  last_heartbeat TIMESTAMP NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 7.2 jobs

```sql
CREATE TABLE jobs (
  id                BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id           BIGINT NULL,            -- Phase 3 多租户前可空
  pipeline_id       VARCHAR(64)  NOT NULL,  -- templateID
  template_version  VARCHAR(40)  NOT NULL,  -- 语义版本或 content hash,用于复现
  template_snapshot LONGTEXT NULL,       -- 派发时的模板 JSON 快照，用于历史复现
  params            JSON         NOT NULL,
  status            ENUM('pending','running','succeeded','failed','cancelled')
                      NOT NULL DEFAULT 'pending',
  worker_id         VARCHAR(64)  NULL,      -- 分配到的 worker
  comfyui_prompt_id VARCHAR(64)  NULL,      -- = jobID,server 内部不暴露
  error_msg         TEXT         NULL,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  started_at        TIMESTAMP NULL,
  finished_at       TIMESTAMP NULL,
  INDEX idx_status_created (status, created_at),
  INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 7.3 job_outputs

```sql
CREATE TABLE job_outputs (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  job_id          BIGINT       NOT NULL,
  filename        VARCHAR(255) NOT NULL,
  storage_url     VARCHAR(512) NOT NULL,
  media_type      ENUM('image','video') NOT NULL,
  width           INT NULL,
  height          INT NULL,
  file_size_bytes BIGINT NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_job (job_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 7.4 models

```sql
CREATE TABLE models (
  id                BIGINT AUTO_INCREMENT PRIMARY KEY,
  name              VARCHAR(255) NOT NULL,
  type              ENUM('checkpoint','lora','template','workflow','video') NOT NULL,
  base_model        VARCHAR(64) NULL,        -- sd15 / sdxl / flux / ...
  sub_tab_id        VARCHAR(64) NULL,        -- 归属 sub_tab("image-all" / "video-hot" / NULL=无归属);由 home_sub_tabs.by_type_json 派生;见 model-sub-tab-flow.md §3 §4
  civitai_id        VARCHAR(64) NULL,
  preview_image_url VARCHAR(512) NULL,
  tags              JSON,
  meta              JSON,
  created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_type (type),
  INDEX idx_sub_tab (sub_tab_id),
  FULLTEXT idx_name_ft (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

> `models.type` 枚举与 `client-data-split-guide.md` §5.2 的 `ModelUploadRequest.modelType`
> (`checkpoint/lora/template/workflow/video`)保持一致。
>
> `models.sub_tab_id` 是首页瀑布按 sub_tab 过滤用的归属标签,NULL = 不归属(归入"全部");
> 不建外键(JSON 列无法强引用),靠运营 / seed 阶段保证一致性。
> 契约字段 `ModelSummary.subTabId` 与 `GetModelsParams.subTabId` 见 `model-sub-tab-flow.md`。

### 7.5 Worker 调度的并发安全:SELECT … FOR UPDATE SKIP LOCKED

多 server 实例(或单实例多 goroutine)从 `jobs` 表抢 pending job 时,
用 `FOR UPDATE SKIP LOCKED` 避免两个调度器抢到同一 job:

```sql
START TRANSACTION;

SELECT id, pipeline_id, params, template_version
FROM jobs
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;          -- 已被其它事务锁定的行直接跳过,不阻塞

-- 取到后立即占用,避免重复派发
UPDATE jobs SET status='running', worker_id=?, started_at=NOW() WHERE id=?;

COMMIT;
```

MySQL 8.0+ 支持 `SKIP LOCKED`。该模式让 dispatcher 天然支持横向扩展,
无需额外分布式锁。

---

## 8. eye-trace-config 配置中心模型

### 8.1 模型概述

`eye-trace-config` 是配置的**唯一编辑入口**,采用"**真相源 → 导出 → 内化**"模型。
本节**取代**早期文档中"client/server 启动时各自读取同一份 yaml"的假设:
两端在运行时和构建时**都不读** `eye-trace-config` 目录本身,跨越边界的只有导出器产出的制品。

- **真相源**:人工 / 运营在 `source/` 下手动编辑,是配置的唯一权威输入。
- **导出**:`npm run export` 先做 schema 校验(失败即停),再按类别产出 `dist/`。
- **内化**:契约枚举编译进二进制;管线模板与运营内容灌入 server DB。

### 8.2 三类配置导出

| 类型 | 真相源 | 导出目标 | 内化方式 |
|---|---|---|---|
| 契约枚举（supportedImageSizes / jobStatus / typeTabIds / mediaType / errorCode / FSM states） | `source/enums.yaml` | → 代码 | `config.gen.ts`(client) + `config_gen.go`(server)，编译进二进制 |
| 管线模板（ComfyUI API-format graph + 参数 schema） | `source/pipelines/` + `source/pipeline-schemas/` | → DB seed | 导出成 seed 灌进 server DB；admin 可改不重新部署；job 派发时 snapshot 模板内容保证复现 |
| 运营内容（banner / 推荐 / 模型上架） | `source/operational/` | → DB seed | 灌进 server DB，client 走 `/content/home` 拉取 |

### 8.3 目录结构

```
eye-trace-config/
├── source/                    # 真相源：人工/运营手动编辑，唯一编辑入口
│   ├── enums.yaml
│   ├── pipelines/
│   │   └── txt2img_sdxl_v1.json
│   ├── pipeline-schemas/
│   │   └── txt2img_sdxl_v1.schema.yaml
│   └── operational/
│       ├── home.yaml
│       └── model-catalog.yaml
├── schema/                    # JSON Schema，校验 source 合法性
│   └── enums.schema.json
├── exporters/                 # 导出器（Node/TS 脚本）
│   ├── to-client.ts
│   ├── to-server.ts
│   └── to-seed.ts
├── dist/                      # 导出产物（git tracked）
│   ├── client/config.gen.ts
│   ├── server/config_gen.go
│   └── seed/
│       ├── pipelines.json
│       └── operational.json
└── package.json
```

### 8.4 导出流程

1. 编辑 `source/`。
2. `npm run export`:schema 校验 → 失败即停 → 产出 `dist/`。
3. 内化:
   - client:`dist/client/config.gen.ts` → vendor 进 `eye-trace-client/src/config/generated/config.gen.ts`(commit)。
   - server:`dist/server/config_gen.go` → vendor 进 server `internal/config/generated/`(commit)。
   - server DB:`dist/seed/*.json` → migrate/seed 命令灌入 DB。

> 铁律:client/server 在运行时与构建时都不读 `eye-trace-config` 目录本身;
> DB 是管线模板与运营内容的权威。

### 8.5 首批待创建文件

- [ ] **`source/enums.yaml`**:补 `mediaType: [image, video]` 与 `errorCode: [...]`
      (`openapi-skeleton.md` 已在用,当前缺失):

  ```yaml
  # === 媒体类型 ===
  mediaType:
    - image
    - video

  # === 错误码(对齐 openapi-skeleton.md ErrorCode) ===
  errorCode:
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
  ```

  > 现有 `enums.yaml` 已含 `supportedImageSizes`、`typeTabIds`、`jobStatus`,本次仅追加上面两组。

- [ ] **`source/pipelines/txt2img_sdxl_v1.json`** + **`source/pipeline-schemas/txt2img_sdxl_v1.schema.yaml`**:Phase 2 首个模板(见 §4)。
- [ ] **`exporters/to-client.ts` / `to-server.ts` / `to-seed.ts`**:三类导出器。
- [ ] **`schema/enums.schema.json`**:校验 `source/enums.yaml` 合法性的 JSON Schema。

### 8.6 交叉引用

- §4 描述模板格式与 `{{PARAM_NAME}}` 占位符约定。
- 本节描述模板如何从 config 导出(`to-seed.ts`)灌入 DB(`pipeline_templates` 表)。
- §7 `jobs` 表的 `template_snapshot` 字段在 job 派发时快照模板内容,保证历史复现。

---

## 9. 命名迁移任务

将 `Game AI Asset Studio` 品牌与 `gameai://` deeplink 全量迁移到 `EyeTraceAI` / `eyetrace://`:

- [ ] **`eye-trace-client/src-tauri/tauri.conf.json`**:
  - `productName`: `"Game AI Asset Studio"` → `"EyeTraceAI"`
  - `identifier`: `"studio.gameai.asset"` → `"ai.eyetrace.app"`
  - deeplink scheme: `gameai` → `eyetrace`
- [ ] **`eye-trace-client/package.json`**:`name`、`description` 字段。
- [ ] **`eye-trace-client/src-tauri/Cargo.toml`**:`name`、`description` 字段。
- [ ] **`eye-trace-client/src/platforms/desktop/deep-link.ts`**:`gameai://` URL 解析 → `eyetrace://`。
- [ ] **所有硬编码 "Game AI" 品牌的 `appConfig` 引用**:全局搜索 `Game AI` / `gameai` 替换。

> 平台分流保持不变:web=`/api`,desktop=`http://localhost:8080/api`。
> 迁移只动品牌标识与 scheme,不动 API base path。


---

---

---

---

---
