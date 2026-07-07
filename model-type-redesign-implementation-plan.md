# /modelinfo 模型 11 分类与数据库格式重设计方案

> 产物类型:深度需求分析 + 实现方案  
> 项目:eye-trace monorepo  
> 目标路由:`http://localhost:1420/modelinfo/{id}`  
> 当前日期:2026-07-01  
> 结论先行:本需求不是“给 `models.type` 多加 6 个枚举值”,而是一次模型目录 taxonomy 重构。

---

## 0. 一句话结论

- 新增 11 类必须成为“模型分类 taxonomy”,不应继续复用当前 `models.type` 的混合语义。
- 当前 `models.type` 同时承载了模型资产类型、首页 tab、模板/工作流/视频入口等含义,已经耦合。
- 推荐方案是:新增 `model_types` 字典表 + `model_type_assignments` 多对多表 + `models.primary_model_type_code` 主分类快照。
- `models.type` 在 `/api/v1` 保留为 legacy 字段,只做兼容,不再作为新分类权威源。
- 分类权威源在 server DB,枚举允许值在 `eye-trace-config/shared/enums.yaml` 与 OpenAPI 中同源约束。
- MinIO 只存对象,不参与分类判断,object key 不能携带会变更的分类语义。
- `/models/{id}` 需要返回 `modelType` / `primaryModelType` / `modelTypes[]`,前端详情页直接消费生成类型。
- `/models` 列表需要支持 `modelType` 过滤、分页、排序,并保留 `Other` 兜底。
- `/api/v1` 用增量字段兼容旧客户端;真正移除 legacy `type` 放到 `/api/v2`。

---

## 1. 背景与已验证事实

- 仓库根目录:`/Users/mac/github/eye-trace`。
- 客户端在 `eye-trace-client/`。
- 客户端技术栈是 React 18 + Vite + Tauri 2 + Zustand。
- 服务端在 `eye-trace-server/backend/`。
- 服务端技术栈是 Go + chi + MySQL + MinIO。
- 契约骨架在 `eye-trace-docs/openapi-skeleton.md`。
- 全量 OpenAPI 在 `eye-trace-docs/openapi.yaml`。
- 共享枚举真相源在 `eye-trace-config/shared/enums.yaml`。
- 当前共享枚举只有 `supportedImageSizes`、`typeTabIds`、`jobStatus`。
- 当前共享枚举缺少 `modelType`。
- 当前 `/models` 列表由 `eye-trace-server/backend/internal/model/repo.go` 查询 `models` 表。
- 当前 `/models/{modelId}` 详情由 `GetModelDetailFull` + `buildModelDetailJSON` 组装。
- 当前前端列表 API 在 `eye-trace-client/src/api/models.ts`。
- 当前前端详情页在 `eye-trace-client/src/app/routes/ModelDetailPage.tsx`。
- 当前前端列表 hook 在 `eye-trace-client/src/features/models/useModelList.ts`。
- 当前服务端 importer 在 `cmd/importer/liblib-deep/models.go` 里硬编码 `pickModelTypeFromCode`。
- 当前旧 importer 在 `cmd/importer/liblib/helpers.go` 里通过 `modelTypeMap` 映射字符串。
- 当前 `models` 表 DDL 在 `migrations/000007_init_core_tables.up.sql`。
- 当前 `models.type` 注释为 `checkpoint / lora / template / workflow / video`。
- 当前 `openapi.yaml` 已经有旧 `ModelType` enum:`checkpoint / lora / template / workflow / video`。
- 当前 `eye-trace-client/src/api/schema.gen.ts` 也生成了旧 `ModelType`。
- 当前 `eye-trace-client/src/api/models.ts` 又手写了一份旧 `ModelType` union。
- 当前 `eye-trace-client/src/api/schemas.ts` 也手写了一份 `ModelTypeSchema`。
- 当前代码违反了“禁止手写请求/响应类型”的目标状态,至少在模型类型上存在漂移风险。

---

## 2. 需求拆解

- 用户访问 `/modelinfo/{id}`。
- 详情页当前从 server + MinIO 拉数据渲染。
- 现在需要新增模型分类 11 类。
- 11 类是:
- `Checkpoint`
- `Textual Inversion`
- `Hypernetwork`
- `Aesthetic Gradient`
- `模板`
- `LoRA`
- `LyCORIS`
- `Controlnet`
- `Poses`
- `Wildcards`
- `Other`
- 强约束是“数据库格式要重新设计”。
- 交付物是实现方案,不是只给代码 patch。
- 方案要覆盖业务目标、功能、非功能、数据模型、契约、集成、迁移、风险、MVP、文件路径。

---

## 3. 业务目标

- 让模型详情页展示准确模型分类。
- 让模型列表页可以按模型分类筛选。
- 让后续上传、导入、运营上架使用同一套分类规则。
- 让分类可以从 Liblib/Civitai/SD 生态扩展,但不被某一家外部源绑死。
- 让 `Other` 成为可观察的兜底,而不是静默脏数据。
- 让分类变更不影响 MinIO 对象 key 和缓存命中。
- 让客户端不再维护手写分类 union。
- 让 server 端成为状态权威,DB 成为分类 assignment 权威。
- 让 OpenAPI 继续作为 client/server 契约真相源。
- 让 `eye-trace-config/shared/enums.yaml` 补齐跨端枚举真相源。

---

## 4. 核心场景

### 4.1 详情展示

- 用户打开 `/modelinfo/15`。
- client 调用 `GET /api/v1/models/15`。
- server 查询 `models`、`model_details`、`model_counters`、`authors`、资产表、分类表。
- 响应返回 `primaryModelType` 和 `modelTypes`。
- 前端在详情卡片“类型”行展示分类中文名。
- 前端在标题下 tag 区展示主分类 badge。
- 若无明确分类,展示 `Other`,同时服务端记录可审计来源。

### 4.2 列表筛选

- 用户在模型列表页点击 `LoRA` 筛选。
- client 调用 `GET /api/v1/models?modelType=lora&limit=...`。
- server 按主分类或次分类匹配。
- 返回分页稳定的 `ModelSummary[]`。
- 前端瀑布流展示分类 badge。

### 4.3 多分类模型

- 一个资源可能既是 `LyCORIS`,又需要被 `LoRA` 生态下搜索到。
- 一个姿态资源可能既归 `Poses`,又和 `Controlnet` 工作流相关。
- 一个模板可能内含 checkpoint、LoRA、ControlNet 依赖。
- UI 需要一个“主分类”展示。
- 搜索/筛选需要支持“分类集合”匹配。
- 因此数据库必须支持多分类。

### 4.4 导入器归类

- Liblib deep importer 拿到 `data.modelType`、`tagsV2`、版本信息、标题、描述。
- importer 不应在 Go switch 里硬编码最终业务枚举。
- importer 应调用统一映射规则或读取 `model_type_source_mappings`。
- 不确定归类写入 `Other`,并写入 source/confidence。

### 4.5 上传/运营上架

- 后续 `POST /models` 或 admin 上架需要提交 `primaryModelType`。
- 可选提交多个 `modelTypes`。
- server 校验枚举是否合法。
- server 负责保证至少一个主分类。
- client 不允许提交未知字符串。

---

## 5. 当前模型分类设计的问题

### 5.1 `models.type` 语义混杂

- 当前 `models.type` 取值为 `checkpoint / lora / template / workflow / video`。
- `checkpoint` 和 `lora` 是模型资产类型。
- `template` 是产品交互形态。
- `workflow` 是工作流资源形态。
- `video` 更像媒体/业务域,不是 SD 模型类型。
- 这五个值不是同一层级 taxonomy。
- 继续往里面塞 11 类会扩大混乱。

### 5.2 `/models` query 参数漂移

- `openapi.yaml` 中 `GET /models` 的 query `type` 当前引用 `MediaType`。
- `MediaType` 只有 `image / video`。
- 服务端 `normalizeQuery` 却校验 `checkpoint / lora / template / workflow / video`。
- 这说明契约和实现已经漂移。
- 新需求不能在这个字段上继续叠语义。

### 5.3 client 手写类型漂移

- `eye-trace-client/src/api/models.ts` 手写:
- `export type ModelType = "checkpoint" | "lora" | "template" | "workflow" | "video";`
- `eye-trace-client/src/api/schemas.ts` 手写同一份 enum。
- `schema.gen.ts` 也有生成 enum。
- 三处同时存在时,后续必然出现漏改。
- 新方案必须把手写 union 删除或降级为 generated type alias。

### 5.4 importer 硬编码映射

- `cmd/importer/liblib-deep/models.go` 中 `pickModelTypeFromCode` 写死数字映射。
- 外部生态的数字 code 不稳定,且无法覆盖 11 类边界。
- 新增分类时不应每次改 importer switch。
- 应该有可配置映射表与审计信息。

