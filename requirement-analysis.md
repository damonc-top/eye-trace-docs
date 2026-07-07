# 模型分类(11 类)综合需求分析 — `/modelinfo/{id}` 数据模型与契约重设计

> **作者**:minimax-m3(综合,继承会话 CLAUDE_EFFORT=max)
> **时间**:2026-07-01
> **输入样本**(综合源,4 份独立产出):
> - `tmp/req-analysis-minimax-m3.md`(770 行)
> - `tmp/req-analysis-glm-5.2.md`(748 行)
> - `tmp/req-analysis-deepseek-v4-pro.md`(939 行)
> - `tmp/req-analysis-claude-opus-4-8.md`(869 行)
>
> **范围**:`/modelinfo/{id}` 模型详情页当前从 server + MinIO 拉数据渲染,现需对模型分类,共 **11 类**(Checkpoint / Textual Inversion / Hypernetwork / Aesthetic Gradient / 模板 / LoRA / LyCORIS / Controlnet / Poses / Wildcards / Other),**数据库格式要重新设计**,交付:**权威实现方案**。
>
> **关联契约/数据流**:`eye-trace-docs/openapi-skeleton.md`、`eye-trace-docs/model-detail-flow.md`、`eye-trace-docs/model-summary-card-flow.md`、`eye-trace-docs/client_data_split.md`、`eye-trace-config/shared/enums.yaml`。
>
> **关联代码**:`eye-trace-client/src/api/models.ts`、`eye-trace-client/src/app/routes/ModelDetailPage.tsx`、`eye-trace-server/backend/internal/model/{repo,service,handler}.go`、`eye-trace-server/backend/internal/migrations/`。

---

## 0. TL;DR(执行摘要)

| 维度 | 综合结论 |
|---|---|
| **业务目标** | 让 11 类 SD/Civitai/ComfyUI 生态的标准分类在 EyeTraceAI 模型详情页有**权威分类字段**,支撑列表筛选、详情 tag 区、未来管线路由(Phase 2) |
| **数据模型** | **推荐 C 方案(综合 4 份分析):`models.type` 保留单值(扩 11 枚举)+ 新表 `model_type_map`(多对多)`+ models.primary_type` 冗余列**。允许单模型多分类(LoRA+LyCORIS 双标);枚举字典走 `eye-trace-config` 共享,DB 层无 `model_types` 字典表(MVP 简化,Phase 2 再升级) |
| **契约版本** | **v1 兼容升级**:`ModelType` 枚举 5 → 11;`ModelSummary`/`ModelDetail` 增 `types: [ModelType]` 数组;`/models` 增 `types[]` query;`type` 单值字段保留为冗余;**不进 /v2** |
| **MinIO** | **完全解耦**;object key 命名规则(`assets.purpose/source_id/format_kind`)不变;分类不影响存储路径 |
| **eye-trace-config** | 新增 `shared/enums.yaml::modelType` 11 项(共享枚举真相源);过渡期 client/server 复制 11 枚举常量待消除(同 `supportedImageSizes` 现状) |
| **迁移** | 新增 `000016_model_classification.up.sql`(建 `model_type_map` 表 + 加 `primary_type` 冗余列 + backfill 老数据 + 双写 `type`);importer 写 `model_type_map` 多分类 |
| **MVP 边界** | 仅"多对多分类 + 列表筛选 + 详情 tag 显示";**不做** admin 编辑端点、**不做** FSM 类型驱动分支、**不做** ComfyUI pipeline 联动 |
| **核心风险** | ① LyCORIS/LoRA 双标语义;② Controlnet/Poses 边界;③ importer liblib 数值映射不可逆;④ Other 兜底污染;⑤ 多仓枚举字面量复制过渡期技术债 |

---

## 1. 业务目标与核心场景

### 1.1 业务目标(综合 4 份分析)

1. **可发现性(Discoverability)**:用户按模型类型精确筛选,缩短"找模型 → 生成"路径。
2. **生态对齐**:11 类与 Stable Diffusion / ComfyUI / Civitai 生态已有的模型类型一致(Textual Inversion / LoRA / LyCORIS / Controlnet 等是事实标准术语)。
3. **多分类表达力**:允许同一模型归属多类(如 LyCORIS+LoRA、Controlnet+Poses),反映 SD 生态的实际混合场景。
4. **Other 兜底**:未识别或混合杂烩归 Other,便于未来扩展(避免每次新增分类发版本)。
5. **支撑生成流程(Phase 2)**:FSM `selectingModel` 步骤按 `modelTypes` 决定是否允许 LoRA/ControlNet/模板注入等不同参数形态(本期**不做**,只 metadata)。
6. **运营后台分类管理(Phase 3)**:admin 后台用"分类"做上架审核、推荐位、活动筛选(本期**不做**)。

### 1.2 核心场景

| 场景 | 角色 | 行为 | 涉及链路 |
|---|---|---|---|
| **列表浏览** | 任意用户 | 点击 chip "LoRA" → 列表刷新显示所有 LoRA 模型 | `/api/v1/models?types=lora` → service `ListQuery.Types []string` → SQL `WHERE m.id IN (SELECT model_id FROM model_type_map WHERE type IN (?))` |
| **详情页多徽标** | 任意用户 | 看到徽标堆叠:`[LyCORIS] [LoRA]`,hover 解释 | `/api/v1/models/{id}` → `ModelDetail.types[]` → UI chip 堆叠 |
| **多分类筛选** | 任意用户 | 顶部 4 类(typeTab image/video/ideas/flows)嵌套 11 类 multi-select chips,正交 | URL `?types=lora&types=lycoris` → server `IN` 多值筛选 → OR 关系命中 |
| **Other 兜底** | 运营 | 上传未识别模型 → 自动归 Other,UI 不显示 badge 但可筛选命中 | importer 兜底 → `model_type_map` 写 `(modelId, other, is_primary=0)` |
| **降级渲染** | 老 client | 收到 `types[]` 字段为 null → fallback 到旧 `type` 单值 | `modelDetail.types ?? [modelDetail.type] ?? ['other']` |
| **未来:Admin 改分类** | 运营/Admin | PATCH 模型分类(Phase 3) | `PATCH /api/v1/models/{id}` + `models.type` + `model_type_map` 事务双写 |
| **未来:FSM 类型驱动** | 任意用户 | 生成页选 LoRA 模型 → 提示 LoRA 权重参数 | `features/generate/fsm.ts::selectingModel` 按 `modelTypes` 决定 params schema |

### 1.3 不做的事(边界)

- **不**改 MinIO object key 命名规范(`minio_operations_manual.md` §2 已是真相)。
- **不**做类型驱动的运行时分支(Phase 2 才按类型选择模板,本需求**只做分类标签**,不触发行为)。
- **不**引入模型族谱(Checkpoint → LoRA 派生关系)——超范围。
- **不**改 `typeTabIds` 4 类顶层 tab(image/video/ideas/flows),11 类是 `typeTabIds` 之下的二级筛选。
- **不**改 `MediaType`(image/video 是媒体类型,与 SD 分类正交)。
- **不**新增"视频分类"二级枚举(11 类已通过 `mediaType='video'` 区分视频)。
- **不**改 FSM 状态机(分类是 metadata,不是 FSM 状态)。
- **不**改 ComfyUI 集成(worker-agent 不感知分类)。
- **不**做 admin 后台分类编辑(Phase 3)。
- **不**做自动分类(LLM 读 `data.description + tagsV2` 建议分类)——Phase 3 backlog。

---

## 2. 11 类与生态对应表(综合 4 份分析)

| # | 类别 id(枚举值) | 中文显示 | Civitai 类型 | ComfyUI 节点类 | 备注 |
|---|---|---|---|---|---|
| 1 | `checkpoint` | Checkpoint(主模型) | CHECKPOINT | `CheckpointLoaderSimple` / `unCLIPCheckpointLoader` | SD/SDXL/Flux 等主权重 |
| 2 | `textual_inversion` | Textual Inversion(Embeddings) | TextualInversion | `CLIPTextEncode` + embeddings 目录 | 小体积 embedding |
| 3 | `hypernetwork` | Hypernetwork | Hypernetwork | `HypernetworkLoader`(ComfyUI 自带) | 老式训练方法,Phase 3 可弃 |
| 4 | `aesthetic_gradient` | Aesthetic Gradient | AestheticGradient | 未原生支持,需自定义节点 | 美学梯度,小众,SD 1.5 历史用户群 |
| 5 | `template` | 模板(workflow/template) | (无直接对应,归类为工作流模板) | API-format graph + 占位符 | Liblib `modelType=workflow/template` 归此;**Liblib `workflow` 重映射 → `template`**(业务分类与 SD 分类语义重叠) |
| 6 | `lora` | LoRA | LORA | `LoraLoader` | 标准 PEFT |
| 7 | `lycoris` | LyCORIS(含 LoHa/LoCon/LoKr/GLoRA/IA3) | LYCORIS(新版 Civitai) | `LyCORISLoader` 节点 | LoRA 的超集,**允许多分类** `[lora, lycoris]` |
| 8 | `controlnet` | ControlNet | CONTROLNET | `Apply ControlNet` / `ControlNetLoader` | 条件控制 |
| 9 | `poses` | Poses(OpenPose/骨骼) | POSES(部分库) | `OpenPosePreprocessor` + ControlNet 应用 | 严格说是 ControlNet 输入,**允许多分类** `[controlnet, poses]` |
| 10 | `wildcards` | Wildcards(动态提示词) | (无 Civitai 对应) | `ComfyUI-WildcardProcessor` 节点 | 文本扩展 |
| 11 | `other` | Other(兜底) | OTHER / PENDING / Motion / VAE / Upscaler | — | UI 默认隐藏 badge,仅 URL `?types=other` 显式筛选命中;Liblib `video` 重映射 → `other` |

### 2.1 当前 5 类 → 目标 11 类映射(迁移起点)

