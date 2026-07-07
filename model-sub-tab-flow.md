# models.sub_tab_id — 模型 sub_tab 归属字段

> 目的:为"models 加 `sub_tab_id` 字段"这件事给出完整契约/数据流说明,作为后续 server / client / importer / seed 改动的唯一真相源。
>
> 涉及子仓库:`eye-trace-docs`、`eye-trace-server`、`eye-trace-client`
> 上位文档:[`project_guide.md` §7](./project_guide.md)(DB 表设计)、[`openapi-skeleton.md` §2 §3](./openapi-skeleton.md)(端点与 Schema)
> 平行文档:[`home-type-tabs-sub-tabs-flow.md`](./home-type-tabs-sub-tabs-flow.md)(`home_type_tabs` / `home_sub_tabs` 配置表);[`model-summary-card-flow.md`](./model-summary-card-flow.md)(`GET /api/v1/models` 列表契约)

---

## 1. 背景与目标

- `home_type_tabs` / `home_sub_tabs` 是首页 tab 配置表,sub_tab id 嵌在 `home_sub_tabs.by_type_json` JSON 里(详见 `home-type-tabs-sub-tabs-flow.md`)。
- 首页"图片模型"标签下的模型瀑布列表需要按 sub_tab 归属过滤 / 切显(用户点 sub_tab 后,瀑布只展示归属该 sub_tab 的 model)。
- 改前必须先给每条 `models` 行打"它属于哪个 sub_tab"的归属标签,server 才能按此 WHERE 过滤、client 才能在渲染时知道每张卡的归属。
- 范围:**本期只做 1:N(一条 model 挂一个 sub_tab)**;N:N 多归属留待 v2,DB 设计不预留中间表。

## 2. 数据流(端到端)

```
client(瀑布 UI + sub_tab 切换)
   │
   │  GET /api/v1/models?subTabId=image-all          ← 用户点 sub_tab 后触发
   │  Authorization: Bearer <jwt>
   │  X-Tenant-Id: <tenant>
   ▼
server handler (handler/models.go)
   │  1) 解析 ?subTabId=image-all
   │     正则校验: ^[\w-]+-[\w-]+$  且 长度 ≤ 64
   │     不合法 → 400 VALIDATION_FAILED
   │  2) 调 service.ListModels(ctx, &ListModelsParams{SubTabID: &subTabId})
   ▼
repo (internal/model/repo.go)
   │  SELECT ... FROM models
   │  WHERE (? == nil OR sub_tab_id = ?)            ← nil 表示不限定
   │  ORDER BY id DESC LIMIT N
   ▼
MySQL models 表 (models.sub_tab_id VARCHAR(64) NULL)
   │
   │  rows → scan → ModelRow
   ▼
service: ModelRow → ModelSummary(subTabId 字段注入)
   │
   ▼
handler: 200 application/json { items: ModelSummary[], nextCursor?: string }
   │
   ▼
client: summaryToCard(s) → ModelItemCard(渲染用),
        渲染时若 s.subTabId === 当前 activeSubTabId → 展示卡片(否则过滤掉)
```

正向流外还有两条副作用链(变更写):

- **importer / seed 写侧**:liblib importer 与 `migrations/seed_models.sql` 在写入 `models` 时,**必须**根据 `home_sub_tabs.by_type_json` 解析出归属 sub_tab,把 `sub_tab_id` 一起 INSERT/UPDATE。
- **admin-schema-viewer 只读侧**:保持不变,只读 SELECT 校验,不入 `/api/v1` 契约。

## 3. 契约与命名

### 3.1 新增字段:`ModelSummary.subTabId`

| 项 | 值 |
|---|---|
| OpenAPI 类型 | `{ type: string, nullable: true }` |
| 含义 | 该 model 归属的 sub_tab(格式见 §3.3);`null` 表示不属于任何 sub_tab |
| 默认下发 | 永远下发字段;值可空(`null` = 不归属,client 默认归到"全部") |
| 读取方式 | client 直接消费 `s.subTabId`,无需 `summaryToCard` 翻译 |

### 3.2 新增查询参数:`GetModelsParams.subTabId`