### 5.5 MinIO 不应参与分类

- 当前图片和头像等资源走 `assets.storage_key`。
- `assets.storage_key` 指向 MinIO object key。
- 分类是业务元数据,不是对象存储路径。
- 如果 object key 携带分类,分类变更会导致对象迁移、缓存失效、历史 URL 失效。
- 所以分类必须在 MySQL,MinIO 只保存对象。

---

## 6. 分类编码规范

### 6.1 推荐 enum code

```yaml
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

### 6.2 显示标签

| code | zh-CN label | en label | 说明 |
|---|---|---|---|
| `checkpoint` | Checkpoint | Checkpoint | 基础权重/大模型检查点 |
| `textual_inversion` | Textual Inversion | Textual Inversion | embedding / TI |
| `hypernetwork` | Hypernetwork | Hypernetwork | SD 早期风格/特征注入机制 |
| `aesthetic_gradient` | Aesthetic Gradient | Aesthetic Gradient | 美学梯度资源,低频但保留 |
| `template` | 模板 | Template | 产品模板/参数模板/一键同款 |
| `lora` | LoRA | LoRA | 标准 LoRA |
| `lycoris` | LyCORIS | LyCORIS | LoCon/LoHa 等 LyCORIS 系 |
| `controlnet` | ControlNet | ControlNet | 控图模型/ControlNet 模块 |
| `poses` | Poses | Poses | 姿态/pose 资源 |
| `wildcards` | Wildcards | Wildcards | prompt wildcard 文本资源 |
| `other` | Other | Other | 未识别或无法归类 |

### 6.3 命名细节

- 枚举 code 使用小写 snake_case。
- `Controlnet` 展示名建议统一为 `ControlNet`。
- `LoRA` 展示名保留大小写。
- `LyCORIS` 展示名保留大小写。
- `模板` 的 code 使用 `template`,避免中文 enum。
- `Other` 的 code 使用 `other`,并永远保留。
- 不建议使用 `model_type` 作为 JSON 字段名;OpenAPI 中对外用 camelCase `modelType`。

---

## 7. 分类关系与业务边界

### 7.1 主分类与多分类

- 每个模型必须有一个主分类。
- 每个模型可以有 0 到 N 个附加分类。
- 主分类用于详情页类型行和卡片主 badge。
- 附加分类用于筛选命中、搜索召回、运营分析。
- 主分类写在 `models.primary_model_type_code`。
- 全量分类写在 `model_type_assignments`。

### 7.2 为什么不是纯单分类

- `LyCORIS` 与 `LoRA` 在用户心智上相关。
- `Poses` 常与 `ControlNet` 组合使用。
- `模板` 可能依赖 `LoRA`、`Checkpoint`、`ControlNet`。
- 单字段无法表达“展示主类”和“筛选可召回”的差异。
- 单字段会迫使 UI 和搜索丢失信息。

### 7.3 为什么不是纯 JSON

- MySQL JSON 查询可用,但索引和统计都更麻烦。
- 列表筛选是高频路径。
- 分类筛选会直接影响 `/models` 的 p95。
- JSON 不利于外键约束和枚举启停。
- JSON 不利于审计 source/confidence。
- 所以不能用 `models.model_types JSON` 作为主设计。

### 7.4 为什么仍保留主分类快照

- 列表页需要高频读取主分类。
- 主分类是每张卡片必需字段。
- 如果每次都 join assignments 和 dictionary,列表查询会变重。
- `models.primary_model_type_code` 是合理反范式。
- 多分类表仍是完整表达。

### 7.5 LoRA 与 LyCORIS

- `lycoris` 不应作为 `lora` 的字符串别名吞掉。
- 用户筛选 `LoRA` 时,产品可以选择是否包含 `LyCORIS`。
- 技术上通过 `model_types.parent_code = 'lora'` 或 UI 聚合实现。
- DB 主分类仍可明确为 `lycoris`。

### 7.6 Checkpoint 互斥性

- 单个模型权重 artifact 通常不会同时是 `checkpoint` 和 `lora`。
- 但产品里的“模板/合集”可能依赖多个模型类型。
- 不建议 DB 层硬限制 `checkpoint` 与其他类型互斥。
- 应在 importer/upload validator 里按 `resource_kind` 做软规则。
- 对单 artifact 上传可限制一个主分类。
- 对模板/合集可允许附加分类。

### 7.7 Template 的边界

- `template` 是产品层资源,不是纯 SD 权重类型。
- 如果 `/modelinfo/{id}` 承载模板详情,它仍需要纳入 taxonomy。
- 模板可以有附加分类表示依赖能力。
- 例如主类 `template`,附加 `controlnet`、`lora`。

### 7.8 Poses 的边界

- `poses` 可能是图片、骨架 JSON、OpenPose 图、姿态包。
- 它不一定是模型权重。
- 如果产品将其放入模型市场,可作为模型分类。
- MinIO asset `media_type` 不应被拿来推断它。

### 7.9 Wildcards 的边界

- `wildcards` 通常是文本资源。
- 它可能没有封面模型文件。
- 仍可在 `models` 中作为可展示资源。
- 生成链路需要判断它不是可直接运行模型。
- 后续可用 `resource_kind` 或 `runnable` 字段区分。

### 7.10 Aesthetic Gradient 的边界

- `aesthetic_gradient` 是低频历史资源。
- 不能因为低频就丢进 `other`。
- 但 UI 默认排序可以靠后。
- `enabled=false` 可以隐藏新建入口,不隐藏历史数据。

---

## 8. 推荐数据库设计

### 8.1 总体结构

- `models` 仍是模型主表。
- `model_types` 是分类字典表。
- `model_type_assignments` 是模型与分类的 assignment 表。
- `model_type_source_mappings` 是外部来源映射表。
- `model_type_migration_audit` 是回填审计表,可选但推荐。

### 8.2 `models` 表新增列

```sql
ALTER TABLE models
  ADD COLUMN catalog_type VARCHAR(32) NOT NULL DEFAULT 'image' AFTER type,
  ADD COLUMN primary_model_type_code VARCHAR(64) NOT NULL DEFAULT 'other' AFTER catalog_type,
  ADD COLUMN model_type_source VARCHAR(32) NOT NULL DEFAULT 'backfill' AFTER primary_model_type_code,
  ADD COLUMN model_type_confidence DECIMAL(5,4) NOT NULL DEFAULT 0.0000 AFTER model_type_source,
  ADD COLUMN taxonomy_version INT UNSIGNED NOT NULL DEFAULT 1 AFTER model_type_confidence,
  ADD KEY idx_models_primary_type_created (primary_model_type_code, status, created_at DESC, id DESC),
  ADD KEY idx_models_catalog_created (catalog_type, status, created_at DESC, id DESC);
