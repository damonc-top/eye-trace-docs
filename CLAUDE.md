# CLAUDE.md

本文件是 `eye-trace-docs/` 子仓库的**目录册**:为该目录下每份文档标注用途与关键章节指针,让读者快速判断"该读哪一份"。本身不做规范沉淀,具体规范全部下沉到各子文档。

## 仓库定位

- **角色**:整个 eye-trace monorepo 的**契约 / 架构唯一真相源**。所有架构、OpenAPI 契约、Phase 规划、数据流设计在此沉淀。
- **规则**:**先读再写**。改 client / server 代码前,先确认本目录有无相关契约或架构约束。
- **语言**:全中文(英文遗留文档以原文为准,如 `liblib-client-function-map.md`)。
- **与根 `CLAUDE.md` 关系**:根 `CLAUDE.md` 只索引本目录,不复制内容;本文件对每份文档做更细的章节指针,二者不冲突。

## 文档清单

> 顺序:架构总纲 → 客户端 → 服务端/契约 → 数据流(首页/模型) → 仿站参照 → 运维。所有路径相对本目录。

| 文件 | 一句话用途 | 关键章节指针 / 必读点 |
|---|---|---|
| `project_guide.md` | **顶层架构总纲**,统合全局拓扑、阶段规划、Worker-Agent、管线模板、DB 表、配置中心、命名迁移 | §1 全局拓扑;§2 开发阶段规划(Phase 1/2);§3 Worker-Agent 详细设计;§4 参数化管线模板系统;§5 Server 核心模块;§6 Client 实时进度接入;§7 数据库核心表(workers/jobs/job_outputs/models);§8 eye-trace-config 配置中心;§9 命名迁移任务 |
| `client-architecture.md` | Phase 1 客户端**实施蓝本**:14 路由外壳、AppShell、UI 原语、设计 token、命名迁移落点、PR 拆分 | §1 目标与边界(1.2 不做的事);§4 目录骨架与模块边界;§5 14 路由页面外壳清单(5.A 六个生成页);§6 AppShell 形态选择器;§7 数据获取与回退;§8 UI 原语与组件层级;§9 设计 token 与断点;§12 命名迁移;§14 Phase 1 完成判据;§15 PR 拆分建议 |
| `openapi-skeleton.md` | **client↔server 唯一契约真相源**(OpenAPI 3.1)。client 用 `openapi-typescript`、server 用 `oapi-codegen`,**禁止手写请求/响应类型** | §0 不可违背的设计决策(传输层/版本/鉴权/租户隔离/job 状态权威/幂等);§1 与 client FSM 对齐(1.1 JobStatus↔FSM、1.2 SSE↔FSM);§2 端点清单 MVP;§3 核心 Schema 骨架;§4 SSE 事件信封;§5 代码生成落地 |
| `client_data_split.md` | 前后端**数据拆分原则**:运营内容 vs 配置 vs taxonomy 三类分家 | §0 核心判定标准("变了要不要重新打包发版");§3 统一内容加载层 `loadHomeContent()`;§4 逐字段拆分清单(4.1 留 YAML / 4.2 迁 server / 4.3 taxonomy / 4.4 修复 fallback / 4.5 进 eye-trace-config);§5 模型数据统一格式(UGC+admin);§6 落地顺序 |
| `model-detail-flow.md` | 模型详情页 `/modelinfo/:id` 的**数据设计与迭代依据**(已落地生产级) | §1 数据链路总览(页面区域→JSON→DB 表→API→前端);§2 Detail 主数据;§3 Author;§4 Comments;§5 Return Pics;§6 Importer 规范;§7 前端消费边界;§9 后续迭代约束 |
| `model-summary-card-flow.md` | `GET /api/v1/models` 列表响应 + 瀑布卡 `ModelSummary` 字段 + `summaryToCard` 映射(2026-06-27 实装版) | §1 当前契约(ModelSummary schema、MediaType/ModelType 枚举);§2 server 实现;§3 client 实现(`summaryToCard`);§4 端到端验证;§5 待办(Phase 2) |
| `home-banner-slot-flow.md` | `/api/v1/content/home` 的 `bannerSlots` 字段**端到端打通记录**(设计-落地-排查-完成) | §1 设计(1.1 三种 slot 类型 carousel/static/staticActions、1.2 跳转目标、1.3 契约位置);§2 落地;§3 排查;§4 完成 |
| `home-recommend-flow.md` | `/api/v1/content/home` 的 `recommend` 字段端到端打通(与 banner 同构:mid-shape → URLBuilder → openapi marshaling) | §1 设计(数据形态源自 liblib 推荐块);§2 落地;§3 排查;§4 完成 |
| `home-type-tabs-sub-tabs-flow.md` | `/api/v1/content/home` 的 `typeTabs` / `subTabs` 字段端到端打通(单例+version+ETag,纯文本无资源) | §1 设计(源自 default.yaml);§2 落地;§3 排查;§4 完成 |
| `liblib-client-function-map.md` | liblib.art 仿站**功能面映射**:1:1 复制可见 client 功能表面,再反推后端 API(英文) | §1 Product Shell;§2 Routes And Pages;§3 Core Interaction Inventory;§4 Demo Data Strategy;§5 First Development Milestone;§6 Live Page Capture Rule;§7 Backend Reverse Map |
| `minio_operations_manual.md` | MinIO 对象存储**运维手册**:上传素材、写 assets 表、让前端取到图 | §0 一图看懂流程;§1 一次性环境检查;§2 对象 key 命名规范;§3 上传到 MinIO;§4 写 assets 表;§5 让前端拿到图(5a/5b/5c);§6 验证;§7 常见操作脚本(删/替换/查孤儿);§8 启动 MinIO;§9 别踩的坑 |
| `DEPLOY.md` | Eye Trace Server **部署指南**:macOS / Linux / Windows 三平台手动部署 | §0 一图看懂流程;§1 系统级依赖;§2 macOS(2.5 config/环境变量、2.6 migration+seed、2.7 编启、2.8 Smoke 自测);§3 Linux(systemd 托管);§4 Windows(NSSM 服务);§5 跑测试;§6 常见问题;§7 升级/回滚 |