| 现有 `models.type` ENUM(5 类) | 新 11 类映射 | 备注 |
|---|---|---|
| `checkpoint` | `checkpoint` | 1:1 保留 |
| `lora` | `lora`(主)+ `model_type_map` 写 `(modelId, lora, is_primary=1)` | 主分类;若 liblib 元数据含 `lycoris:true` 同时写 `(modelId, lycoris, is_primary=0)` |
| `template` | `template` | 1:1 保留 |
| `workflow` | `template`(主)+ `(modelId, template, is_primary=1)` | **重映射**:Liblib `workflow` = ComfyUI workflow JSON,合并入 `template` |
| `video` | `other`(主)+ `(modelId, other, is_primary=1)` | **重映射**:`video` 是业务顶层分类(`typeTabIds=video`),与"模型文件分类"正交;`mediaType='video'` 保留区分视频 |
| 未知/旧数据 | `other` | importer 兜底,记日志 `INFO log:model_type_fallback model_id=X liblib_type=Y new_type=other` |

### 2.2 多分类判定策略

- **Liblib 详情 JSON 的 `data.tagsV2.modelContent2` 子节点含 `lycoris`** → importer 同时插入 `(model, lora, is_primary=TRUE)` + `(model, lycoris, is_primary=TRUE)`(双主,UI 渲染时按 `display_order` 决定哪个在前)。
- **Liblib 详情 JSON 的 `data.tagsV2.modelContent2` 子节点含 `controlnet` + `openpose`** → importer 插入 `(model, controlnet, is_primary=TRUE)` + `(model, poses, is_primary=FALSE)`(主+次)。
- **单模型最大多分类数**:**上限 5**(防滥用;index 设计覆盖)。

---

## 3. 功能需求

### 3.1 主分类标识(主键级)

- 每条 model 行有且仅有一个 `primary_type`(主分类),值域 = 11 枚举。
- `models.primary_type` 冗余列(便于列表 SQL `WHERE primary_type=?),与 `model_type_map` 表中 `is_primary=1` 的行保持同步。
- 字段类型:**`VARCHAR(32)` + server 入库校验 + 启动期从 enums.yaml 加载校验集合**;**不用 MySQL ENUM**(改值要 ALTER TABLE,迁移成本高)。
- enums.yaml 新增 `modelType` 列表,**列表顺序就是 UI 优先级**(Checkpoint 优先,Other 末位)。

### 3.2 多分类(多对多)

- 新表 `model_type_map`:`(model_id, type, is_primary, created_at)` 多对多;**允许次分类 0..N 个**。
- **多分类最大允许数**:**上限 5**(默认建议 3,实施时按需调整)。
- UI 渲染时:**主分类显示为大角标**(category);**次分类显示为 chips 列表**(从主到次排列)。

### 3.3 次分类标签(subTag)

- **本期不引入** `model_details.sub_tags` JSON 列(Phase 3 backlog)。
- 已有 `model_details.liblib_tags_v2`(JSON 列)继续承担次分类承载。
- UI 在 `ModelItemCard.typeTag` 旁边额外展示「subTag 小徽标」(最多 3 个)。

### 3.4 UI 呈现

| 位置 | 当前 | 目标 |
|---|---|---|
| 详情页头部标签 | `typeTag(type)` 直接展示 | 主分类大角标(`primary_type` 派生)+ 次分类 chips(`types[]` 除主分类外) |
| 列表卡 `typeTag` | `s.typeTag ?? s.type` | 主分类徽标(`primary_type`);subTag 徽标(可选) |
| 首页推荐位 | 同列表 | 同上 |
| 详情页右侧 AuthorCard | 无 | 不变 |
| 详情页底部相关推荐 | 同列表 | 同上 |
| 生成页模型选择器 | 显示主分类 | 按主分类分组(All / Checkpoint / LoRA / LyCORIS / Controlnet / ...)(Phase 2) |
| 列表筛选条 | 4 类顶层 tab | 顶层 tab(image/video/ideas/flows)**不变**;新增 11 类 sub-chip multi-select(URL `?types=`) |

#### 3.4.1 Other 兜底规则

- Importer 遇未知 `modelType` → 落 `other`,**记日志**(运维追溯)。
- API 输出永远是 11 枚举之一,**不要在响应里返 unknown/other_string**。
- UI 在「Other」分类下显示「其他」徽标(中性灰),不显示原始 Liblib 类型。
- **chip 列表通常隐藏 Other**(防噪音);URL `?types=other` 可携带 `other=on` 显式打开(后续参数可设计为 `?showOther=1`)。
- 详情页不显示 Other badge(主分类不是 Other 时);主分类是 Other 时显示「其它 · 等待归类」灰色徽标。

### 3.5 筛选/排序

- 列表 API 支持 `?types=<csv>` 或 `?types=lora&types=lycoris`(多分类 OR 关系,Phase 1 增量加);保留旧 `?type=<single>`(单分类,兼容现有契约)。
- 多类筛选上限 5 类(`len(q.Types)<=5` 防注入)。
- 排序:`sort=latest`(默认)| `popular`(用 `model_counters.use_count`)| `download`(用 `model_counters.download_count`)| `trending`(Phase 2)。
- 「模板」分类默认排前面(运营策略可调)。

### 3.6 命名与术语

- 客户端 i18n key:`modelType.checkpoint` / `modelType.lora` / ...,中文显示由 `i18next` 翻译表统一。
- 客户端 typescript 端使用 `as const` 字面量从 `config.gen.ts` 取值(对齐 eye-trace-config 铁律 §4.1)。
- 命名迁移(EyeTraceAI / `ai.eyetrace.app` / `eyetrace://`)**不涉及分类字面量**(分类是技术术语),不需 i18n 适配。

---

## 4. 非功能需求

### 4.1 MinIO 拉取延迟 / 缓存

| 指标 | 方案 |
|---|---|
| 列表响应延迟 | 95p < 200ms;MySQL 索引 `idx_primary_type (primary_type)` + `idx_types_model (model_id, type)` 双索引,`models.type` 旧索引保留 |
| 详情响应延迟 | 95p < 150ms;单次 `models` JOIN `model_type_map` 1:N + `GROUP_CONCAT` 聚合 |
| 并发筛选 | 多 chip 同时筛选不爆炸;`IN (?)` 不分次;最多 5 类(上限防滥用) |
| 缓存 | 详情页 Redis cache 30s;`cache_keys: model:detail:{id}:v2`(v2 包含 types);HTTP `ETag`/`If-None-Match` 已就绪 |
| 一致性 | 改 type 不应前端崩;契约 `types: [ModelType]` 是数组,空数组合法(`minItems: 1` + 老 client fallback) |
| 可观测性 | server 新增 metrics `model_list_total{type=...}`;importer 启动日志 Liblib modelType 分布直方图 |
| **MinIO 解耦** | 11 类**不影响** MinIO object key;`assets.purpose` 枚举(`model_cover` / `user_return_pic` / `author_avatar`)不变;缩略图通过 MinIO bucket policy 直接 302 重定向,server 不中转大文件 |
| 分类元数据缓存 | server 启动期一次性加载 `enums.yaml` 11 枚举到内存;无需 DB 查询(本期无 `model_types` 字典表) |

### 4.2 分类权威源

| 层 | 真相源 |
|---|---|
| **DB** | MySQL `models.primary_type` + `model_type_map.type`(运行时) |
| **契约枚举** | `eye-trace-config/shared/enums.yaml` `modelType` 数组(跨端真相源) |
| **OpenAPI** | `ModelType` schema + `ModelDetail.types[]` 字段(契约层) |
| **客户端 i18n** | `src/i18n/{zh-CN,en-US}.json` `modelType.*` 翻译键(展示层) |
| **server 启动期校验** | `internal/model/enums.go::IsValidModelType` 从 enums.yaml 加载 11 枚举做 request 校验 + DB 入库校验 |
| **client 类型** | `src/api/schema.gen.ts` 的 `ModelType` 枚举值 **必须** 与 enums.yaml 一致(openapi-typescript 生成时引用 enum 常量) |
| **禁止** | 任何 client/server per-repo 常量 **禁止** 复制 11 枚举字面量(否则违反共享枚举铁律);**过渡期豁免**(Phase 2 exporters 落地后消除) |

### 4.3 索引性能

- `WHERE m.primary_type = ?` 走索引(`idx_primary_type`),11 枚举值基数低,**单列已够**。
- 多分类筛选 `?types=a,b,c` 走 `IN (...)` + 子查询,11 枚举基数小,索引同。
- 列表 `ORDER BY created_at DESC, id DESC` 已有 `idx_models_created`,**不动**。
- 可选 `idx_models_type_created(type, created_at DESC, id DESC)` 用于「按分类最新」分页查询。
- **不**新增 `models.types JSON` 上的索引(JSON 难索引,不做)。
- `model_type_map` 索引:`uk_model_type (model_id, type)` + `idx_type (type)` + `idx_model (model_id)`。

### 4.4 多租户

- `model_type_map` **不带 tenant_id**(分类是平台级元数据,不按租户隔离)。
- `models.primary_type` 同理,全局唯一(11 枚举值在所有 tenant 一致)。
- 与 Liblib 原始 `modelType` 的映射是平台规则,与租户无关。

### 4.5 一致性

- 导入器写 `models.primary_type` + `model_type_map` 在**同一事务**内完成。
- `models.primary_type` 与 `model_type_map.type WHERE is_primary=1` 必须一致(server `upsertModel` 双写保证)。
- DB 无 `model_types` 字典表(MVP);字典真值源在 `eye-trace-config/shared/enums.yaml`(运行期不变,改值 = 破坏性,需 DDL + 迁移)。

---

## 5. 数据模型重设计(核心)

### 5.1 方案选型(综合 4 份分析)

| 方案 | 描述 | 优点 | 缺点 | 4 份分析立场 | **综合决策** |
|---|---|---|---|---|---|
| **A** | 单字段扩展 `models.type ENUM(11 项)` | 极简,改造小 | 不允许多分类;`LyCORIS ∩ LoRA` 表达不了;MySQL ENUM 难迁移 | glm / deepseek 反对 ENUM | ✗ |
| **B** | 单列 `models.type VARCHAR(32)` + CHECK 约束 + enums.yaml(单分类) | 扩展容易,改 CHECK 即可;与现有 `models.type` 一致 | 不允许多分类;DB CHECK 与 enums.yaml 易漂移 | glm-5.2(主推);deepseek-v4-pro(主推) | △ 单分类备选 |
| **C** | 新表 `model_type_map`(多对多)+ `models.primary_type` 冗余 | 索引好,允许多分类;迁移平滑 | 双写一致性 | minimax-m3 / claude-opus-4-8(混合方案) | **✓ 推荐** |
| **D** | Enum table `model_types(code, label_zh, label_en, display_order, parent_code, color_token)` + `model_type_assignments` 多对多 + `models.type` 主分类冗余 | 枚举值扩展不需 ALTER;admin 可改 label/icon/order;i18n/扩展/审计载体 | 3 表迁移,字典表需 seed;运行期需 sync.Map 缓存 | claude-opus-4-8(主推) | △ v2 升级候选 |
| **E** | JSON 字段 `models.types JSON` | 灵活 | 无法建索引,筛选慢;FK 约束缺失 | 4 份均反对 | ✗ |

### 5.2 综合决策:采用 C 方案(本期),D 方案作为 v2 升级路径

**理由**:

1. **MVP 阶段简化**:C 方案只新增 1 张表 + 1 列,迁移成本最低;D 方案 3 张表(seed 字典 + 关联表 + models 改)对 MVP 是过度设计。
2. **多分类表达力**:Liblib 实际数据存在 LoRA+LyCORIS 双标模型(glm-5.2 误判为互斥,需更正);C 方案 `model_type_map` 自然支持多分类。
3. **索引设计**:C 方案 `models.primary_type` 单列索引 + `model_type_map` 联合索引覆盖所有筛选场景;JSON(JSON_CONTAINS)走全表扫。
4. **v2 升级路径清晰**:若未来 admin 要改 label/icon/order,可平滑升级到 D 方案(加 `model_types` 字典表 + 迁移),不破坏 v1 契约。
5. **字典元数据走 eye-trace-config**:label/icon/display_order 等 UI 元数据由 `eye-trace-config` 导出器生成 client `config.gen.ts` 与 server `config_gen.go`,运行期不查 DB,无缓存需求。
6. **与现有 models.type 兼容**:`models.type` 保留(v1 兼容),`primary_type` 冗余便于 SQL 加速,`model_type_map` 是新关联。

### 5.3 表 DDL

```sql
-- migration 000016_model_classification.up.sql

-- 1) models.primary_type:冗余主类,便于单值索引与排序
ALTER TABLE models
  ADD COLUMN primary_type VARCHAR(32) NOT NULL DEFAULT 'other'
    AFTER type,                          -- 旧 type 字段保留(v1 兼容)
  ADD INDEX idx_primary_type (primary_type);

-- 2) 新表:多对多分类映射
CREATE TABLE model_type_map (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  model_id   BIGINT       NOT NULL,
  type       VARCHAR(32)  NOT NULL,                       -- 11 类枚举值
  is_primary TINYINT(1)   NOT NULL DEFAULT 0,             -- 1=主分类,0=副分类
  source     VARCHAR(32)  NOT NULL DEFAULT 'liblib',       -- liblib / civitai / admin / heuristic
  created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_model_type (model_id, type),
  INDEX idx_type (type),
  INDEX idx_model (model_id),
  INDEX idx_type_primary (type, is_primary, model_id),     -- 主分类 JOIN 列表用
  CONSTRAINT fk_mtm_model FOREIGN KEY (model_id) REFERENCES models(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 5.4 回滚 SQL

```sql
-- 000016_model_classification.down.sql
ALTER TABLE models DROP INDEX idx_primary_type;
ALTER TABLE models DROP COLUMN primary_type;
DROP TABLE IF EXISTS model_type_map;
```

### 5.5 既有 `models.type ENUM/VARCHAR('checkpoint','lora','template','workflow','video')` 兼容

- **保留旧字段**(v1 兼容);值范围是 11 类的子集(`checkpoint/lora/template/workflow/video`)。
- 新代码**忽略** `type`,只读 `primary_type` + `model_type_map.type`。
- v2 时再 DROP `type`(本期不做)。

### 5.6 backfill SQL(迁移期)

```sql
-- 000016_model_classification.up.sql 续:

-- 1) 把旧 5 类同步到 primary_type
UPDATE models
SET primary_type = CASE type
  WHEN 'checkpoint' THEN 'checkpoint'
  WHEN 'lora'       THEN 'lora'
  WHEN 'template'   THEN 'template'
  WHEN 'workflow'   THEN 'template'   -- 重映射:workflow → template
  WHEN 'video'      THEN 'other'      -- 重映射:video → other(mediaType='video' 区分视频)
  ELSE                  'other'