| 项 | 值 |
|---|---|
| OpenAPI 位置 | `parameters` (GET /models) |
| 类型 | `{ type: string, nullable: true, maxLength: 64 }` |
| 含义 | 仅返回 `models.sub_tab_id = ?` 的项;不传或 null = 全量下发(向后兼容) |
| 校验 | handler 处正则校验 `^[\w-]+-[\w-]+$`,非合法 → `400 VALIDATION_FAILED` |

### 3.3 `sub_tab_id` 命名约定

```
sub_tab_id = "{type_tab_id}-{sub_tab_id}"
```

- `type_tab_id` ∈ `home_type_tabs.tabs_json[].id`(目前为 `image` / `video` / `ideas`)
- `sub_tab_id` ∈ `home_sub_tabs.by_type_json[type_tab_id][].id`(如 `all` / `hot` / `trending`)
- 合法值示例:`image-all`、`image-hot`、`video-hot`、`ideas-trending`
- **DB 端不建外键**:`home_sub_tabs.by_type_json` 是 JSON 列,无法强引用;靠运营 / seed 阶段保证一致性
- **校验入口**:server handler 解析 subTabId 时校验正则 + 长度;**不查 `home_sub_tabs` 实时匹配**(运行时配置变更频繁,过度校验会引入 stale cache 问题)

## 4. DB schema

### 4.1 新列

```sql
ALTER TABLE models
  ADD COLUMN sub_tab_id VARCHAR(64) NULL,
  ADD INDEX idx_sub_tab (sub_tab_id);
```

- 类型:VARCHAR(64),与 `base_model` 列同构
- 可空:NULL = 不归属任何 sub_tab
- 索引:KEY `idx_sub_tab`(`sub_tab_id`) — 加速瀑布过滤
- 不建外键:运营自由,JSON 列无法强引用

### 4.2 migration 文件

```
eye-trace-server/backend/migrations/000017_models_add_sub_tab_id.up.sql
eye-trace-server/backend/migrations/000017_models_add_sub_tab_id.down.sql
```

> 序号沿用现有 `000001_…` ~ `000015_…` 风格;若发布期已有 `000016_*` 占位,顺延递增即可。

## 5. 改动清单(PR checklist)

精确到文件级,任何缺漏应在本表登记后再补:

| 子仓 | 路径 | 动作 |
|---|---|---|
| `eye-trace-docs/openapi.yaml` | `ModelSummary` + `GetModelsParams` | 新增 `subTabId` 字段/参数(§3.1 §3.2) |
| `eye-trace-server/backend/migrations/000017_models_add_sub_tab_id.up.sql` | 新建 | 见 §4.1 |
| `eye-trace-server/backend/migrations/000017_models_add_sub_tab_id.down.sql` | 新建 | `DROP INDEX idx_sub_tab; ALTER TABLE models DROP COLUMN sub_tab_id;` |
| `eye-trace-server/backend/internal/api/api.gen.go` | 重新生成 | `cd backend && make gen`(从 `schema/public_api.yaml` 读;记得同时把 `eye-trace-docs/openapi.yaml` 同步到 `backend/schema/public_api.yaml` 后再 gen) |
| `eye-trace-server/backend/internal/model/repo.go` | 修改 | `ListModels` SQL 加 `WHERE (? == nil OR sub_tab_id = ?)`,在 `ModelRow` 结构增 `SubTabID sql.NullString` |
| `eye-trace-server/backend/internal/model/handler.go` | 修改 | 解析 `subTabId` 参数、正则校验、注入到 service params(handler 仍 ≤ 30 行) |
| `eye-trace-server/backend/internal/model/service.go` | 修改 | `ModelRow → ModelSummary` 翻译时透传 `SubTabID`(NULL 仍写 `null`,不写空串) |
| `eye-trace-server/backend/cmd/importer/liblib/importer.go` | 修改 | 写 `models` 行时,根据 `home_sub_tabs.by_type_json` 解析 sub_tab 归属并 set `sub_tab_id` |
| `eye-trace-server/backend/cmd/importer/liblib-deep/models.go` | 修改 | 同上,liblib-deep 导入路径独立处理(可能直接由 liblib 解析后的 tag 派生) |
| `eye-trace-server/backend/migrations/seed_models.sql` | 修改 | 已存在(Makefile `seed` 目标已引用);在 INSERT 列清单中加 `sub_tab_id`,UPDATE 路径同步 |
| `eye-trace-client/src/api/schema.gen.ts` | 重新生成 | `npm run gen:api`(从 `eye-trace-docs/openapi.yaml`) |
| `eye-trace-client/src/api/models.ts` | 修改 | `summaryToCard` 透传 `subTabId`(client 内部 `ModelItemCard` 也增同名可选字段) |
| `eye-trace-client/src/features/(model list 相关 store/view)` | 修改 | 瀑布组件增加 sub_tab 切换触发 `GET /models?subTabId=`;渲染时按 `s.subTabId` 与当前 active 过滤;空态组件复用(已有) |