```

### 8.3 `catalog_type` 的用途

- `catalog_type` 表示产品目录域。
- 初始值可使用 `image / video / ideas / flows`。
- 它对应当前 `typeTabIds`。
- 它不等于 `modelType`。
- 它取代把 `workflow / video` 塞进 `models.type` 的做法。
- 对 `/models?type=...` 的历史混乱,后续用 `typeTabId` 或 `catalogType` 清理。

### 8.4 `primary_model_type_code` 的用途

- 表示详情页展示的主分类。
- 取值来自 11 类。
- 不允许为空。
- 默认 `other`。
- 与 `model_type_assignments.is_primary=1` 保持一致。

### 8.5 `model_type_source`

- 表示主分类来源。
- 推荐值:
- `manual`
- `admin`
- `importer`
- `mapping`
- `heuristic`
- `backfill`
- `user`
- `system`
- 这不是跨端 enum,可以先用 VARCHAR。

### 8.6 `model_type_confidence`

- 表示分类置信度。
- `1.0000` 表示人工确认或外部强映射。
- `0.7000` 表示规则推断。
- `0.3000` 表示弱关键词猜测。
- `0.0000` 表示 `other` 或未知。
- 后续运营后台可以按低置信度筛查。

### 8.7 `taxonomy_version`

- 表示当前分类规则版本。
- 初始为 `1`。
- 以后分类规则变化时可增量回填。
- 防止不知道某行是用哪套规则归类的。

### 8.8 `model_types` DDL

```sql
CREATE TABLE IF NOT EXISTS model_types (
  code          VARCHAR(64)  NOT NULL,
  label_zh      VARCHAR(64)  NOT NULL,
  label_en      VARCHAR(64)  NOT NULL,
  parent_code   VARCHAR(64)  NULL,
  description   VARCHAR(512) NULL,
  sort_order    INT          NOT NULL DEFAULT 0,
  enabled       TINYINT      NOT NULL DEFAULT 1,
  created_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (code),
  KEY idx_model_types_enabled_sort (enabled, sort_order),
  KEY idx_model_types_parent (parent_code),
  CONSTRAINT fk_model_types_parent
    FOREIGN KEY (parent_code) REFERENCES model_types(code)
    ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.9 `model_types` seed

```sql
INSERT INTO model_types
  (code, label_zh, label_en, parent_code, sort_order, enabled, description)
VALUES
  ('checkpoint', 'Checkpoint', 'Checkpoint', NULL, 10, 1, 'Base checkpoint model'),
  ('textual_inversion', 'Textual Inversion', 'Textual Inversion', NULL, 20, 1, 'Embedding/Textual Inversion resource'),
  ('hypernetwork', 'Hypernetwork', 'Hypernetwork', NULL, 30, 1, 'Hypernetwork resource'),
  ('aesthetic_gradient', 'Aesthetic Gradient', 'Aesthetic Gradient', NULL, 40, 1, 'Aesthetic gradient resource'),
  ('template', '模板', 'Template', NULL, 50, 1, 'Product template or one-click generation template'),
  ('lora', 'LoRA', 'LoRA', NULL, 60, 1, 'LoRA model'),
  ('lycoris', 'LyCORIS', 'LyCORIS', 'lora', 70, 1, 'LyCORIS/LoCon/LoHa model'),
  ('controlnet', 'ControlNet', 'ControlNet', NULL, 80, 1, 'ControlNet model/resource'),
  ('poses', 'Poses', 'Poses', 'controlnet', 90, 1, 'Pose/OpenPose resource'),
  ('wildcards', 'Wildcards', 'Wildcards', NULL, 100, 1, 'Prompt wildcard resource'),
  ('other', 'Other', 'Other', NULL, 999, 1, 'Fallback model type')
ON DUPLICATE KEY UPDATE
  label_zh = VALUES(label_zh),
  label_en = VALUES(label_en),
  parent_code = VALUES(parent_code),
  sort_order = VALUES(sort_order),
  enabled = VALUES(enabled),
  description = VALUES(description);
```

### 8.10 `model_type_assignments` DDL

```sql
CREATE TABLE IF NOT EXISTS model_type_assignments (
  id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  model_id        BIGINT UNSIGNED NOT NULL,
  model_type_code VARCHAR(64)     NOT NULL,
  is_primary      TINYINT         NOT NULL DEFAULT 0,
  source          VARCHAR(32)     NOT NULL DEFAULT 'system',
  confidence      DECIMAL(5,4)    NOT NULL DEFAULT 1.0000,
  sort_order      INT             NOT NULL DEFAULT 0,
  created_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  primary_model_id BIGINT UNSIGNED
    GENERATED ALWAYS AS (IF(is_primary = 1, model_id, NULL)) STORED,
  PRIMARY KEY (id),
  UNIQUE KEY uk_mta_model_type (model_id, model_type_code),
  UNIQUE KEY uk_mta_one_primary (primary_model_id),
  KEY idx_mta_type_model (model_type_code, model_id),
  KEY idx_mta_model_primary (model_id, is_primary),
  KEY idx_mta_source_confidence (source, confidence),
  CONSTRAINT fk_mta_model
    FOREIGN KEY (model_id) REFERENCES models(id)
    ON DELETE CASCADE,
  CONSTRAINT fk_mta_type
    FOREIGN KEY (model_type_code) REFERENCES model_types(code)
    ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.11 为什么使用 generated column 保证唯一主分类

- MySQL 没有 PostgreSQL 那种 partial unique index。
- `UNIQUE(model_id, is_primary)` 会导致每个模型只能有一个非主分类,不可用。
- generated column `primary_model_id = IF(is_primary=1, model_id, NULL)` 可实现“每个模型最多一个 primary”。
- 多个 NULL 在 MySQL unique index 下允许共存。
- 这样可以允许 N 个 secondary。

### 8.12 `model_type_source_mappings` DDL

```sql
CREATE TABLE IF NOT EXISTS model_type_source_mappings (
  id                  BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  source              VARCHAR(32)     NOT NULL,
  external_type_code  VARCHAR(64)     NULL,
  external_type_name  VARCHAR(128)    NULL,
  keyword             VARCHAR(128)    NULL,
  model_type_code     VARCHAR(64)     NOT NULL,
  confidence          DECIMAL(5,4)    NOT NULL DEFAULT 1.0000,
  priority            INT             NOT NULL DEFAULT 100,
  enabled             TINYINT         NOT NULL DEFAULT 1,
  note                VARCHAR(512)    NULL,
  created_at          TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at          TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_mtsm_source_code (source, external_type_code, enabled, priority),
  KEY idx_mtsm_source_name (source, external_type_name, enabled, priority),
  KEY idx_mtsm_keyword (source, keyword, enabled, priority),
  CONSTRAINT fk_mtsm_type
    FOREIGN KEY (model_type_code) REFERENCES model_types(code)
    ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.13 `model_type_source_mappings` 的价值

- 把外部源映射从 Go switch 中剥离出来。
- 支持 Liblib、Civitai、手动导入等多来源。
- 支持同一来源下数字 code 和字符串 name 双通道。
- 支持关键词规则作为兜底。
- 支持运营调整映射而不重编译 importer。

### 8.14 `model_type_migration_audit` DDL

```sql
CREATE TABLE IF NOT EXISTS model_type_migration_audit (
  id                    BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  model_id              BIGINT UNSIGNED NOT NULL,
  old_type              VARCHAR(64)     NULL,
  new_primary_type_code VARCHAR(64)     NOT NULL,
  assigned_types_json   JSON            NULL,
  source                VARCHAR(32)     NOT NULL,
  confidence            DECIMAL(5,4)    NOT NULL,
  rule_version          INT UNSIGNED    NOT NULL,
  reason                VARCHAR(1024)   NULL,
  created_at            TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_mtma_model (model_id),
  KEY idx_mtma_type (new_primary_type_code),
  KEY idx_mtma_confidence (confidence)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.15 审计表是否必须

- MVP 可以不建审计表。
- 但如果存量数据来自 Liblib 抓取,推荐建。
- 分类回填很容易产生错误。
- 审计表能解释某条为什么被归为 `other` 或 `lycoris`。
- 后续运营后台可以按审计记录纠错。

---

## 9. 数据模型方案对比

### 9.1 方案 A:只扩 `models.type` 字段

- 做法:把 `models.type` enum 扩成 11 类。
- 优点:改动最少。
- 缺点:无法多分类。
- 缺点:会破坏 `workflow / video` 老语义。
- 缺点:列表筛选和详情展示复用同一字段,无法表达附加分类。
- 缺点:与当前 OpenAPI `type` 参数漂移叠加。
- 结论:不推荐。

### 9.2 方案 B:`models.model_types JSON`

- 做法:主表加 JSON 数组。
- 优点:写入简单。
- 优点:不需要 join。
- 缺点:高频筛选索引差。
- 缺点:无 FK 约束。
- 缺点:难以表达 source/confidence/isPrimary。
- 缺点:迁移和统计不清晰。
- 结论:不推荐作为主设计。

### 9.3 方案 C:纯枚举表 + 多对多表

- 做法:所有分类都 join dictionary 和 assignment。
- 优点:范式干净。
- 优点:多分类和审计自然。
- 缺点:列表页每条都需要 join/聚合。
- 缺点:主分类展示字段成本偏高。
- 结论:可行,但列表路径需要优化。

### 9.4 方案 D:字典表 + 多对多 + 主表主分类快照

- 做法:`models.primary_model_type_code` + `model_type_assignments` + `model_types`。
- 优点:列表快。
- 优点:详情完整。
- 优点:多分类可表达。
- 优点:主分类可索引。
- 优点:与 v1 兼容最好。
- 缺点:需要保证主表快照与 assignment 一致。
- 结论:推荐。

---

## 10. MinIO 与对象 key 设计

### 10.1 原则

- MinIO object key 是物理存储定位。
- 模型分类是业务属性。
- 两者不应互相推断。
- 分类变更不应移动对象。
- 对象迁移不应改变分类。

### 10.2 当前合理做法

- 当前 `assets` 表保存:
- `storage_backend`
- `storage_key`
- `media_type`
- `mime`
- `width`
- `height`
- server 通过 `assets.URL(id)` 或 `DirectURL(key)` 输出 URL。
- `/models` 和 `/models/{id}` 只拼 URL,不碰 MinIO。
- 这点应保留。

### 10.3 不推荐的 key

```text
models/lora/{model_id}/cover.png
models/controlnet/{model_id}/cover.png
```

- 这种 key 把分类写入路径。
- 模型从 `lora` 改为 `lycoris` 会要迁移对象。
- CDN 旧 URL 可能失效。
- DB 和对象存储会出现双写不一致。

### 10.4 推荐的 key

```text
models/{model_id}/cover/{asset_id}.{ext}
models/{model_id}/versions/{version_id}/artifact/{sha256}.{ext}
models/{model_id}/gallery/{asset_id}.{ext}
models/{model_id}/author_avatar/{asset_id}.{ext}
```

- 路径只包含稳定 ID 和 asset role。
- 分类变化不改变 key。
- asset role 是物理用途,不是业务 taxonomy。

### 10.5 MinIO 延迟优化

- `/models` 列表不得同步读取 MinIO object。
- `/models/{id}` 详情不得同步读取 MinIO object。
- URL 拼接只依赖 MySQL `assets` 行。
- 图片实际加载由浏览器请求 `/api/v1/assets/{id}/content` 或 CDN。
- `assets` 代理端点应继续提供 ETag / Cache-Control。
- 详情 JSON 可以设置短 TTL 或 ETag。
- `model_types` 字典可以长 TTL。

---

## 11. OpenAPI 契约影响

### 11.1 `openapi-skeleton.md` 必须先改

- `eye-trace-docs/openapi-skeleton.md` 是契约骨架。
- 新增 schema 必须先进入 skeleton。
- 再同步到 `openapi.yaml`。
- 再生成 client/server types。
- 不允许 client/server 先手写临时类型。

### 11.2 新增 `ModelType`

```yaml
ModelType:
  type: string
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
```

### 11.3 新增 `ModelTypeInfo`

```yaml
ModelTypeInfo:
  type: object
  required: [code, label, sortOrder, enabled]
  properties:
    code: { $ref: '#/components/schemas/ModelType' }
    label: { type: string }
    labelEn: { type: string, nullable: true }
    parentCode: { $ref: '#/components/schemas/ModelType', nullable: true }
    description: { type: string, nullable: true }
    sortOrder: { type: integer }
    enabled: { type: boolean }
```

### 11.4 新增 `ModelClassification`

```yaml
ModelClassification:
  type: object
  required: [code, label, primary]
  properties:
    code: { $ref: '#/components/schemas/ModelType' }
    label: { type: string }
    primary: { type: boolean }
    confidence: { type: number, minimum: 0, maximum: 1, nullable: true }
    source: { type: string, nullable: true }
```

### 11.5 `ModelSummary` v1 增量字段

```yaml
ModelSummary:
  properties:
    modelType:
      $ref: '#/components/schemas/ModelType'
      description: 主分类 code;v1 additive field
    modelTypeLabel:
      type: string
      description: 主分类展示名
    modelTypes:
      type: array
      items: { $ref: '#/components/schemas/ModelClassification' }
      description: 主分类 + 附加分类
```

### 11.6 `ModelSummary.type` 的兼容策略

- v1 保留 `type`。
- v1 中 `type` 标记 deprecated。
- v1 中 `type` 继续返回旧 5 类兼容值。
- v1 新 UI 不再使用 `type`。
- v2 删除或重命名 `type`。

### 11.7 `ModelDetail` v1 增量字段

```yaml
ModelDetail:
  properties:
    modelType:
      $ref: '#/components/schemas/ModelType'
    modelTypeLabel:
      type: string
    modelTypes:
      type: array
      items: { $ref: '#/components/schemas/ModelClassification' }
    primaryModelType:
      $ref: '#/components/schemas/ModelClassification'
```

### 11.8 `GET /models` 参数变化

```yaml
/models:
  get:
    parameters:
      - in: query
        name: modelType
        schema:
          type: array
          items: { $ref: '#/components/schemas/ModelType' }
        style: form
        explode: true
        description: 按模型分类过滤;重复参数 OR 匹配
      - in: query
        name: primaryModelType
        schema: { $ref: '#/components/schemas/ModelType' }
        description: 只按主分类过滤
      - in: query
        name: typeTabId
        schema:
          type: string
          enum: [image, video, ideas, flows]
        description: 顶层产品 tab;替代旧 type 混合语义
```

### 11.9 过滤语义

- `modelType=lora&modelType=lycoris` 默认是 OR。
- `primaryModelType=lora` 只匹配主分类。
- `modelType=lora` 默认匹配主分类或附加分类。
- `typeTabId=image` 匹配产品目录域。
- `type` 参数在 v1 可保留但 deprecated。

### 11.10 新增 `GET /model-types`

```yaml
/model-types:
  get:
    summary: 模型分类字典
    responses:
      '200':
        content:
          application/json:
            schema:
              type: object
              required: [items]
              properties:
                items:
                  type: array
                  items: { $ref: '#/components/schemas/ModelTypeInfo' }
```

### 11.11 为什么需要 `/model-types`

- UI 需要 label、排序、enabled。
- `enums.yaml` 只适合定义允许值。
- DB 才是运行时展示和运营启停的权威。
- `/model-types` 可以长期缓存。
- 客户端无需硬编码 11 个 label。

### 11.12 `/api/v2` 的形态

- `/api/v1` 目标是兼容。
- `/api/v2` 目标是清理命名。
- v2 中 `type` 不再表示模型分类。
- v2 推荐字段:
- `catalogType`
- `modelType`
- `modelTypes`
- `mediaType`
- `resourceKind`
- v2 query 推荐:
- `catalogType=image`
- `modelType=lora`
- `sort=latest`

---

## 12. 共享枚举影响

### 12.1 `eye-trace-config/shared/enums.yaml`

需要新增:

```yaml
# === Model taxonomy ===
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

### 12.2 `typeTabIds` 不要复用

- `typeTabIds` 是顶层产品 tab。
- 它当前取值是 `image / video / ideas / flows`。
- 这不是模型分类。
- 不要把 11 类塞进 `typeTabIds`。
- 新增 `modelType` 是正确边界。

### 12.3 导出器现状

- `eye-trace-config/README.md` 描述了 `source/`、`exporters/`、`dist/`。
- 但当前实际只有 `shared/enums.yaml` 和 `home/*.example.yaml`。
- 因此本次 MVP 可先更新 `shared/enums.yaml`。
- 后续需要补导出器或继续用 OpenAPI 生成兜底。
- 不能在 client/server 各自复制 enum。

### 12.4 枚举权威与 DB 权威的分工

- 枚举允许值:OpenAPI + `enums.yaml`。
- 分类 assignment:server DB。
- 分类 label/order/enabled:server DB `model_types`。
- 客户端类型:OpenAPI 生成。
- 服务端类型:oapi-codegen 生成。

---

## 13. Server 实现方案

### 13.1 新 migration

- 新建 `eye-trace-server/backend/migrations/000017_model_types.up.sql`。
- 新建 `eye-trace-server/backend/migrations/000017_model_types.down.sql`。
- up migration 做:
- 创建 `model_types`。
- seed 11 类。
- 给 `models` 加 `catalog_type`。
- 给 `models` 加 `primary_model_type_code`。
- 给 `models` 加 source/confidence/taxonomy_version。
- 创建 `model_type_assignments`。
- 创建 `model_type_source_mappings`。
- 可选创建 `model_type_migration_audit`。
- 回填现有数据。
- down migration 尽量 drop 新表和新列。

### 13.2 回填 SQL 第一层

```sql
UPDATE models
SET catalog_type = CASE
  WHEN type = 'video' THEN 'video'
  WHEN type = 'workflow' THEN 'flows'
  ELSE 'image'
END
WHERE catalog_type IS NULL OR catalog_type = '';
```

### 13.3 回填 primary type 第一层

```sql
UPDATE models
SET primary_model_type_code = CASE
  WHEN type = 'checkpoint' THEN 'checkpoint'
  WHEN type = 'lora' THEN 'lora'
  WHEN type = 'template' THEN 'template'
  ELSE 'other'
END,
model_type_source = 'backfill',
model_type_confidence = CASE
  WHEN type IN ('checkpoint', 'lora', 'template') THEN 0.9000
  ELSE 0.3000
END
WHERE primary_model_type_code IS NULL
   OR primary_model_type_code = ''
   OR primary_model_type_code = 'other';
```

### 13.4 回填 assignments

```sql
INSERT INTO model_type_assignments
  (model_id, model_type_code, is_primary, source, confidence)
SELECT id, primary_model_type_code, 1, model_type_source, model_type_confidence
FROM models
ON DUPLICATE KEY UPDATE
  is_primary = VALUES(is_primary),
  source = VALUES(source),
  confidence = VALUES(confidence);
```

### 13.5 基于 raw payload 的二次回填

- 如果 `model_details.raw_payload` 中可解析外部 `modelType`,优先使用 source mapping。
- 如果 tags 中出现 `LyCORIS`、`LoCon`、`LoHa`,映射 `lycoris`。
- 如果 tags/name/description 出现 `ControlNet`,映射 `controlnet`。
- 如果出现 `OpenPose`、`Pose`,映射 `poses`。
- 如果出现 `Wildcard`,映射 `wildcards`。
- 如果出现 `Embedding`、`Textual Inversion`,映射 `textual_inversion`。
- 如果出现 `Hypernetwork`,映射 `hypernetwork`。
- 如果出现 `Aesthetic Gradient`,映射 `aesthetic_gradient`。
- 弱关键词的 confidence 不应超过 `0.7000`。
- 弱规则回填应该写审计。

### 13.6 Repo 层改动

目标文件:

- `eye-trace-server/backend/internal/model/repo.go`
- `eye-trace-server/backend/internal/model/service.go`
- `eye-trace-server/backend/internal/model/base_models.go`
- `eye-trace-server/backend/internal/api/handler/model.go`

### 13.7 `ListQuery` 改动

```go
type ListQuery struct {
    Type             string // legacy, deprecated
    TypeTabID        string
    CatalogType      string
    ModelTypes       []string
    PrimaryModelType string
    BaseModel        string
    Tags             []string
    Search           string
    Cursor           string
    Limit            int
    Sort             Sort
}
```

### 13.8 `normalizeQuery` 改动

- 不再把 `q.Type` 当新分类。
- 对 `q.ModelTypes` 校验 11 类。
- 对 `q.PrimaryModelType` 校验 11 类。
- 对 `q.TypeTabID` 校验 `typeTabIds`。
- legacy `q.Type` 只允许旧客户端使用,并转换为 `CatalogType` 或 legacy filter。

### 13.9 列表 SQL 改动

当前查询:

```sql
WHERE m.status = 'approved'
  AND m.type = ?
```

目标查询:

```sql
WHERE m.status = 'approved'
  AND (? = '' OR m.catalog_type = ?)
  AND (? = '' OR m.primary_model_type_code = ?)
  AND (
    ? = 0
    OR m.primary_model_type_code IN (...)
    OR EXISTS (
      SELECT 1
      FROM model_type_assignments mta
      WHERE mta.model_id = m.id
        AND mta.model_type_code IN (...)
    )
  )
```

### 13.10 避免重复行

- 如果列表直接 join `model_type_assignments`,多分类会造成重复模型行。
- 推荐用 `EXISTS` 做筛选。
- 只有需要聚合 `modelTypes[]` 时才单独查询或用批量查询。
- 列表页先返回主分类即可,MVP 不必为每个 summary 聚合所有分类。

### 13.11 `ModelRow` 改动

```go
type ModelRow struct {
    ID                   int64
    Name                 string
    Type                 string // legacy
    CatalogType          string
    PrimaryModelTypeCode string
    PrimaryModelTypeLabel string
    ...
}
```

### 13.12 `Summary` 改动

```go
type Summary struct {
    ModelID        string                 `json:"modelId"`
    Title          string                 `json:"title"`
    Cover          string                 `json:"cover"`
    MediaType      string                 `json:"mediaType"`
    Type           string                 `json:"type"` // legacy
    ModelType      string                 `json:"modelType"`
    ModelTypeLabel string                 `json:"modelTypeLabel"`
    ModelTypes     []ModelClassification  `json:"modelTypes,omitempty"`
    ...
}
```

### 13.13 `DetailFullResult` 改动

```go
type DetailFullResult struct {
    ...
    Type                  string // legacy
    CatalogType           string
    PrimaryModelTypeCode  string
    PrimaryModelTypeLabel string
    ModelTypes            []ModelClassification
}
```

### 13.14 分类查询接口

Repo 新增:

```go
ListModelTypes(ctx context.Context) ([]ModelTypeRow, error)
ListModelTypeAssignments(ctx context.Context, modelIDs []int64) (map[int64][]ModelTypeAssignmentRow, error)
GetModelTypeAssignments(ctx context.Context, modelID int64) ([]ModelTypeAssignmentRow, error)
```

### 13.15 Handler 新增

- `GET /api/v1/model-types`。
- handler 可放在 `internal/api/handler/model.go` 或独立 `model_type.go`。
- `cmd/server/main.go` 注册时接入同一个 service。

### 13.16 Service 层缓存

- `model_types` 字典可进内存缓存。
- TTL 可以是 5 分钟。
- 或启动加载,admin 改动后重启。
- MVP 用 DB 查询也可接受,因为字典只有 11 行。

### 13.17 importer 改动

目标文件:

- `eye-trace-server/backend/cmd/importer/liblib-deep/models.go`
- `eye-trace-server/backend/cmd/importer/liblib/helpers.go`
- `eye-trace-server/backend/cmd/importer/liblib/importer.go`
- `eye-trace-server/backend/cmd/importer/liblib-detail/main.go`

### 13.18 importer 应做的事

- 计算 `legacyType` 保持 v1 兼容。
- 计算 `primaryModelTypeCode` 写新字段。
- 写入 `model_type_assignments`。
- 写入 `source/confidence`。
- 不再只依赖 `pickModelTypeFromCode`。
- 映射失败时写 `other`。
- 映射失败要 log 出 `external modelType`、`name`、`uuid`。

### 13.19 importer 写入示意

```sql
INSERT INTO models (
  name,
  type,
  catalog_type,
  primary_model_type_code,
  model_type_source,
  model_type_confidence,
  ...
) VALUES (?, ?, ?, ?, ?, ?, ...)
ON DUPLICATE KEY UPDATE
  name = VALUES(name),
  type = VALUES(type),
  catalog_type = VALUES(catalog_type),
  primary_model_type_code = VALUES(primary_model_type_code),
  model_type_source = VALUES(model_type_source),
  model_type_confidence = VALUES(model_type_confidence),
  ...
```

### 13.20 assignment 写入示意

```sql
INSERT INTO model_type_assignments
  (model_id, model_type_code, is_primary, source, confidence)
VALUES (?, ?, 1, ?, ?)
ON DUPLICATE KEY UPDATE
  is_primary = VALUES(is_primary),
  source = VALUES(source),
  confidence = VALUES(confidence);
```

---

## 14. Client 实现方案

### 14.1 类型生成

- 更新 `openapi-skeleton.md`。
- 更新 `openapi.yaml`。
- 运行 client openapi-typescript。
- 更新 `eye-trace-client/src/api/schema.gen.ts`。
- 不手写新 union。
- 删除或替换 `src/api/models.ts` 中的手写 `ModelType`。

### 14.2 `src/api/models.ts`

当前:

```ts
export type ModelType = "checkpoint" | "lora" | "template" | "workflow" | "video";
```

目标:

```ts
export type ModelType = components["schemas"]["ModelType"];
export type ModelClassification = components["schemas"]["ModelClassification"];
export type ModelTypeInfo = components["schemas"]["ModelTypeInfo"];
```

### 14.3 `ListModelsParams`

```ts
export interface ListModelsParams {
  modelType?: ModelType[];
  primaryModelType?: ModelType;
  typeTabId?: string;
  baseModel?: string;
  tag?: string[];
  search?: string;
  cursor?: string;
  limit?: number;
  signal?: AbortSignal;
}
```

### 14.4 query 拼接

```ts
if (params.modelType?.length) {
  for (const t of params.modelType) search.append("modelType", t);
}
if (params.primaryModelType) search.set("primaryModelType", params.primaryModelType);
if (params.typeTabId) search.set("typeTabId", params.typeTabId);
```

### 14.5 `summaryToCard`

当前:

```ts
typeTag: s.typeTag ?? s.type
```

目标:

```ts
typeTag: s.typeTag ?? s.modelTypeLabel ?? s.modelType ?? s.type
```

### 14.6 详情页 `modelTypeLabel`

当前 `ModelDetailPage.tsx` 中有硬编码 switch。

目标:

- 优先使用 `detail.modelTypeLabel`。
- 其次从 `detail.primaryModelType.label` 取。
- 最后用 generated label map 或 code 原样兜底。

```ts
function modelTypeLabel(detail: ApiModelDetail): string {
  return detail.modelTypeLabel
    ?? detail.primaryModelType?.label
    ?? detail.modelType
    ?? detail.type;
}
```

### 14.7 `src/api/schemas.ts`

- 如果该文件仍用于运行时 zod 校验,必须更新 `ModelTypeSchema`。
- 更好的方向是从 OpenAPI schema 生成 zod,不要手写。
- MVP 可同步更新,但要标注 TODO 删除手写 schema。

### 14.8 `useModelList`

- 当前 `useModelList()` 没有筛选参数。
- 目标支持:
- `useModelList({ modelType, typeTabId, sort })`。
- 筛选变化时清空 `items`、`cursor`、`seenIds`。
- 使用 AbortController 取消旧请求。
- 保留防 StrictMode 双请求逻辑。

### 14.9 UI 筛选

目标文件:

- `eye-trace-client/src/app/routes/ImageModelListPage.tsx`
- `eye-trace-client/src/components/layout/TypeTabs.tsx`
- `eye-trace-client/src/components/layout/SubTabs.tsx`
- `eye-trace-client/src/components/layout/ModelItemList.tsx`

建议:

- 顶部保持 `typeTabIds`:图片模型 / 视频特效 / 发现灵感 / 工作流。
- 11 类作为二级筛选 chips。
- `Other` 放最后。
- 默认不选任何分类,表示全部。
- 多选时 OR 匹配。
- 若产品要简单,MVP 先单选。

### 14.10 UI 排序

- 当前契约只支持 `sort=latest`。
- 需求提到筛选/排序,建议接口先预留。
- MVP 保持 `latest`。
- Phase 2 增加:
- `popular` -> `model_counters.heat`
- `mostUsed` -> `model_counters.run_count`
- `mostImages` -> `model_counters.image_count`
- `updated` -> `models.updated_at`

### 14.11 降级策略

- 如果 server 没有返回 `modelType`,前端用 `typeTag ?? type`。
- 如果 `/model-types` 失败,前端用 OpenAPI enum fallback label map。
- 如果某行返回未知 modelType,前端展示原字符串并上报。
- `Other` 永远可展示。

---

## 15. API 响应示例

### 15.1 `GET /api/v1/model-types`

```json
{
  "items": [
    {
      "code": "checkpoint",
      "label": "Checkpoint",
      "labelEn": "Checkpoint",
      "parentCode": null,
      "sortOrder": 10,
      "enabled": true
    },
    {
      "code": "lycoris",
      "label": "LyCORIS",
      "labelEn": "LyCORIS",
      "parentCode": "lora",
      "sortOrder": 70,
      "enabled": true
    },
    {
      "code": "other",
      "label": "Other",
      "labelEn": "Other",
      "parentCode": null,
      "sortOrder": 999,
      "enabled": true
    }
  ]
}
```

### 15.2 `GET /api/v1/models?modelType=lora`

```json
{
  "items": [
    {
      "modelId": "15",
      "title": "动漫游戏CG质感角色V2｜次时代3D漫剧",
      "cover": "http://127.0.0.1:8080/api/v1/assets/106/content",
      "mediaType": "image",
      "type": "template",
      "modelType": "template",
      "modelTypeLabel": "模板",
      "modelTypes": [
        {
          "code": "template",
          "label": "模板",
          "primary": true,
          "source": "importer",
          "confidence": 1
        },
        {
          "code": "lora",
          "label": "LoRA",
          "primary": false,
          "source": "heuristic",
          "confidence": 0.7
        }
      ],
      "useCount": 22800,
      "imageCount": 389,
      "viewId": "model.detail"
    }
  ],
  "nextCursor": null,
  "totalHint": 1
}
```

### 15.3 `GET /api/v1/models/{id}`

```json
{
  "id": "15",
  "title": "动漫游戏CG质感角色V2｜次时代3D漫剧",
  "type": "template",
  "modelType": "template",
  "modelTypeLabel": "模板",
  "primaryModelType": {
    "code": "template",
    "label": "模板",
    "primary": true,
    "source": "importer",
    "confidence": 1
  },
  "modelTypes": [
    {
      "code": "template",
      "label": "模板",
      "primary": true,
      "source": "importer",
      "confidence": 1
    }
  ],
  "mediaType": "image",
  "cover": "http://127.0.0.1:8080/api/v1/assets/106/content",
  "description": "<p>...</p>"
}
```

---

## 16. v1 与 v2 策略

### 16.1 v1 可做的兼容改动

- 新增响应字段。
- 新增 query 参数。
- 新增 `/model-types`。
- 保留 `type` 字段。
- 保留旧 `ModelType` 的消费方式一段时间。
- 不把 `type` 的含义强行改成 11 类。

### 16.2 v1 不应做的事

- 不删除 `type`。
- 不把 `type` 从旧 5 类直接变成 11 类。
- 不让 `GET /models?type=lora` 同时有三种含义。
- 不让客户端手写兼容类型。

### 16.3 v2 应做的事

- 删除 legacy `type`。
- 明确 `catalogType`。
- 明确 `modelType`。
- 明确 `mediaType`。
- 明确 `resourceKind`。
- query 参数改为:
- `catalogType`
- `modelType`
- `primaryModelType`
- `baseModel`
- `tag`
- `search`
- `sort`
- `cursor`
- `limit`

### 16.4 推荐时间线

- Phase 1: v1 additive。
- Phase 2: 前端全量切到 `modelType`。
- Phase 3: server 继续返回 legacy `type`,但监控旧字段使用。
- Phase 4: `/api/v2` 清理旧命名。

---

## 17. 存量迁移方案

### 17.1 迁移原则

- 先扩表,后回填,再切读。
- 新旧字段双写一段时间。
- 任何无法确定的都归 `other`。
- `other` 不是失败,是需要运营审查的状态。
- 迁移不改 MinIO object key。
- 迁移不删除 `models.type`。

### 17.2 迁移步骤

1. 部署 DB migration。
2. seed 11 个 `model_types`。
3. 给 `models` 添加新列。
4. 根据旧 `models.type` 做第一轮回填。
5. 根据 `model_details.raw_payload` 做第二轮回填。
6. 根据 `model_details.liblib_tags_v2` 做第三轮回填。
7. 写 `model_type_assignments`。
8. 跑一致性校验。
9. 部署 server 双读/双写。
10. 重新生成 OpenAPI types。
11. 部署 client 消费新字段。
12. 观察 `other` 占比和未知映射日志。

### 17.3 回填优先级

- 人工已标注 > 外部强类型 > 外部标签 > 名称关键词 > legacy type > other。
- 如果多个规则冲突,优先 confidence 高者。
- 如果 confidence 相同,优先 source priority 高者。
- 如果仍冲突,保留主分类,其他写 secondary。

### 17.4 legacy 映射建议

| old `models.type` | new `primary_model_type_code` | `catalog_type` | confidence |
|---|---|---|---|
| `checkpoint` | `checkpoint` | `image` | 0.90 |
| `lora` | `lora` | `image` | 0.90 |
| `template` | `template` | `image` | 0.90 |
| `workflow` | `other` | `flows` | 0.30 |
| `video` | `other` | `video` | 0.30 |

### 17.5 关键词映射建议

| 匹配文本 | new type | confidence |
|---|---|---|
| `LyCORIS` / `LoCon` / `LoHa` | `lycoris` | 0.80 |
| `Textual Inversion` / `Embedding` | `textual_inversion` | 0.80 |
| `Hypernetwork` | `hypernetwork` | 0.80 |
| `Aesthetic Gradient` | `aesthetic_gradient` | 0.80 |
| `ControlNet` / `Controlnet` | `controlnet` | 0.80 |
| `OpenPose` / `Pose` / `姿态` | `poses` | 0.70 |
| `Wildcard` / `通配符` | `wildcards` | 0.75 |
| `模板` / `Template` | `template` | 0.80 |

### 17.6 一致性校验

```sql
-- 每个模型必须有主分类字段
SELECT COUNT(*) FROM models
WHERE primary_model_type_code IS NULL
   OR primary_model_type_code = '';

-- 每个模型必须有 primary assignment
SELECT COUNT(*)
FROM models m
LEFT JOIN model_type_assignments a
  ON a.model_id = m.id AND a.is_primary = 1
WHERE a.id IS NULL;

-- 主表快照必须与 primary assignment 一致
SELECT m.id, m.primary_model_type_code, a.model_type_code
FROM models m
JOIN model_type_assignments a
  ON a.model_id = m.id AND a.is_primary = 1
WHERE m.primary_model_type_code <> a.model_type_code;

-- unknown code 不允许存在
SELECT DISTINCT m.primary_model_type_code
FROM models m
LEFT JOIN model_types mt ON mt.code = m.primary_model_type_code
WHERE mt.code IS NULL;
```

### 17.7 回滚策略

- 因为 v1 是 additive,回滚 server/client 不需要回滚 DB。
- 如果 migration 出问题,down 删除新表/列。
- 如果分类错误,修映射后重跑 backfill。
- 不要回滚 MinIO。
- 不要改历史 asset key。

---

## 18. 查询与索引设计

### 18.1 列表默认查询

- 默认条件:
- `m.status = 'approved'`
- `m.catalog_type = 'image'` 或不传则全部。
- `ORDER BY m.created_at DESC, m.id DESC`
- 使用游标分页。

### 18.2 主分类筛选

```sql
WHERE m.primary_model_type_code = ?
  AND m.status = 'approved'
ORDER BY m.created_at DESC, m.id DESC
```

对应索引:

```sql
idx_models_primary_type_created
```

### 18.3 任意分类筛选

```sql
WHERE m.status = 'approved'
  AND (
    m.primary_model_type_code IN (...)
    OR EXISTS (
      SELECT 1
      FROM model_type_assignments a
      WHERE a.model_id = m.id
        AND a.model_type_code IN (...)
    )
  )
```

对应索引:

```sql
idx_mta_type_model (model_type_code, model_id)
```

### 18.4 多选语义

- `modelType=a&modelType=b` 默认 OR。
- OR 更符合用户筛选“显示这些类型”。
- 若未来需要 AND,新增 `modelTypeMode=all`。
- MVP 不做 AND。

### 18.5 排序索引

- `latest`: `idx_models_status_created` 已覆盖。
- `popular`: 需要 join `model_counters.heat`。
- `mostUsed`: 需要 join `model_counters.run_count`。
- `mostImages`: 需要 join `model_counters.image_count`。
- 如果排序高频,可在 `model_counters` 增加:

```sql
KEY idx_mc_run_count (tenant_id, run_count DESC, model_id),
KEY idx_mc_image_count (tenant_id, image_count DESC, model_id),
KEY idx_mc_heat (tenant_id, heat DESC, model_id)
```

### 18.6 深翻页稳定性

- 当前 cursor 是 `created_at + id`。
- 分类筛选不改变 cursor 结构。
- 排序扩展时 cursor 必须包含排序字段。
- 例如 `run_count` 排序 cursor 需要 `run_count|created_at|id`。
- MVP 保持 `latest` 最安全。

---

## 19. 非功能要求

### 19.1 延迟目标

- `GET /models` p95 目标小于 300ms。
- `GET /models/{id}` p95 目标小于 500ms。
- `GET /model-types` p95 目标小于 100ms。
- 这些接口不应因 MinIO 单对象延迟而抖动。

### 19.2 缓存策略

- `GET /model-types`: `Cache-Control: public, max-age=300` 或 ETag。
- `GET /models`: 可加 `Cache-Control: private, max-age=30` 或 ETag,取决于租户隔离。
- `GET /models/{id}`: 可加 ETag,ETag 基于 `models.updated_at` + details updated。
- `/assets/{id}/content`: 继续走 ETag / Cache-Control。

### 19.3 租户隔离

- 当前契约强调 `Authorization + X-Tenant-Id`。
- 模型目录是否租户隔离需要统一。
- 如果未来多租户模型不同,`model_type_assignments` 也要带 tenant_id 或通过 `models.tenant_id` 间接隔离。
- 当前 `models` 表没有 tenant_id。
- 现状 liblib seed 使用 `model_counters.tenant_id`。
- 本次不强行引入 tenant_id,但要在方案风险中标记。

### 19.4 可观测性

- 记录未知外部 type。
- 记录归入 `other` 的数量。
- 指标:
- `model_type_backfill_total`
- `model_type_other_total`
- `model_type_unknown_external_total`
- `model_type_filter_query_ms`
- `model_type_assignment_mismatch_total`

### 19.5 数据质量

- `other` 占比超过阈值要报警。
- 新导入模型如果 `model_type_confidence < 0.5`,写入审查队列。
- admin 修改主分类时写 audit log。
- 低置信度数据不应静默进入高曝光推荐。

---

## 20. MVP 边界

### 20.1 MVP 必做

- 新增 `modelType` enum。
- 新增 `model_types` 表并 seed 11 类。
- 新增 `models.primary_model_type_code`。
- 新增 `model_type_assignments`。
- 回填存量数据。
- `GET /models` 返回 `modelType` 和 `modelTypeLabel`。
- `GET /models/{id}` 返回 `modelType`、`modelTypeLabel`、`primaryModelType`。
- client 详情页展示新分类。
- client 列表卡片使用新分类 badge。
- importers 写新字段。
- 保留 v1 legacy `type`。

### 20.2 MVP 可选

- `GET /model-types`。
- `modelTypes[]` 全量返回。
- 列表多选筛选 UI。
- 审计表。
- source mappings DB 化。

### 20.3 MVP 不做

- 不删除 `models.type`。
- 不发布 `/api/v2`。
- 不重写模型上传全流程。
- 不迁移 MinIO object key。
- 不做复杂 AND 筛选。
- 不做排序扩展。
- 不做 admin 分类管理后台。

### 20.4 Phase 2

- 完成 `/model-types`。
- 列表页 11 类筛选 chips。
- 多选筛选。
- `Other` 数据治理报表。
- `model_type_source_mappings` 管理。
- 排序扩展。
- 运行时缓存。

### 20.5 Phase 3

- `/api/v2` 清理 `type`。
- 引入 `resourceKind`。
- 引入 admin 手动纠错。
- 引入模型上传时分类校验。
- 引入生成链路对 `modelType` 的能力判断。

---

## 21. 具体文件落点

### 21.1 文档与契约

- `eye-trace-docs/openapi-skeleton.md`
- `eye-trace-docs/openapi.yaml`
- `eye-trace-docs/model-summary-card-flow.md`
- `eye-trace-docs/model-detail-flow.md`
- `eye-trace-docs/client_data_split.md`
- `eye-trace-docs/model-type-redesign-implementation-plan.md`

### 21.2 共享枚举

- `eye-trace-config/shared/enums.yaml`
- 后续若导出器落地:
- `eye-trace-config/schema/enums.schema.json`
- `eye-trace-config/exporters/to-client.ts`
- `eye-trace-config/exporters/to-server.ts`
- `eye-trace-config/dist/client/config.gen.ts`
- `eye-trace-config/dist/server/config_gen.go`

### 21.3 Server migration

- `eye-trace-server/backend/migrations/000017_model_types.up.sql`
- `eye-trace-server/backend/migrations/000017_model_types.down.sql`

### 21.4 Server model 层

- `eye-trace-server/backend/internal/model/repo.go`
- `eye-trace-server/backend/internal/model/service.go`
- `eye-trace-server/backend/internal/model/service_test.go`
- 可新增 `eye-trace-server/backend/internal/model/types.go`
- 可新增 `eye-trace-server/backend/internal/model/type_repo.go`

### 21.5 Server handler

- `eye-trace-server/backend/internal/api/handler/model.go`
- 可新增 `eye-trace-server/backend/internal/api/handler/model_type.go`
- `eye-trace-server/backend/internal/api/api.gen.go` 由 oapi-codegen 生成。
- `eye-trace-server/backend/cmd/server/main.go` 注册新 endpoint。

### 21.6 Server importer

- `eye-trace-server/backend/cmd/importer/liblib-deep/models.go`
- `eye-trace-server/backend/cmd/importer/liblib-deep/details.go`
- `eye-trace-server/backend/cmd/importer/liblib/helpers.go`
- `eye-trace-server/backend/cmd/importer/liblib/importer.go`
- 可新增 `eye-trace-server/backend/internal/modeltype/resolver.go` 或 importer 内部 helper。

### 21.7 Client API

- `eye-trace-client/src/api/schema.gen.ts`
- `eye-trace-client/src/api/models.ts`
- `eye-trace-client/src/api/schemas.ts`
- `eye-trace-client/src/api/__tests__/models.test.ts`

### 21.8 Client UI

- `eye-trace-client/src/app/routes/ModelDetailPage.tsx`
- `eye-trace-client/src/app/routes/ImageModelListPage.tsx`
- `eye-trace-client/src/features/models/useModelList.ts`
- `eye-trace-client/src/components/layout/ModelItemList.tsx`
- `eye-trace-client/src/config/static/types.ts`

---

## 22. 实施顺序

### 22.1 Step 1:契约先行

- 改 `openapi-skeleton.md`。
- 改 `openapi.yaml`。
- 加 `ModelType` 11 类。
- 加 `ModelTypeInfo`。
- 加 `ModelClassification`。
- 给 `ModelSummary` 和 `ModelDetail` 加新字段。
- 给 `GET /models` 加 `modelType` 和 `primaryModelType` 参数。
- 可加 `GET /model-types`。

### 22.2 Step 2:共享枚举

- 改 `eye-trace-config/shared/enums.yaml`。
- 新增 `modelType`。
- 保持 `typeTabIds` 不动。
- 在注释中明确两者边界。

### 22.3 Step 3:DB migration

- 新增 migration。
- 创建表。
- seed 字典。
- add columns。
- backfill。
- consistency check。

### 22.4 Step 4:Server types 生成

- 运行 oapi-codegen。
- 更新 `internal/api/api.gen.go`。
- 编译解决 handler 接口变化。

### 22.5 Step 5:Server repo/service/handler

- 扩 `ListQuery`。
- 扩 SQL。
- 扩 `ModelRow`。
- 扩 `Summary`。
- 扩 `DetailFullResult`。
- 输出新字段。
- 加测试。

### 22.6 Step 6:Importer 双写

- 替换硬编码 switch。
- 写 `primary_model_type_code`。
- 写 `model_type_assignments`。
- 保留旧 `models.type`。
- log unknown。

### 22.7 Step 7:Client types 生成

- 运行 openapi-typescript。
- 更新 `schema.gen.ts`。
- 删除手写 `ModelType` union。
- 更新 zod schema 或移除。

### 22.8 Step 8:Client UI

- `summaryToCard` 使用 `modelTypeLabel`。
- 详情页类型行使用 `modelTypeLabel`。
- 列表页可先只显示,后加筛选。

### 22.9 Step 9:验证

- DB consistency SQL。
- Go tests。
- Client tests。
- 手动打开 `/modelinfo/{id}`。
- 手动打开模型列表。
- 验证 `Other`。
- 验证旧字段兼容。

---

## 23. 测试计划

### 23.1 DB 测试

- migration up 成功。
- migration down 成功。
- `model_types` 11 行存在。
- 每个 `models` 行有 `primary_model_type_code`。
- 每个 `models` 行有 primary assignment。
- `primary_model_type_code` 都能 join 到 `model_types`。
- 多 secondary assignment 不违反唯一约束。
- 两个 primary assignment 会被唯一约束拒绝。

### 23.2 Server 单元测试

- `normalizeQuery` 接受 11 类。
- `normalizeQuery` 拒绝未知类。
- `ListSummary` 返回 `modelType`。
- `GetDetailFull` 返回 `primaryModelType`。
- `ToSummary` 在 label 缺失时 fallback 到 code。
- legacy `type` 仍返回旧值。

### 23.3 Server 集成测试

- `GET /api/v1/model-types` 返回 11 类。
- `GET /api/v1/models?modelType=lora` 返回 LoRA 或 secondary 命中。
- `GET /api/v1/models?primaryModelType=lora` 只返回主分类 LoRA。
- `GET /api/v1/models?modelType=unknown` 返回 400。
- `GET /api/v1/models/{id}` 返回分类对象。

### 23.4 Client 单元测试

- `listModels` 正确拼接多个 `modelType`。
- `summaryToCard` 使用 `modelTypeLabel`。
- `summaryToCard` fallback 到旧 `type`。
- `ModelDetailPage` `modelTypeLabel` fallback 正确。

### 23.5 手动验收

- 打开 `/modelinfo/{id}`。
- 类型行展示 11 类之一。
- 标题下 badge 展示主分类。
- 列表卡片 badge 展示主分类。
- `Other` 展示在最后。
- 筛选 `LoRA` 后列表不重复。
- 刷新和翻页 cursor 稳定。

---

## 24. 风险与歧义

### 24.1 外部生态映射不稳定

- Liblib/Civitai 的 type code 可能不同。
- 同一个 name 在不同来源可能语义不同。
- 解决:使用 `model_type_source_mappings`。

### 24.2 LyCORIS 与 LoRA 层级

- LyCORIS 可以被看成 LoRA 家族。
- 也可以作为独立分类。
- 需求明确列出 LyCORIS,所以主分类必须独立。
- 是否在 LoRA 筛选中包含 LyCORIS 是产品选择。

### 24.3 ControlNet 与 Poses 边界

- Pose 资源经常用于 ControlNet。
- 但 Pose 本身可能是图片/JSON,不是 ControlNet 模型。
- 推荐保留两个分类。
- 可通过 parent 或 UI 聚合建立关系。

### 24.4 模板边界

- 模板不是纯模型权重。
- 但当前产品已有模板详情页和生成入口。
- 保留 `template` 有实际业务价值。
- 后续用 `resourceKind` 区分可运行性。

### 24.5 Workflow / Video 老数据

- 新 11 类不包含 workflow/video。
- 老 `models.type=workflow/video` 不能直接丢。
- 推荐转为 `catalog_type=flows/video` + `primary_model_type_code=other`。
- 若业务决定视频模型也是模型分类,应另开需求扩展 enum,不要混入本 11 类。

### 24.6 Other 泛滥

- 回填规则太保守会导致大量 `other`。
- 规则太激进会误分类。
- 解决:confidence + audit + 运营审查。

### 24.7 契约破坏

- 如果直接把 `type` 改成 11 类,旧客户端会错。
- v1 必须 additive。
- v2 才能破坏性清理。

### 24.8 查询性能

- 多分类 join 可能拖慢列表。
- 用 `primary_model_type_code` 支撑主路径。
- 任意分类用 `EXISTS`。
- 大规模数据再考虑 materialized filter table。

### 24.9 手写类型继续漂移

- client 已有手写 `ModelType`。
- 如果不清理,生成类型也没意义。
- 本次必须把模型分类相关手写 union 合并到 generated type。

---

## 25. 关键决策记录

### 25.1 分类 code 使用英文 snake_case

- 原因:跨端稳定,适合 URL/query/DB。
- 中文只作为 label。

### 25.2 DB assignment 是分类权威

- 原因:server 是状态权威。
- MinIO 和 client 都不是分类权威。

### 25.3 保留 legacy `models.type`

- 原因:v1 兼容。
- 但新逻辑不再依赖它。

### 25.4 支持多分类

- 原因:SD 生态边界天然重叠。
- 主分类解决 UI 展示。
- 附加分类解决筛选召回。

### 25.5 MVP 不做 v2

- 原因:当前前后端已有生成链路和旧字段,直接 v2 会扩大范围。
- additive 更稳。

---

## 26. 粗略工作量评估

### 26.1 后端

- migration + seed:0.5 天。
- repo/service/handler:1 到 1.5 天。
- importer 双写:0.5 到 1 天。
- tests:0.5 天。
- 合计:2.5 到 3.5 天。

### 26.2 前端

- types regen + API 层:0.5 天。
- 详情页展示:0.25 天。
- 列表 badge:0.25 天。
- 筛选 UI MVP:0.5 到 1 天。
- tests:0.25 天。
- 合计:1.5 到 2.5 天。

### 26.3 契约与文档

- OpenAPI + docs:0.5 到 1 天。

### 26.4 总计

- MVP:4 到 6 人天。
- 包含筛选 UI 和 importer 稳定化:5 到 8 人天。
- 包含 admin 治理与 v2:另算。

---

## 27. 推荐落地版本切片

### 27.1 Slice A:只展示分类

- DB 新字段。
- 回填。
- `/models/{id}` 返回 `modelTypeLabel`。
- 详情页展示。
- 不做筛选 UI。
- 风险最低。

### 27.2 Slice B:列表返回分类

- `/models` 返回 `modelTypeLabel`。
- 瀑布卡 badge 使用新字段。
- old `typeTag` 保留。

### 27.3 Slice C:筛选

- `GET /models?modelType=...`。
- 列表页 11 类 chips。
- `Other` 放最后。

### 27.4 Slice D:导入器治理

- source mappings。
- unknown logs。
- low confidence audit。

---

## 28. 最终推荐

- 采用“字典表 + assignment 表 + 主表主分类快照”的混合设计。
- 将 11 类定义为新 `modelType`,不要复用 `typeTabIds`。
- `/api/v1` 增加 `modelType` 字段,保留 legacy `type`。
- `/api/v2` 再删除 legacy `type`。
- 分类 assignment 归 DB 管。
- 允许多分类,但每个模型必须只有一个主分类。
- MinIO object key 不包含分类。
- client 只消费 OpenAPI generated types。
- importer 从映射表/统一 resolver 写分类,不继续散落 switch。

---

## 29. 下一步可执行清单

- [ ] 修改 `eye-trace-docs/openapi-skeleton.md`。
- [ ] 修改 `eye-trace-docs/openapi.yaml`。
- [ ] 修改 `eye-trace-config/shared/enums.yaml`。
- [ ] 新增 `000017_model_types.up.sql`。
- [ ] 新增 `000017_model_types.down.sql`。
- [ ] 更新 server `ListQuery`。
- [ ] 更新 server `ListModels` SQL。
- [ ] 更新 server `CountModels` SQL。
- [ ] 更新 server `GetModelDetailFull`。
- [ ] 更新 server `Summary`。
- [ ] 更新 server `DetailFullResult`。
- [ ] 新增 `/model-types` handler。
- [ ] 更新 liblib importers。
- [ ] 运行 oapi-codegen。
- [ ] 运行 openapi-typescript。
- [ ] 删除 client 手写 `ModelType` union。
- [ ] 更新 client `summaryToCard`。
- [ ] 更新 `ModelDetailPage` 类型展示。
- [ ] 更新 `useModelList` 支持 `modelType` 参数。
- [ ] 补 Go tests。
- [ ] 补 Vitest tests。
- [ ] 手动验收 `/modelinfo/{id}`。