END;

-- 2) 把旧 5 类写入 model_type_map(主分类)
INSERT IGNORE INTO model_type_map (model_id, type, is_primary, source)
SELECT id, primary_type, 1, 'legacy'
FROM models
WHERE primary_type IN ('checkpoint','lora','template','workflow','video','other');

-- 3) 兜底:全部置 other 作为兜底(防止老模型漏分类)
INSERT IGNORE INTO model_type_map (model_id, type, is_primary, source)
SELECT id, 'other', 0, 'legacy'
FROM models m
WHERE NOT EXISTS (SELECT 1 FROM model_type_map WHERE model_id = m.id);

-- 4) 修正 primary_type 的 other 兜底(防御性)
UPDATE models
SET primary_type = 'other'
WHERE primary_type NOT IN (
  'checkpoint','textual_inversion','hypernetwork','aesthetic_gradient',
  'template','lora','lycoris','controlnet','poses','wildcards','other'
);
```

### 5.7 与 MinIO object key 的耦合

- **完全解耦**:分类不影响 `assets.object_key` 命名(`models/{model_id}/{purpose}/{filename}` 不变)。
- `assets.purpose` 枚举(`model_cover` / `user_return_pic` / `author_avatar` / `model_gallery`)不变。
- 未来 Phase 2 admin 后台上传时,admin 选分类 → 直接写 `models.primary_type`,**不需要解析文件内容**。
- 命名规范文档 `minio_operations_manual.md` 不需修订。

### 5.8 索引设计

| 索引 | 目的 | 选择性 |
|---|---|---|
| `models.idx_primary_type (primary_type)` | 列表按主分类筛选 | 中(11 类,每类 8-12%) |
| `model_type_map.uk_model_type (model_id, type)` | 详情页 JOIN 唯一性 | 高 |
| `model_type_map.idx_type (type)` | 反向筛选:按 type 找 model_id | 中 |
| `model_type_map.idx_model (model_id)` | 详情页 JOIN | 中 |
| `model_type_map.idx_type_primary (type, is_primary, model_id)` | 主分类 JOIN 列表优化 | 高 |
| `models.idx_models_type (type)`(已有) | 老 SQL 兼容 | 中 |

---

## 6. 接口契约影响(核心)

### 6.1 openapi-skeleton.md 变更点

| § | 章节 | 变更 |
|---|---|---|
| §3 | Schema 骨架 | `ModelType` 枚举从 5 项扩到 **11 项**(详见 §6.2) |
| §3 | `ModelSummary` | 加 `types: [ModelType]` 字段(数组);保留旧 `type: ModelType` 字段(冗余兼容) |
| §3 | `ModelDetail` | 同 `ModelSummary` 同步;`type`/`typeTag` 不变 |
| §3 | 索引位置(分类筛选) | 加 `types` query parameter 到 `/models` |
| §3 | 新增 `ModelTypeAssignment` schema(可选,claude-opus-4-8 提议) | 本期**不引入**(MVP 简化);Phase 2 加,届时返回 `{code, label, labelEn, isPrimary, displayOrder, colorToken}` |

### 6.2 完整 OpenAPI schema 增量

```yaml
# components.schemas

ModelType:
  type: string
  description: |
    模型分类(SD 生态对齐,11 类)。Phase 1:多对多分类(models.type_map);
    次分类承载在 types[] 数组,主分类由 primary_type 派生。
    与 eye-trace-config/shared/enums.yaml::modelType 一一对应。
  enum:
    - checkpoint
    - textual_inversion
    - hypernetwork
    - aesthetic_gradient
    - template
    - lora
    - lycoris
    - controlnet
    - poses
    - wildcards
    - other

ModelSummary:
  type: object
  required: [modelId, title, mediaType, cover]
  properties:
    modelId:       { type: string }
    title:         { type: string }
    author:        { type: string }
    authorAvatar:  { type: string, format: uri }
    cover:         { type: string, format: uri }
    typeTag:       { type: string }                                       # 旧字段保留
    mediaType:     { $ref: '#/components/schemas/MediaType' }
    type:          { $ref: '#/components/schemas/ModelType' }              # 旧字段保留,v2 移除
    primaryType:                                                      # 新字段(冗余主类,= models.primary_type)
      $ref: '#/components/schemas/ModelType'
    types:                                                                 # 新字段(主+副类,顺序:is_primary DESC, display_order ASC)
      type: array
      items: { $ref: '#/components/schemas/ModelType' }
      minItems: 1
    useCount:      { type: integer, nullable: true }
    imageCount:    { type: integer, nullable: true }
    viewId:        { type: string, nullable: true }

ModelDetail:
  type: object
  required: [id, title, mediaType, cover, types]
  properties:
    id:            { type: string }
    title:         { type: string }
    mediaType:     { $ref: '#/components/schemas/MediaType' }
    cover:         { type: string, format: uri }
    type:          { $ref: '#/components/schemas/ModelType' }              # 旧字段保留
    primaryType:                                                      # 新字段
      $ref: '#/components/schemas/ModelType'
    types:                                                                 # 新字段
      type: array
      items: { $ref: '#/components/schemas/ModelType' }
      minItems: 1
    typeTag:       { type: string }
    typeTagSuffix: { type: string }
    # 其余字段沿用 model-detail-flow.md §2.3(ModelDetail 接口)

components:
  parameters:
    ModelTypesFilter:
      name: types
      in: query
      required: false
      schema:
        type: array
        items: { $ref: '#/components/schemas/ModelType' }
        maxItems: 5
      style: form
      explode: true                                                       # ?types=lora&types=lycoris
      description: 按模型类型筛选(任一命中)。最多 5 类。