### 非 .md 但与本目录契约强相关

| 文件 | 用途 |
|---|---|
| `openapi.yaml` | 完整 OpenAPI 3.1 规范文件。`openapi-skeleton.md` 是其骨架指导;client/server 代码生成真正读这一份。server `make gen` 从 `eye-trace-server/backend/schema/public_api.yaml` 生成,内容应与本文件保持一致(以 server schema 仓为准时需双向核对)。 |

## 编写约定

1. **CLAUDE.md 只做大纲索引**——本文件不复制子文档内容,新增规范一律下沉到本目录新建子文档,并在此表格登记。
2. **契约变更需同步生成**:改 `openapi.yaml` / `openapi-skeleton.md` 后,client 跑 `openapi-typescript` 重生成、server 跑 `make gen`(`oapi-codegen`)重生成 `api.gen.go`,并在 CI 用 `make oapi-check` 校验一致。**禁止手写 client/server 请求/响应类型**。
3. **中文优先**;保留英文历史文档原貌(如 `liblib-client-function-map.md`)。
4. **数据流文档命名**:`<域>-<字段>-flow.md`,章节统一为 `1. 设计 → 2. 落地 → 3. 排查 → 4. 完成`,便于横向对照(banner / recommend / typeTabs 三份同构)。
5. **上位/平行文档引用**:子文档开头应注明上位文档(如 client-architecture → project_guide §2.1)与平行文档,避免内容漂移。
6. **图表与样本**:DOM 抽取引用 `tmp/` 下样本(如 `tmp/slot.html`、`tmp/py-4.html`、`tmp/*-detail.json`),不要把样本内容复制进文档。

## 与其他子仓关系

| 子仓 | 引用本目录的方式 |
|---|---|
| `eye-trace-client/` | 读 `openapi-skeleton.md` / `openapi.yaml` 生成 TS 类型与 zod schema;读 `client-architecture.md` 定目录与路由;读 `client_data_split.md` 定数据源;读 `*-flow.md` 对齐字段与映射函数 |
| `eye-trace-server/` | 读 `openapi-skeleton.md` / `openapi.yaml` 生成 `api.gen.go`;读 `project_guide.md` §5/§7 定 module 与 DB 表;读 `DEPLOY.md` 部署;读 `minio_operations_manual.md` 理解 assets 表与 storage key |
| `eye-trace-config/` | 被 `project_guide.md` §8 / `client_data_split.md` §4.5 定义三类导出与共享枚举清单;YAML 改动需回头核对这两处 |
| `ComfyUI/` | 仅 `project_guide.md` §3/§4 涉及(Worker-Agent 代理 + 管线模板),本目录不维护 ComfyUI 自身文档 |

## 注意事项

- **根 CLAUDE.md 索引有 3 处陈旧引用**:列出但本目录**不存在**的文件 —— `server_guide.md`、`liblib-replica-reference.md`、`go-server-structure.md`(后者在 `project_guide.md` 文首被提及为既有分卷,实际已不存在,其内容已被 `project_guide.md` §5 吸收)。修改这些文档前先确认是否为重命名/合并遗留,避免凭空创建与既有内容冲突的文件。
- **`openapi.yaml` 与 server 仓 schema 的双源风险**:server 实际从 `eye-trace-server/backend/schema/public_api.yaml` 生成代码;本目录的 `openapi.yaml` 是契约真相源。两份应保持一致,改动任一份须同步另一份并跑 `make oapi-check`。
- **数据流文档(`*-flow.md`)是"已落地"快照**:带时间戳与实装版本,迭代时优先改这些文档再改代码,不要从页面临摹反推字段(见 `model-detail-flow.md` 文首声明)。
- **`tmp/` 不在本目录**:样本与抓取数据在 monorepo 根的 `tmp/`,文档引用时用相对描述,不复制内容。