**不动**:

- `admin-schema-viewer`:`models` 表天然出现 `sub_tab_id` 列(只读契约),**不**加可视化编辑入口(保持只读)
- `home_type_tabs` / `home_sub_tabs` 配置表(JSON 列):见 §6

## 6. 不在本期范围

- **N:N 多归属**:一条 model 挂多个 sub_tab(留待 v2,可建中间表 `model_sub_tab_bindings` 或加 JSON 列)
- **admin 后台可视化编辑 sub_tab 归属**:本期只通过 importer / seed / 手改 SQL 维护,不动 admin 后台
- **给 `sub_tabs` 单独建表**:继续用 JSON 列(`home_sub_tabs.by_type_json`),理由见 `000005_home_tabs.up.sql` 注释
- **客户端 sub_tab 切换动效 / 历史筛选记录**:本期只做"过滤行为可用",UX 不优化

## 7. 验证

1. **DB 结构核对**:起 `admin-schema-viewer`(`make admin`),选 `models` 表 → 看到 `sub_tab_id` 列 = `VARCHAR(64) NULL`,索引 `idx_sub_tab` 存在。
2. **过滤接口**:
   ```bash
   curl -H "Authorization: Bearer $T" -H "X-Tenant-Id: $TENANT" \
        "$BASE/api/v1/models?subTabId=image-all" | jq '.items[].subTabId'
   # 所有返回的 subTabId 都应是 "image-all" 或 null(数据问题)
   ```
3. **向后兼容**:
   ```bash
   # 不传 subTabId 时,行为不变:全量下发
   curl -H "Authorization: Bearer $T" -H "X-Tenant-Id: $TENANT" \
        "$BASE/api/v1/models" | jq '.items | length'
   ```
4. **校验入参**:传 `subTabId=invalid` 应得 `400 VALIDATION_FAILED`;传 `subTabId=image_all`(下划线)同样 400。
5. **client 渲染**:首页/瀑布组件切换 sub_tab → 仅展示归属卡;无数据时显示已有空态组件。

## 8. 风险与回滚

### 8.1 风险

| 风险 | 缓解 |
|---|---|
| 老 `models` 行 `sub_tab_id` 全 NULL,等价于"无归属",瀑布过滤可能"全空" | seed / importer 在上线前把历史行根据 `home_sub_tabs.by_type_json` 解析后回填;`sub_tab_id IS NULL` 仍归"全部"展示,不丢老数据 |
| `sub_tab_id` 与 `home_sub_tabs` 配置漂移(运营改了 sub_tab id 后老 model 失效) | DB 不强约束,只能靠运营约束 + 定期 reconcile 脚本;v2 N:N 化前不算严重 |
| 客户端解析旧响应无 `subTabId` 字段(rolling upgrade 期) | client `summaryToCard` 对 `s.subTabId === undefined` 视同 `null`,降级"无归属" |

### 8.2 回滚

- **DB**:`down.sql` 删 `idx_sub_tab` + `sub_tab_id` 列(无约束可逆,无数据丢失 —— 字段已 NULL,删列是无损 down)
- **契约回滚**:`openapi.yaml` 删 `ModelSummary.subTabId` + `GetModelsParams.subTabId`;同步 `backend/schema/public_api.yaml`;重跑 `make gen`(server)+ `npm run gen:api`(client)
- **行为回滚**:不传 `subTabId` 即与回滚后等价(全量下发路径从未改变)
- **客户端回滚**:瀑布组件回到不按 sub_tab 过滤的旧渲染(commit revert 即可)