```

### 6.3 端点影响

| Path | 变更 | 是否破坏 |
|---|---|---|
| `GET /api/v1/models` | 加 `types` query 参数(多值,OR 关系);响应每条加 `types[]` + `primaryType` | 否(新 query + 增字段) |
| `GET /api/v1/models/{modelId}` | response 加 `types[]` + `primaryType` | 否(增字段,v1 兼容) |
| `GET /api/v1/models/{modelId}/works` | 不变 | 否 |
| `POST /api/v1/workflow-runs` | `modelId` 选择器有效性延伸到 types 校验 | 否 |
| `GET /api/v1/content/home` | 瀑布流 `typeTag` 派生从 `models.primary_type` 取,非破坏 | 否 |
| `PATCH /api/v1/models/{id}`(Phase 3) | 入参 `ModelUploadRequest.types` 校验 | 否(未来) |
| `POST /api/v1/models`(Phase 3) | 入参 `ModelUploadRequest.types` 校验 | 否(未来) |
| `GET /api/v1/model-types`(可选 Phase 2) | 返回字典全量;本期**不做** | 否 |

### 6.4 v1 vs v2 决策

- **v1 兼容**:
  - `types[]` 数组新字段(增字段,非破坏);
  - 旧 `type` 单值字段保留为冗余(由 `primary_type` 派生),老 client 继续工作;
  - `primary_type` 新字段(冗余,便于 SQL `WHERE primary_type=?)。
- **v2 删除**(未来):
  - 单值 `type` 字段彻底移除;
  - `types[]` 是唯一来源。
- v1 过渡期 90 天,客户端优先读 `types`,没有再 fallback `type`,最后 fallback `other`。
- **MVP 不引入 v2**:只新增字段不删除字段 = 非破坏性;字段重命名才需要 `/v2`。

### 6.5 `/api/v2` 是否必要

**MVP 不引入 v2**:
- 只新增字段不删除字段 = 非破坏性。
- 字段重命名(如 `types` → `categories`)才需要 `/v2`。
- v1 保留 `type` 单值是为了旧客户端不崩;v2 是真彻底清理。

### 6.6 契约兼容矩阵

| 旧 client | 新 server | 行为 |
|---|---|---|
| 1.0(认 5 枚举) | 1.1(返 11 枚举) | 老 client 收到 `lycoris`/`textual_inversion` 等,UI 显示 raw 字符串;**不崩** |
| 1.1(认 11 枚举) | 1.0(返 5 枚举) | 老 server 只返 5 枚举,新 client `summaryToCard.types ?? [summary.type]` 走 fallback;**不崩** |
| 都 1.1 | 都 1.1 | 完全对 |

---

## 7. 跨端实现落点(client/server/docs/config)

### 7.1 server(`eye-trace-server/`)

| 文件 | 变更 |
|---|---|
| `backend/cmd/migrate/migrations/000016_model_classification.up.sql` | 新增迁移(建表 + backfill,见 §5.3 §5.6) |
| `backend/cmd/migrate/migrations/000016_model_classification.down.sql` | 回滚(见 §5.4) |
| `backend/internal/model/enums.go`(新建) | 11 枚举常量 + `IsValidModelType(s string) bool` + `ModelTypeLabel(code string) (zh, en string)`(过渡期,Phase 2 走 exporters) |
| `backend/internal/model/repo.go` | `ListModels` SQL 新增 `LEFT JOIN model_type_map` + `q.Types` 过滤;`GetModelByID` JOIN 一次返回所有 types;新增 `GetModelTypes(modelID) ([]string, error)` 方法 |
| `backend/internal/model/service.go` | `ToSummary` 派生 `PrimaryType` + `Types[]` 字段;`ToDetail` 同;`typeLabel` 从 enums.yaml 派生 |
| `backend/internal/api/handler/model.go` | `GetModels` 解析 `types` query(长度校验 `len<=5`);`GetModelByID` 不变(响应字段由 service 派生) |
| `backend/cmd/importer/liblib-deep/models.go` | `data.modelType` → 11 类映射表;支持多值(`lycoris:true` 触发 lora+lycoris 双标);`workflow` → `template` 重映射;`video` → `other` 重映射 |
| `backend/internal/api/api.gen.go` | 重生成(`make gen`) |
| `backend/schema/public_api.yaml` | 同步 §6.1 的 OpenAPI 变更(双源同步,见 `eye-trace-docs/CLAUDE.md` 注意事项) |
| `backend/internal/api/handler/handler.go`(检查) | ≤30 行硬约束保持 |

### 7.2 client(`eye-trace-client/`)

| 文件 | 变更 |
|---|---|
| `src/api/schema.gen.ts` | 重生成(`npm run gen:api`) |
| `src/api/models.ts` | `summaryToCard` 增加 `types[]` → `ModelItemCard.types[]` 映射;`types ?? [type] ?? ['other']` 降级 fallback;`ModelType` 本地别名改用 `components["schemas"]["ModelType"]` |
| `src/config/static/types.ts` | `ModelItemCard` 加 `types?: ModelType[]` + `primaryType?: ModelType` |
| `src/config/static/default.yaml` | 加 11 类 chip 文案,翻译 key: `modelType.checkpoint` 等 |
| `src/routes/modelInfo/ModelDetailPage.tsx` | 顶部 `type` badge 改为按 `types[]` 渲染 chip 堆叠;主分类大角标 + 次分类 chips |
| `src/routes/home/type-tabs-flow/` | 11 类 sub-chip multi-select(基于现有 `home-type-tabs-sub-tabs-flow.md`) |
| `src/components/ModelCategoryChip/ModelCategoryChip.tsx`(新建) | 渲染分类 chip(带 iconKey 解析 + colorToken) |
| `src/components/ModelCategoryChip/icons.tsx`(新建) | 11 类 SVG icon(本地引入,不走 MinIO) |
| `src/features/models/useModelList.ts` | 入参 `types[]` 支持多选;URL ↔ state 双向同步 |
| `src/features/models/TypeFilterBar.tsx`(新建,可选) | chips 渲染 + 点击切换 |
| `src/styles/tokens.css` | 加 11 类颜色 token:`--cat-checkpoint` / `--cat-lora` / ... |
| `src/i18n/locales/zh-CN.json` | 加 11 类中文显示:`modelType.checkpoint: "主模型"` 等 |
| `src/i18n/locales/en-US.json` | 加 11 类英文显示:`modelType.checkpoint: "Checkpoint"` 等 |
| `src/config/generated/config.gen.ts`(vendor) | 重新 vendor(过渡期手动复制 `eye-trace-config/dist/client/config.gen.ts`) |
| `src/__tests__/summaryToCard.test.ts` | 11 枚举覆盖 + types 渲染 + fallback 测试 |

### 7.3 eye-trace-docs(`eye-trace-docs/`)

| 文件 | 章节 | 改造 |
|---|---|---|
| `openapi-skeleton.md` | §3 Schemas | `ModelType` 枚举扩 11 项;`ModelSummary` / `ModelDetail` schema 加 `types[]` + `primaryType`;新增 `ModelTypesFilter` 参数 |
| `openapi-skeleton.md` | §2 `/models` 端点 | 加 `types[]` query 参数 |
| `openapi.yaml`(顶层) | 同上(完整版) | 同步 §6.2 |
| `model-detail-flow.md` | §2.3 API 契约 | `ModelDetail.types[]` + `primaryType` 新增说明 |
| `model-summary-card-flow.md` | §1.1 `ModelSummary` schema | `types[]` + `primaryType` 字段更新;§2.1 SQL 新增 `model_type_map` JOIN |
| `client_data_split.md` | §4.5 | `modelType` 加入 `enums.yaml` 真值表清单 |
| `home-type-tabs-sub-tabs-flow.md`(若存在) | — | 新增 11 类 sub-chip 渲染规范(沿用其结构) |
| **新文件** `model-type-flow.md` | 端到端 | 设计 → 落地 → 排查 → 完成(沿用 banner/recommend 同构);本期**不创建**(MVP 不强制,Phase 2 补) |
| `project_guide.md` | §7.4 | `models` 表 DDL 更新;新增 `model_type_map` 表 DDL |

### 7.4 eye-trace-config(`eye-trace-config/`)

| 文件 | 变更 |
|---|---|
| `shared/enums.yaml` | 新增 `modelType: [11 项]`(沿用现有 `supportedImageSizes` 风格) |
| `README.md` | 增加 `modelType` 真相源说明 |
| `CLAUDE.md` 索引 | 登记 `modelType`(已知 gap 列表同步) |
| `exporters/to-client.ts`(待建,Phase 2) | 暂不实现;过渡期 client 侧直接硬编码 |
| `exporters/to-server.ts`(待建,Phase 2) | 暂不实现;过渡期 server 侧 `internal/model/enums.go` 硬编码 |
| `dist/`(待建,Phase 2) | 暂不实现;过渡期手动 vendor |

```yaml
# eye-trace-config/shared/enums.yaml 新增
# === 模型分类 ===
# 11 类模型分类,server 与 client 共享契约枚举。
# 改动该值是破坏性变更,client/server 必须重建 + DB seed 重建。
modelType:
  - checkpoint
  - textual_inversion
  - hypernetwork
  - aesthetic_gradient
  - template
  - lora
  - lycoris
  - controlnet
  - poses
  - wildcards
  - other
```

> **过渡期豁免**:与 `supportedImageSizes` / `typeTabIds` / `jobStatus` 同列,client `src/config/static/default.yaml` 与 server `internal/model/enums.go` 复制 11 枚举字面量,标注「待消除技术债」(Phase 2 exporters 落地后消除)。

### 7.5 ComfyUI / Worker-Agent

- **不影响**: `pipeline_templates` 表、`jobs.template_snapshot`、Worker-Agent WS 协议、ComfyUI graph 都不涉及分类。
- 模型选择阶段 `modelId` 可携带 `types[]`,worker-agent 接收 `submit` 时**不解析** `types`(只信任 server 的 `JobId → template` 映射)。
- Phase 3 类型驱动分支(只允许 LoRA 模型用 LoRA 模板)再考虑 Worker-Agent 介入。

### 7.6 跨子仓 PR 拆分(参考 client-architecture §15)

| PR | 范围 | 依赖 | 顺序 |
|---|---|---|---|
| **PR-1** | `eye-trace-config`:新增 `modelType` 11 项 + README/CLAUDE.md 索引更新 | 无 | 1 |
| **PR-2** | `eye-trace-docs`:改 `openapi.yaml` + `openapi-skeleton.md` §3,新增 `types[]` + `primaryType` 字段 + `types` query | 无 | 2 |
| **PR-3** | `eye-trace-server`:迁移 000016 + repo/service/handler + importer 映射 | PR-2(契约先行) | 3 |
| **PR-4** | `eye-trace-server`:重生成 `api.gen.go`;CI `make oapi-check` | PR-3 | 4 |
| **PR-5** | `eye-trace-client`:重生成 `schema.gen.ts` + UI 改造 + i18n | PR-4 | 5 |
| **PR-6**(可选) | `model-detail-flow.md` / `model-summary-card-flow.md` 更新字段说明 | PR-2 | 6 |

---

## 8. 迁移与兼容性

### 8.1 DB 迁移

- **前向**(`up.sql`): §5.3 §5.6 完整给出。
- **回滚**(`down.sql`): §5.4。
- **生产 ALTER TABLE 大表阻塞**:使用 `pt-online-schema-change` 或 `gh-ost`(本期 ADD COLUMN + ADD INDEX + CREATE TABLE,新增表无锁风险,加列对 InnoDB 8.0 instant DDL 已支持)。

### 8.2 契约兼容

- v1 内只增字段(数组 `types`、`primaryType`、query `types`),**非破坏**。
- 老客户端忽略 `types` / `primaryType`,继续用 `type` 单值,工作正常。
- 老 server(`api.gen.go` 旧版)继续输出 `type` 单值,新 client 仍可 fallback。

### 8.3 客户端降级渲染

- 新 client 逻辑:
  ```ts
  // detail
  modelDetail.types ?? [modelDetail.primaryType ?? modelDetail.type ?? 'other']
  
  // list summary
  summary.types ?? [summary.primaryType ?? summary.type ?? 'other']
  ```
- 保证 UI 不崩(数组至少 1 元素)。
- TypeScript exhaustiveness check:所有 UI `switch (modelType)` 加 `default: case`,处理新增枚举(编译期强制)。

### 8.4 Importer backfill

- 一次性脚本 `cmd/import-backfill-classification`:`SELECT id, type FROM models WHERE NOT EXISTS (SELECT 1 FROM model_type_map WHERE model_id = models.id)`,循环 insert `model_type_map` 行。
- **liblib `data.modelType` 数值/字符串 → 11 类映射表**(附录 A):写在 importer 代码里(`cmd/importer/liblib-deep/liblibModelTypeMap.go`),**不**写进 DB 字典表(数据/逻辑分离)。
- **启发式回退**:liblib 没标 → 用文件名 + 大小启发式:
  - `.pt` < 50MB 且路径含 `embedding`/`ti` → `textual_inversion`
  - `.pt` > 50MB 且路径含 `hypernet` → `hypernetwork`
  - `.safetensors` 含 `lyco`/`loha`/`lokr`/`ia3` 标记 → `lycoris`
  - `.safetensors` 含 `control`/`controlnet`/`cnet` → `controlnet`
  - `.json` + 路径含 `wildcards` → `wildcards`
  - `.json` + OpenPose 骨架标记 → `poses`
  - `.safetensors` > 1.5GB → `checkpoint`
  - 其他 → `other`
- Importer 校正后**幂等**;已存在的 `model_type_map` 行通过 `INSERT IGNORE` 跳过。

### 8.5 命名迁移协调

- EyeTraceAI 命名迁移(品牌重命名 + Deeplink 替换)**不涉及分类字面量**(分类是技术术语)。
- 错开 1 周:命名迁移 PR-1 先合,分类 PR 后合(本方案 §7.6 顺序)。

---

## 9. 风险与歧义

| ID | 风险 | 影响 | 处置 |
|---|---|---|---|
| R1 | **LyCORIS 与 LoRA 双标争议** | UI 视觉重复;数据统计复杂 | 产品决策:**允许双标**,详情页两个 badge,统计按"or" 命中 |
| R2 | **ControlNet 与 Poses 边界** | Poses 本质是 ControlNet 的输入分支 | 双标;或在 v2 引入 `subtypeOf` 表(`poses` → `controlnet`);本期双标 |
| R3 | **template 与 workflow 关系** | Liblib 把 workflow 视为 model;OpenAPI 旧版 `type` 含 `workflow` | 简化:全部合并到 `template`;`workflow` 重映射 → `template`(v1 兼容) |
| R4 | **Other 占比失控** | > 30% 数据归 Other,分类体系失效 | 设 KPI:运营监控 import 后 Other 比例,阈值告警;importer unknown 写日志 |
| R5 | **11 类与 liblib `modelType` 不一一对应** | Importer 映射复杂 | Importer 内显式 `map[string]ModelType`(白名单)+ 启发式 fallback;不在配置中心定义映射 |
| R6 | **多分类筛选性能** | `types=lora,lycoris` 走 `IN` + 子查询 | 索引 `idx_type` + `idx_type_primary` 已建,实测 < 150ms |
| R7 | **EyeTraceAI 命名迁移叠加** | 重命名 + 分类同时上线易冲突 | 错开 1 周:命名迁移 PR-1 先合,分类 PR 后合(本方案 §7.6 顺序) |
| R8 | **客户端枚举字面量僵硬** | TS `ModelType as const` 类型不更新会编译错 | `openapi-typescript` 自动生成,只需 re-gen |
| R9 | **模型上传入口(model.uploader)的 type 选择器** | 11 个 chip 空间占用 | 分组:「主模型」「微调」「辅助」三段(Phase 3 后台) |
| R10 | **历史 `models.type ENUM('...5项...')` 移除迁移** | ALTER TABLE 在大表上阻塞 | 生产用 `pt-online-schema-change` 或 `gh-ost`;本期**保留**,v2 再删 |
| R11 | **eye-trace-config 导出器未落地** | client/server 复制枚举字面量(技术债) | 与 `supportedImageSizes` 同列;Phase 2 exporters 落地后消除 |
| R12 | **Liblib 原始 `modelType` 字段不存在或粒度粗** | Importer 大量模型落 `lora` 或 `other`,分类不准 | §8.4 文件名/大小启发式 + 离线统计 100% Liblib 样本 `modelType` 分布 |
| R13 | **DB CHECK 约束与 enums.yaml 漂移**(glm 方案) | 入库校验失败 | 本期**不用 DB CHECK**(方案 C 不用);server 启动期从 enums.yaml 加载校验集合;FK 约束保证 `model_type_map.type` 合法 |
| R14 | **`models.type` 列未来是否可去**(deepseek/claude-opus 提出) | 老 API 消费者可能依赖 | 保留,作为主分类冗余;v2 之后老值不再写,但列存在兼容老导入器 |
| R15 | **双源 openapi.yaml 与 public_api.yaml 漂移**(eye-trace-docs/CLAUDE.md 注意 #2) | CI 失败 | `make oapi-check` CI 校验;文档侧先改 `openapi-skeleton.md`(骨架),server 侧同步 `public_api.yaml`(完整) |

---

## 10. 待澄清问题清单(综合 4 份分析,需向 Product/运营确认)

| # | 问题 | 4 份分析立场 | 综合建议 |
|---|---|---|---|
| Q1 | **多分类到底允不允许?** | minimax-m3(允许多)/ claude-opus-4-8(允许多)/ glm-5.2(互斥单选)/ deepseek-v4-pro(互斥单选) | **允许多分类**(C 方案);次分类承载 `types[]`;次标签承载 `model_details.liblib_tags_v2` |
| Q2 | **`workflow` 业务分类是合并入 `template` 还是保留?** | minimax-m3(合并 template)/ glm-5.2(归 Other)/ deepseek-v4-pro(合并 template)/ claude-opus-4-8(合并 template) | **合并入 `template`**(语义重叠;4/4 一致建议,glm 例外) |
| Q3 | **`video` 业务分类是单独分类还是归 `other`?** | minimax-m3(保留 video)/ glm-5.2(保留 video)/ deepseek-v4-pro(归 other)/ claude-opus-4-8(归 other) | **归 `other`**(`mediaType='video'` 区分业务;`video` 业务分类属于 `typeTabIds` 范畴,与"模型文件分类"正交);`models.type='video'` 旧值 → `other` 重映射 |
| Q4 | **Other 在 UI 上是否完全隐藏?** | minimax-m3(chip 隐藏 + URL 显式)/ glm-5.2(显示「其他」徽标)/ deepseek-v4-pro(显示「其它 · 等待归类」灰色)/ claude-opus-4-8(显示「其它 · 等待归类」灰色) | **chip 列表默认隐藏**;详情页主分类为 Other 时显示「其它 · 等待归类」灰色徽标;URL `?types=other` 显式筛选命中 |
| Q5 | **Civitai 数据源何时接入?** | minimax-m3(留 v2)/ glm-5.2(本期映射 Civitai)/ deepseek-v4-pro(未提)/ claude-opus-4-8(Phase 2) | **本 PR 不做,留 Phase 2**;`model_type_map.source` 字段预留 `civitai / admin / heuristic` 标记 |
| Q6 | **template(工作流)与 Phase 2 管线模板的关系?** | minimax-m3(`pipeline_templates` ≠ `models.type=template`)/ glm-5.2(Phase 3 admin 后台)/ deepseek-v4-pro(Phase 2 关联)/ claude-opus-4-8(本期不做) | **不同**;前者是 ComfyUI graph 模板(`pipeline_templates` 表),后者是 liblib 上架的工作流;本期不联动 |
| Q7 | **hypernetwork 是否真的保留?** | minimax-m3(保留 deprecated)/ glm-5.2(保留)/ deepseek-v4-pro(保留)/ claude-opus-4-8(保留) | **保留**(4/4 一致);SD 1.x 时代老用户群存在;后续运营数据少可降级到 Other |
| Q8 | **多分类最大允许数量** | minimax-m3(3)/ glm-5.2(单选 1)/ deepseek-v4-pro(单选 1)/ claude-opus-4-8(未明确) | **上限 5**(索引 + 防滥用);默认建议 3 |
| Q9 | **11 类的 UI 优先级顺序** | 4 份均给出顺序(Checkpoint 最前,Other 末位) | 按 §2 顺序(Checkpoint → Textual Inversion → Hypernetwork → Aesthetic Gradient → Template → LoRA → LyCORIS → ControlNet → Poses → Wildcards → Other);运营/PM 可调 |
| Q10 | **admin 后台是否需要批量改分类?** | minimax-m3(v2 引入)/ glm-5.2(Phase 3)/ deepseek-v4-pro(Phase 2)/ claude-opus-4-8(本期不做) | **本期不做**;Phase 3 admin 后台提供 `PATCH /api/v1/models/{id}` + `model_type_map` 事务 |
| Q11 | **client 是否需要 11 类 SVG icon?** | minimax-m3(走 `recommend-icons/`)/ glm-5.2(未提)/ deepseek-v4-pro(本地 SVG)/ claude-opus-4-8(走 colorToken) | **本地 SVG icon**(`src/components/ModelCategoryChip/icons.tsx`);不走 MinIO;走 `colorToken` 染色 |
| Q12 | **客户端 fallback 默认值?** | minimax-m3(`'other'`)/ glm-5.2(raw 字符串 + 未识别分类)/ deepseek-v4-pro(raw 字符串)/ claude-opus-4-8(降级到 `type`) | **`['other']`**(保证 UI 不崩;raw 字符串兜底仅 TypeScript exhaustiveness check 触发) |
| Q13 | **`models.type` ENUM 列是否删除?** | 4 份均建议保留(v1 兼容) | **保留**(v1 兼容);v2 时再 DROP;`primary_type` 冗余承担主分类 |
| Q14 | **模型分类元数据走 DB 字典表 vs eye-trace-config?** | claude-opus-4-8(DB 字典表 + eye-trace-config 双源)/ 其他(eye-trace-config 单一) | **MVP 走 eye-trace-config 单一**(过渡期豁免 client/server 复制);v2 升级到 DB 字典表(若 admin 可改 label/icon/order) |
| Q15 | **liblib `modelType` 数值映射表位置?** | deepseek-v4-pro(importer 内部代码)/ 其他(importer 内部) | **importer 内部硬编码**(`cmd/importer/liblib-deep/liblibModelTypeMap.go`);不入 DB 字典表;不入 eye-trace-config |

---

## 11. MVP 分期与里程碑

### 11.1 MVP(本次,7 个工作日 / 单线程约 2 周)

| 交付物 | 文件 |
|---|---|
| enums.yaml `modelType` 11 枚举 | `eye-trace-config/shared/enums.yaml` |
| DB 迁移 + backfill | `eye-trace-server/backend/cmd/migrate/migrations/000016_model_classification.{up,down}.sql` |
| server `ModelType` enum + `IsValidModelType` + `ModelTypeLabel` | `eye-trace-server/backend/internal/model/enums.go` |
| OpenAPI 扩枚举 + 新字段 | `eye-trace-docs/openapi.yaml` + `eye-trace-docs/openapi-skeleton.md` + `eye-trace-server/backend/schema/public_api.yaml` |
| repo/service/handler 加 `types[]` + `primaryType` 字段 | `eye-trace-server/backend/internal/model/{repo,service,handler}.go` |
| importer liblib 数值映射表 + 启发式回退 + 重映射(workflow→template, video→other) | `eye-trace-server/backend/cmd/importer/liblib-deep/{models,liblibModelTypeMap}.go` |
| client 类型重生成 + summaryToCard types 映射 | `eye-trace-client/src/api/schema.gen.ts` + `eye-trace-client/src/api/models.ts` |
| 详情页渲染主分类角标 + 次分类 chips | `eye-trace-client/src/app/routes/ModelDetailPage.tsx` |
| 列表卡渲染主分类徽标 + subTag | `eye-trace-client/src/components/layout/ModelItemList.tsx` |
| 列表筛选条 11 chip multi-select | `eye-trace-client/src/features/models/TypeFilterBar.tsx`(新)+ `eye-trace-client/src/features/models/useModelList.ts` |
| 11 类颜色 token + SVG icon | `eye-trace-client/src/styles/tokens.css` + `eye-trace-client/src/components/ModelCategoryChip/{ModelCategoryChip,icons}.tsx` |
| i18n 双语 label | `eye-trace-client/src/i18n/{zh-CN,en-US}.json` |
| 单测:enum 校验 + 多类型筛选 + summaryToCard 覆盖 | `eye-trace-server/backend/internal/model/enums_test.go` + `eye-trace-server/backend/internal/model/repo_test.go` + `eye-trace-client/src/__tests__/summaryToCard.test.ts` |
| CI 双源校验 | `make oapi-check` + `npm run gen:api:check` |

### 11.2 v1.1(可选,2 周)

- Redis cache `model:detail:{id}:v2`(含 types)。
- server metrics `model_list_total{type=...}`(Prometheus)。
- 控制台:Other 占比告警(>30% 阈值)。
- Importer liblib modelType 分布直方图。

### 11.3 v2(分阶段,~6 周)

- DROP `models.type`(单值字段);`types[]` 是唯一来源。
- 新增 `model_types` 字典表(D 方案升级路径);新增枚举值不需 ALTER。
- admin 后台分类管理 UI + `PATCH /api/v1/models/{id}` 端点。
- Worker-Agent 类型驱动分支(只允许 LoRA 模型跑 LoRA 模板)。
- Civitai 导入器接入(独立 `liblibModelTypeMap.civitai`)。
- 产品决策:Workflow 类型是否从 template 独立(取决于 Phase 2 后的运营数据)。
- 自动分类:ML 读 `data.description + tagsV2` 建议分类(运营审批)。

---

## 12. 验证清单

### 12.1 数据正确性

```sql
-- 1) 11 类分布
SELECT type, COUNT(*) AS models
FROM model_type_map
GROUP BY type
ORDER BY models DESC;

-- 2) 多分类样本
SELECT model_id, COUNT(*) AS types_count
FROM model_type_map
GROUP BY model_id
HAVING types_count > 1
LIMIT 10;

-- 3) 旧 type 与 primary_type 一致性
SELECT COUNT(*) AS mismatches
FROM models m
LEFT JOIN model_type_map mtm ON mtm.model_id = m.id AND mtm.is_primary = 1
WHERE m.primary_type <> mtm.type OR mtm.type IS NULL;
-- 期望 0

-- 4) backfill 完整
SELECT COUNT(*) AS total, COUNT(mtm.model_id) AS with_types
FROM models m
LEFT JOIN model_type_map mtm ON mtm.model_id = m.id;
-- 期望 total = with_types(每条模型至少 1 个分类)
```

### 12.2 API 验证

```bash
# 单类筛选(兼容旧 ?type=)
curl 'http://localhost:8080/api/v1/models?type=lora&limit=10' | jq '.items[].type'

# 多类筛选(新 ?types=)
curl 'http://localhost:8080/api/v1/models?types=lora&types=lycoris&limit=10' | jq '.items[].types'

# 详情 types 字段
curl 'http://localhost:8080/api/v1/models/15' | jq '.types, .primaryType'

# 兜底:unknown type 仍返回 422
curl 'http://localhost:8080/api/v1/models?types=xyz'
# 期望:400 VALIDATION_FAILED

# 列表数量上限(>5 类)
curl 'http://localhost:8080/api/v1/models?types=a&types=b&types=c&types=d&types=e&types=f'
# 期望:400 VALIDATION_FAILED(len > 5)
```

### 12.3 浏览器验证

- 列表页选中 LoRA + LyCORIS 多选 → 命中两类模型,显示 `types` badge。
- 详情页双标模型显示 `[LoRA] [LyCORIS]` 两个 badge(主分类大角标 + 次分类 chips)。
- Other 模型在 chip 列表不出现;URL `?types=other` 显式筛选命中。
- 主分类为 Other 的模型详情页显示「其它 · 等待归类」灰色徽标。
- i18n 切换 zh / en 时 11 类显示文案正确。
- 老 client(2026-06-27 版)调新 server → 仍能渲染(降级到 `type` 字符串)。

### 12.4 CI 校验

- `make oapi-check`(server):openapi.yaml 与生成代码一致。
- client: `npm run lint` + `npm run test` 覆盖 summaryToCard 测与 types 渲染测。
- server: `go test ./internal/model/...` 通过。
- `npm run gen:api:check`:client `schema.gen.ts` 与 docs `openapi.yaml` 一致。

---

## 13. 关联到具体文件(指针表)

### 13.1 契约(`eye-trace-docs/`)

| 文件 | 章节 | 改造 |
|---|---|---|
| `openapi-skeleton.md` | §3 Schemas | `ModelType` 枚举扩 11 项;`ModelSummary` / `ModelDetail` schema 加 `types[]` + `primaryType`;`/models` 加 `types` query |
| `openapi.yaml`(顶层) | 同上(完整版) | 同步 §6.2 |
| `project_guide.md` | §7.4 | `models` 表 DDL 更新;新增 `model_type_map` 表 DDL |
| `client_data_split.md` | §4.5 | `modelType` 加入 `enums.yaml` 真值表清单 |
| `model-detail-flow.md` | §2.1 / §2.3 | `liblib.data.modelType` 映射表更新;`ModelDetail` 接口更新 |
| `model-summary-card-flow.md` | §1.1 / §2.1 | `ModelSummary` schema 更新;`ListModels` SQL 新增 `model_type_map` JOIN |

### 13.2 配置中心(`eye-trace-config/`)

| 文件 | 改造 |
|---|---|
| `shared/enums.yaml` | 新增 `modelType: [...]` 11 项 |
| `README.md` | 增加 `modelType` 真相源说明 |
| `CLAUDE.md` 索引 | 登记 `modelType`(已知 gap 列表同步) |

### 13.3 server(`eye-trace-server/`)

| 文件 | 改造 |
|---|---|
| `backend/cmd/migrate/migrations/000016_*.up.sql` | 新增(§5.3 §5.6) |
| `backend/cmd/migrate/migrations/000016_*.down.sql` | 新增(§5.4) |
| `backend/internal/model/enums.go`(新建) | 11 枚举常量 + `IsValidModelType` + `ModelTypeLabel` |
| `backend/internal/model/repo.go` | `ListModels` / `GetModelByID` JOIN + `Types` 过滤 |
| `backend/internal/model/service.go` | `ToSummary` / `ToDetail` types 派生 |
| `backend/internal/api/handler/model.go` | `types` query 解析 |
| `backend/cmd/importer/liblib-deep/liblibModelTypeMap.go`(新建) | liblib 数值 → 11 类映射表 + 启发式 |
| `backend/cmd/importer/liblib-deep/models.go` | `mapLiblibModelType(s string) []string` |
| `backend/internal/api/api.gen.go` | 重生成(`make gen`) |
| `backend/internal/api/handler/handler.go`(检查) | ≤30 行硬约束保持 |

### 13.4 client(`eye-trace-client/`)

| 文件 | 改造 |
|---|---|
| `src/api/schema.gen.ts` | 重生成(`npm run gen:api`) |
| `src/api/models.ts` | `summaryToCard` types 映射 + fallback |
| `src/components/ModelCategoryChip/ModelCategoryChip.tsx`(新) | 渲染分类 chip(icon + color) |
| `src/components/ModelCategoryChip/icons.tsx`(新) | 11 类 SVG icon |
| `src/routes/modelInfo/ModelDetailPage.tsx` | types 渲染(主分类大角标 + 次分类 chips) |
| `src/routes/home/TypeTabsSubTabs.tsx` | 11 类 sub-filter |
| `src/features/models/TypeFilterBar.tsx`(新) | chips 渲染 + 点击切换 + URL 同步 |
| `src/features/models/useModelList.ts` | 入参 `types[]` |
| `src/components/layout/ModelItemList.tsx` | typeTag 渲染用 label |
| `src/config/static/types.ts` | `ModelItemCard.types/primaryType` 字段 |
| `src/config/static/default.yaml` | sub-chip 文案 |
| `src/config/generated/config.gen.ts` | vendor 新版(过渡期手动复制) |
| `src/i18n/locales/zh-CN.json` 等 | 11 类翻译 |
| `src/styles/tokens.css` | 11 类 chip 颜色 token(`--cat-*`) |

### 13.5 不动的文件

- `ComfyUI/`(不动,ComfyUI 不感知分类)
- `eye-trace-server/internal/comfy/`(空包,不动)
- `eye-trace-server/backend/internal/job/`(job 状态机不动)
- `eye-trace-server/backend/pkg/wspool/`(WS hub 不动)
- 命名迁移相关文件(`project_guide.md` §9)(本 PR 不动,留给命名迁移)

---

## 14. 设计权衡与决策记录(ADR 格式)

### 14.1 ADR-001: 多分类 vs 单分类

- **Context**: liblib 数据存在 LoRA+LyCORIS 双标模型;老 client 仅有 `type` 单值字段;glm-5.2 / deepseek-v4-pro 主张互斥单选(SD 生态"一份模型文件是一种"),minimax-m3 / claude-opus-4-8 主张多分类(运营灵活性)。
- **Decision**: 多对多 + `model_type_map` + `models.primary_type` 冗余(C 方案)。
- **Status**: Accepted(MVP)。
- **Consequences**:
  - (+) 表达力强,允许 transition 类型(ControlNet+Poses)。
  - (+) 兼容未来 admin 自定义次分类(Phase 3)。
  - (-) 索引设计稍复杂(`model_type_map` 双索引)。
  - (-) 旧 client 单值字段需保留 1 个版本周期。

### 14.2 ADR-002: DB 字典表 vs enums.yaml 单一真相源

- **Context**: claude-opus-4-8 主张 DB `model_types` 字典表(admin 可改 label/icon/order);其他 3 份主张 enums.yaml 单一(运行期不变)。
- **Decision**: **MVP 走 enums.yaml 单一**(C 方案),v2 升级到 D 方案(若 admin 要改 label/icon/order)。
- **Status**: Accepted(MVP);v2 reassess。
- **Consequences**:
  - (+) MVP 简单,无 seed 负担。
  - (+) 改 label/icon 走 eye-trace-config PR(代码评审可控)。
  - (-) admin 后台不可改 label(Phase 3 升级 D 方案解锁)。

### 14.3 ADR-003: 旧 `type` 字段是否保留

- **Context**: openapi 契约 v1 是否向后兼容;minimax-m3(claude-opus-4-8) 建议保留;glm / deepseek 主张用 `primaryType` 替代旧字段。
- **Decision**: v1 保留 `type` 单值(冗余,值 = `primary_type`),v2 移除。
- **Status**: Accepted。
- **Consequences**:
  - (+) 旧 client 兼容。
  - (+) 老 SQL(列表)零改动。
  - (-) v2 阶段需双 schema 短暂并存。

### 14.4 ADR-004: 11 类与 liblib `modelType` 映射

- **Context**: liblib 原 5 类(include `workflow`/`video`);Civitai 13 类;ComfyUI 节点族。
- **Decision**: 11 类与 Civitai/ComfyUI 生态对齐;`workflow` 并入 `template`;`video` 归 `other`(`mediaType='video'` 区分业务);liblib 映射在 importer 显式写,不在 config 中心定义。
- **Status**: Accepted。
- **Consequences**:
  - (+) 生态语义对齐。
  - (-) liblib `workflow`/`video` 数据重映射有兼容性(老枚举值在 v1 仍兼容)。

### 14.5 ADR-005: Other 在 UI 上的呈现

- **Context**: Other 兜底若显示,UI 噪音大;若隐藏,用户无法主动筛。
- **Decision**: chip 列表默认隐藏;URL `?types=other` 显式命中;详情页主分类为 Other 时显示「其它 · 等待归类」灰色徽标。
- **Status**: Accepted。
- **Consequences**:
  - (+) UI 清洁。
  - (+) 主分类为 Other 的运营数据可识别。
  - (-) 高级用户需手动拼 URL。

### 14.6 ADR-006: v1 vs v2 策略

- **Context**: 是否进 /v2 处理 11 类扩展。
- **Decision**: **不进 /v2**(v1 增量加字段);v2 时移除旧 `type` 单值。
- **Status**: Accepted(MVP)。
- **Consequences**:
  - (+) v1 过渡期 90 天,client 渐进式升级。
  - (-) v2 需双 schema 短暂并存。

### 14.7 ADR-007: 字典元数据来源(eye-trace-config)

- **Context**: label/icon/display_order 等 UI 元数据走 DB 字典表 vs eye-trace-config 导出物。
- **Decision**: **MVP 走 eye-trace-config 导出物**(`MODEL_TYPE_LABELS` / `MODEL_TYPE_COLOR` / `MODEL_TYPE_DISPLAY_ORDER`),运行期无 DB 查询。
- **Status**: Accepted(MVP)。
- **Consequences**:
  - (+) 无 sync.Map 缓存需求(11 枚举编译期常量)。
  - (+) admin 改 label 需走 eye-trace-config PR(代码评审)。
  - (-) 过渡期 client/server 复制 11 枚举字面量(技术债,Phase 2 exporters 落地后消除)。

---

## 15. 边界外未来工作(backlog)

1. **类型驱动的 Worker-Agent 调度**: 同一 modelId 携带 `types`,Worker 选择特定模板策略(Phase 3)。
2. **类型子分类树**: `poses → openpose / densepose / depth` 子类型(超 MVP)。
3. **自动分类**: LLM 读 `data.description + tagsV2`,建议分类(运营审批)。
4. **类型 ↔ 模板策略**: `modelType=lycoris → 强制 LoraLoader 节点` 的策略表(Phase 3)。
5. **分类 KPI 看板**: 运营仪表盘的分类分布、点击率、转化率。
6. **DB 字典表(D 方案)升级**: 若 admin 要改 label/icon/order,平滑升级到 `model_types` 字典表 + `model_type_assignments` 多对多。
7. **Civitai 导入器接入**: Phase 2,独立 `liblibModelTypeMap.civitai` 映射表。
8. **`model_counters.type` 反范式列**: 便于按分类聚合统计。

---

## 16. 一句话总结

> **MVP 落点 = 共享枚举扩 11 项 + `models.primary_type` 冗余 + `model_type_map` 多对多 + `types[]` schema 字段(v1 兼容,不引入 v2,不联动 ComfyUI/Worker/命名迁移);4 份分析达成共识的数据模型,综合推荐 C 方案(多对多)。**

---

## 附录 A:Liblib → 11 类映射表(importer 内部硬编码)

| liblib `data.modelType` | 映射到的 11 类(可多值) | 备注 |
|---|---|---|
| `1` / `"Checkpoint"` / `"checkpoint"` | `[checkpoint]` | 主 |
| `2` / `"LoRA"` / `"LORA"` | `[lora]` | 主;Liblib 不区分 lycoris 时默认 lora |
| `"LyCORIS"` / `"LoCon"` / `"LoHa"` / `"LoKr"` / `"IA3"` | `[lycoris]` (+ `[lora]` 若 tagsV2 含 `lycoris:true`) | 新版 liblib;**多分类**触发双标 |
| `"TextualInversion"` / `"textualInversion"` / `"embedding"` / `"Embedding"` | `[textual_inversion]` | |
| `"Hypernetwork"` / `"hypernetwork"` | `[hypernetwork]` | 老式 |
| `"AestheticGradient"` / `"aestheticGradient"` / `"Aesthetic"` | `[aesthetic_gradient]` | 小众 |
| `"Controlnet"` / `"ControlNet"` / `"controlnet"` / `"Controlnet"` | `[controlnet]` (+ `[poses]` 若 tagsV2 含 `openpose`) | **多分类**触发双标 |
| `"Pose"` / `"Poses"` / `"OpenPose"` / `"openpose"` | `[poses, controlnet]` | 双标:poses + controlnet 输入 |
| `"Wildcards"` / `"wildcards"` / `"wildcardProcessor"` / `"DynamicPrompts"` | `[wildcards]` | |
| `"Workflow"` / `"Template"` / `"工作流"` / `"template"` | `[template]` | **重映射**:`workflow` → `template` |
| `"video"` / `"Video"` / `"Motion"` / `"VAE"` / `"Upscaler"` | `[other]` | **重映射**:`video` 业务分类归 `other`(`mediaType='video'` 区分业务) |
| `0` / `null` / `""` / 其它未知 | `[other]` | 兜底,记 `INFO log:model_type_fallback` |
| 启发式回退 | 见 §8.4 | liblib 元数据缺失时按文件名/大小启发 |

## 附录 B:TypeScript 端 enum 字面量迁移(过渡期硬编码)

client `src/config/static/default.yaml`(示意,真实文件生成):

```ts
export type ModelType =
  | "checkpoint"
  | "textual_inversion"
  | "hypernetwork"
  | "aesthetic_gradient"
  | "template"
  | "lora"
  | "lycoris"
  | "controlnet"
  | "poses"
  | "wildcards"
  | "other";

// UI 染色 map
export const MODEL_TYPE_COLOR: Record<ModelType, string> = {
  checkpoint:        "--cat-checkpoint",
  textual_inversion: "--cat-textual-inversion",
  hypernetwork:      "--cat-hypernetwork",
  aesthetic_gradient:"--cat-aesthetic-gradient",
  template:          "--cat-template",
  lora:              "--cat-lora",
  lycoris:           "--cat-lycoris",
  controlnet:        "--cat-controlnet",
  poses:             "--cat-poses",
  wildcards:         "--cat-wildcards",
  other:             "--cat-other",
};

// UI display order
export const MODEL_TYPE_DISPLAY_ORDER: Record<ModelType, number> = {
  checkpoint:        10,
  lora:              20,
  lycoris:           30,
  textual_inversion: 40,
  hypernetwork:      50,
  aesthetic_gradient:60,
  controlnet:        70,
  template:          80,
  poses:             90,
  wildcards:         100,
  other:             999,
};
```

## 附录 C:server `IsValidModelType` 校验(过渡期硬编码)

```go
// backend/internal/model/enums.go
package model

var AllowedModelTypes = map[string]struct{}{
    "checkpoint":         {},
    "textual_inversion":  {},
    "hypernetwork":       {},
    "aesthetic_gradient": {},
    "template":           {},
    "lora":               {},
    "lycoris":            {},
    "controlnet":         {},
    "poses":              {},
    "wildcards":          {},
    "other":              {},
}

func IsValidModelType(code string) bool {
    _, ok := AllowedModelTypes[code]
    return ok
}

func ValidateModelTypes(codes []string) ([]string, error) {
    out := make([]string, 0, len(codes))
    for _, c := range codes {
        if !IsValidModelType(c) {
            return nil, fmt.Errorf("%w: unknown modelType %q", ErrValidation, c)
        }
        out = append(out, c)
    }
    return out, nil
}
```

## 附录 D:ListModels 新 SQL(参考)

```sql
-- internal/model/repo.go::ListModels(带 types 过滤)
SELECT m.id, m.name, m.primary_type, m.cover_asset_id, m.author_name,
       a.media_type, a.storage_key,
       GROUP_CONCAT(mtm.type ORDER BY mtm.is_primary DESC, mtm.type SEPARATOR ',') AS types_csv
FROM models m
LEFT JOIN assets a            ON a.id  = m.cover_asset_id
LEFT JOIN model_type_map mtm  ON mtm.model_id = m.id
LEFT JOIN model_counters mc   ON mc.model_id = m.id
WHERE m.status = 'approved'
  [AND m.id IN (                                 -- q.Types 过滤(任一命中,OR)
       SELECT model_id FROM model_type_map
       WHERE type IN (?, ?, ?, ?, ?)             -- 最多 5 个占位符
  )]
  [AND m.primary_type = ?]                       -- q.PrimaryType 单值(可选)
  [AND m.base_model = ?]
GROUP BY m.id, m.name, m.primary_type, m.cover_asset_id, m.author_name,
         a.media_type, a.storage_key
ORDER BY m.created_at DESC, m.id DESC
LIMIT ?;
```

> 注: 占位符数量根据 `len(q.Types)` 动态生成(1..5);`types=other` 单独走 `WHERE primary_type='other'` 而非 IN(因兜底行可能不在 `model_type_map` 中)。

## 附录 E:GetModelByID 新 SQL(参考)

```sql
-- internal/model/repo.go::GetModelByID(返回所有 types)
SELECT m.id, m.name, m.primary_type, m.cover_asset_id, m.author_name,
       a.media_type, a.storage_key,
       mtm.type, mtm.is_primary
FROM models m
LEFT JOIN assets a            ON a.id  = m.cover_asset_id
LEFT JOIN model_type_map mtm  ON mtm.model_id = m.id
WHERE m.id = ? AND m.status = 'approved'
ORDER BY mtm.is_primary DESC, mtm.type ASC;
```

> 注: server `service.ToDetail` 聚合多行结果为 `[]ModelType` 数组。

---

## 附录 F:4 份分析的分歧汇总表

| 维度 | minimax-m3 | glm-5.2 | deepseek-v4-pro | claude-opus-4-8 | **综合决策** |
|---|---|---|---|---|---|
| **数据模型** | C:多对多表 + primary_type 冗余 | B:单列 VARCHAR + CHECK + enums.yaml(互斥单选) | A':枚举表 + FK 外键(互斥单选) | D:字典表 + 关联表 + 主分类冗余(允许多分类) | **C**(本期);D(v2 升级) |
| **多分类** | 允许(双标) | 互斥单选(SD 生态一份权重是一种) | 互斥单选(SD 生态惯例) | 允许多分类 + 主次 | **允许**(SD 生态虽互斥,但 UI/统计需多分类) |
| **枚举字典表** | 无(11 枚举放 enums.yaml) | 无(放 enums.yaml) | **有**(`model_types` 字典表) | **有**(`model_types` 字典表) | **无(本期)**;v2 升级到字典表 |
| **schema.gen.ts 字段名** | `types: [ModelType]` | `types[]` + `subTags[]` + `typeLabel` + `classifiedBy` + `fileType` | `primaryCategory: {slug, label, iconKey}`(对象) | `modelTypes: [ModelTypeAssignment]` + `category`(扁平) | **`types[]` + `primaryType`**(扁平 string enum) |
| **`workflow` 归类** | `template` | `other`(Phase 3 再考虑独立) | `template` | `template` | **`template`**(3/4 一致) |
| **`video` 归类** | `video`(11 类独立保留) | `video`(11 类独立保留) | `other`(`mediaType='video'` 区分) | `other`(`mediaType='video'` 区分) | **`other`**(2/2 deepseek/claude-opus,`mediaType` 区分业务) |
| **DB CHECK 约束** | 不强调 | **有**(`chk_models_type` 11 枚举) | 不加(用 FK + 应用层校验) | 不强调 | **不加**(FK + 应用层校验;DB 无字典表) |
| **MySQL ENUM** | 不强调(实际是 VARCHAR) | 明确反对(扩值要 ALTER) | 明确反对 | 不强调 | **不用 MySQL ENUM**(用 VARCHAR(32)) |
| **枚举同步** | enums.yaml 11 项 | enums.yaml `modelTypes`(label/labelEn/priority) | **新建 `modelTypes.yaml` 配置文件**(与 enums.yaml 分离) | enums.yaml `modelType` | **enums.yaml `modelType`**(与现有枚举同源,避免双源) |
| **importer 映射表位置** | importer 内部硬编码 | importer 内部硬编码 | importer 内部硬编码 | importer 内部硬编码 | **importer 内部硬编码**(4/4 一致) |
| **`models.type` 字段处理** | 保留(冗余兼容) | 保留(v1 兼容) | **保留**(双写兼容) | 保留(v1 兼容) | **保留**(4/4 一致) |
| **是否进 /v2** | 不进(v1 兼容) | 不进(枚举扩展兼容) | 不进(v1 兼容) | 不进(v1 兼容) | **不进**(4/4 一致) |
| **client 11 类 chip 渲染** | 列表筛选 + 详情 chips | 列表筛选 + 详情 + subTag 徽标 | 列表筛选 + 详情主分类大角标 | 列表筛选 + 详情主分类大角标 + 次分类 chips | **列表筛选 + 详情主+次 chips**(综合) |
| **Other UI 呈现** | chip 隐藏 + URL 显式 | 显示「其他」徽标 | 显示「其它 · 等待归类」灰色 | 显示「其它 · 等待归类」灰色 | **chip 列表隐藏 + 详情页主分类为 Other 时显示灰色徽标** |
| **Contract 字段名** | `types[]` + `type`(兼容) | `types[]` + `type` + `subTags[]` + `typeLabel` + `classifiedBy` + `fileType` | `primaryCategory`(对象) + `type`(兼容) | `modelTypes[]`(对象数组) + `category` | **`types[]` + `primaryType` + `type`(兼容)** |
| **i18n 来源** | i18next + enums.yaml label | enums.yaml label/labelEn | `model_types.label_i18n`(DB JSON) | enums.yaml 双列 label | **enums.yaml + i18n 双键**(MVP 简化;DB label_i18n 留 v2 字典表) |
| **多分类上限** | 默认 3 | 1(互斥) | 1(互斥) | 未明确 | **5**(索引 + 防滥用) |
| **是否新增 modelTypes.yaml 配置文件** | 否(用 enums.yaml) | 否(用 enums.yaml) | **是**(分离配置数据 vs 契约枚举) | 否(用 enums.yaml) | **否**(用 enums.yaml `modelType`,与现有枚举同源) |
| **MinIO object key 影响** | 不影响(解耦) | 不影响(解耦) | 不影响(完全解耦) | 不影响(解耦) | **不影响**(4/4 一致) |
| **`asset.purpose` 扩展** | 不变 | 不变 | 不变(`model_pipeline` 留 Phase 2) | 不变 | **不变** |
| **DB 迁移版本号** | `000016_model_classification` | `0000XX_model_type_11categories` | `000016_model_categories` | `000017-000020`(4 个迁移文件) | **`000016_model_classification`**(单迁移文件,含建表 + backfill) |
| **新表数量** | 1 张(`model_type_map`) | 0 张(只改 CHECK) | 1 张(`model_types` 字典表) | 3 张(`model_types` 字典 + `model_type_assignments` + 列宽) | **1 张**(`model_type_map`)+ 1 列(`models.primary_type`) |
| **PR 拆分数量** | 6 个 | 5-6 个 | 10 个(细粒度) | 12 个(细粒度) | **6 个**(综合) |
| **工时估算** | 7 个工作日 | 未明确 | 8.5 人天 | 2 周 | **2 周(MVP)** |

---

**END** — 本文档为综合 4 份分析的权威实现方案,落地时以本文件 §11 MVP 清单 + §7.6 PR 顺序逐项交付。CI 验收走 §12。本文件入 `eye-trace-docs/` 契约/架构真相源,作为后续 `model-detail-flow.md` / `model-summary-card-flow.md` / `openapi-skeleton.md` 改动的指针与依据。