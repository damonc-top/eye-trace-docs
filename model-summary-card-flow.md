# Model Summary Card 数据格式 — 现状

> 时间:2026-06-27(更新)
> 范围:`GET /api/v1/models` 列表响应 + 瀑布卡 `ModelSummary` 字段 + client 端 `summaryToCard` 映射
> 涉及子仓库:`eye-trace-docs`、`eye-trace-server`、`eye-trace-client`
> 接续:[model-detail-flow.md](./model-detail-flow.md) — 详情接口消费同样的 DB 表但字段更宽

---

## 1. 当前契约(2026-06-27 实装版)

### 1.1 `ModelSummary` schema(openapi.yaml)

```yaml
ModelSummary:
  required: [modelId, title, mediaType, cover]
  properties:
    modelId:       { type: string }                    # 字符串化的 models.id
    title:         { type: string }                    # models.name
    author:        { type: string }                    # author_profile.nickname 或 author_name(flat)
    authorAvatar:  { type: string, format: uri }       # 头像 URL(走 /api/v1/assets 或 MinIO 直连)
    cover:         { type: string, format: uri }       # 封面 URL
    typeTag:       { type: string }                    # models.type_tag 派生(uppercase)
    mediaType:     { $ref: '#/components/schemas/MediaType' }
    type:          { $ref: '#/components/schemas/ModelType' }  # 5 枚举:checkpoint / lora / template / workflow / video
    useCount:      { type: integer, minimum: 0, nullable: true }   # = liblib counter.runCount
    imageCount:    { type: integer, minimum: 0, nullable: true }   # = liblib counter.imageCount
    viewId:        { type: string, nullable: true }                # client 跳详情用,默认 "model.detail"
```

### 1.2 与 `ModelItemCard`(client 内部) 的差异

`ModelSummary` 是 server → client 协议;`ModelItemCard` 是 client 内部消费类型(`@/config/static/types.ts`)。

client 端 `summaryToCard(s: ModelSummary): ModelItemCard` 在 `src/api/models.ts` 做翻译,关键映射:

| ModelSummary | ModelItemCard |
|---|---|
| `modelId: string` | `id: number`(Number(s.modelId)) |
| `title` | `title` |
| `cover` | `cover` |
| `mediaType` | `mediaType`(只映射 image 枚举) |
| `author` | `author` |
| `authorAvatar` | `authorAvatar` |
| `typeTag ?? type` | `typeTag`(模板类型优先于 tag 标签) |
| `viewId ?? "model.detail"` | `viewId` |
| `params: { id }` | `params` |
| `actionLabel: "去WebUI使用"` | `actionLabel`(写死,本期不读 server) |

ModelItemCard 还有 `previews[]` / `gallery[]` / `variant` 字段;summary 路径下全部是 `variant: "single"`(本期 server 端 `preview-deck` / `gallery` 多图 variant 还没接入,留 Phase 2)。

---

## 2. server 实现

### 2.1 SQL 查询(单 trip,5 表 JOIN)

`internal/model/repo.go::ListModels`:

```sql
SELECT m.id, m.name, m.type, m.cover_asset_id, m.preview_asset_ids, m.gallery,
       m.author_name, m.author_avatar_id,
       m.type_tag, m.type_tag_suffix, m.exclusive, m.action_label, m.view_id, m.external_href,
       m.created_at,
       a.media_type, a.storage_key,                     -- cover media type + MinIO key
       aa.storage_key,                                   -- author avatar MinIO key
       mc.run_count, mc.image_count                       -- counter 数字栏
FROM models m
LEFT JOIN assets a        ON a.id  = m.cover_asset_id
LEFT JOIN assets aa       ON aa.id = m.author_avatar_id
LEFT JOIN model_counters mc ON mc.model_id = m.id
WHERE m.status = 'approved'
  [AND m.type = ?]            -- q.Type
  [AND m.base_model = ?]      -- q.BaseModel
  [AND JSON_CONTAINS(...)]    -- q.Tags AND
  [AND MATCH(m.name) AGAINST(? IN BOOLEAN MODE)]  -- q.Search
  [AND cursor predicate]      -- q.Cursor
ORDER BY m.created_at DESC, m.id DESC
LIMIT ?
```

### 2.2 handler 派生 — `service.ToSummary`

`internal/model/service.go::ToSummary` 把 `ModelRow` 拍扁成 `Summary`:

| 派生逻辑 | 字段 |
|---|---|
| `r.ID` | `ModelID = strconv.FormatInt(r.ID, 10)` |
| `r.Name` | `Title` |
| `r.CoverAssetID` → `s.assets.URL(*id)` | `Cover` |
| `r.CoverMediaType` | `MediaType`(空时 fallback "image") |
| `r.Type` | `Type`(5 枚举,client 跳详情用) |
| `r.AuthorName` | `Author`(`*string`) |
| `r.AuthorAvatarKey` → `s.assets.DirectURL(key)` 或 `r.AuthorAvatarID` → `s.assets.URL(*id)` | `AuthorAvatar` |
| `r.TypeTag` | `TypeTag` |
| `r.ViewID` | `ViewID`(default "model.detail") |
| `r.UseCount`(从 model_counters JOIN) | `UseCount` |
| `r.ImageCount`(同上) | `ImageCount` |

### 2.3 handler 入口 — `handler.GetModels`

`internal/api/handler/model.go::GetModels`:

1. 解析 `api.GetModelsParams` → `model.ListQuery`
2. 调 `service.ListSummary`(不走 `service.List + ToCard` 路径,ListSummary 直接返 `[]Summary`)
3. marshal JSON: `{items: [Summary], nextCursor, totalHint}`

```go
// service.ListSummary 与 service.List 同源,两条独立路径:
//
// service.List        → ToCard  → 瀑布卡 multi-variant(Phase 2 接 preview-deck / gallery)
// service.ListSummary → ToSummary → GET /models 单条 / 列表
```

### 2.4 字段数据来源(本期样本 model=15)

| ModelSummary 字段 | 来源 SQL 列 | 实测值 |
|---|---|---|
| `modelId` | `m.id` | "15" |
| `title` | `m.name` | "动漫游戏CG质感角色V2｜次时代3D漫剧" |
| `author` | `au.nickname`(JOIN authors)| "Dave" |
| `authorAvatar` | `aa.storage_key` → DirectURL | "http://127.0.0.1:9000/eye-trace-assets/models/0/author_avatar/avatar/ea845bc...jpg" |
| `cover` | `m.cover_asset_id` → `assets.URL(id)` | "http://127.0.0.1:8080/api/v1/assets/106/content" |
| `type` | `m.type` | "template" |
| `typeTag` | `m.type_tag`(本期 NULL → 派生空) | — |
| `mediaType` | `a.media_type` | "image" |
| `useCount` | `mc.run_count` | 22800 |
| `imageCount` | `mc.image_count` | 389 |
| `viewId` | `m.view_id`(本期 NULL → default "model.detail") | "model.detail" |

---

## 3. client 实现

### 3.1 `src/api/models.ts::listModels`

```ts
export async function listModels(params: ListModelsParams = {}): Promise<ModelListResponse> {
  // 1) 拼 query:type, baseModel, search, cursor, limit, tag[], sort=latest
  // 2) apiClient.get<ModelListResponse>('/models?...')
  // 3) 不再走 zod 解析(老 schema 已与 server 不匹配)
  return data;
}
```

### 3.2 `src/api/models.ts::summaryToCard`

`ModelSummary[]` → `ModelItemCard[]`,喂给 `ModelItemList` 瀑布卡:

```ts
export function summaryToCard(s: ModelSummary): ModelItemCard {
  return {
    id: Number(s.modelId),
    title: s.title,
    cover: s.cover,
    mediaType: s.mediaType === "image" ? "image" : "image",
    author: s.author,
    authorAvatar: s.authorAvatar,
    typeTag: s.typeTag ?? s.type,
    exclusive: false,
    actionLabel: "去WebUI使用",
    viewId: s.viewId ?? "model.detail",
    params: { id: s.modelId },
  };
}
```

### 3.3 `src/features/models/useModelList.ts`

无限滚动 + 触底加载:

- 首次 mount 自动 load 24 / 24 (per viewport)
- 触底 sentinel:监听 AppShell `<main>` overflow:auto
- 已加载 id Set 防重
- StrictMode 防双调用,KeepAlive 切走再回来会重拉

### 3.4 路由

| ViewId | Path | 组件 |
|---|---|---|
| `image.model.list` | `/image-model` | `ImageModelListPage`(接 `useModelList`) |
| `model.detail` | `/modelinfo/:id` | `ModelDetailPage`(接 `getModelById`,见 [model-detail-flow.md](./model-detail-flow.md)) |

### 3.5 客户端瀑布卡渲染(对应 `summaryToCard` 输出)

`ModelItemList` 渲染流程:

1. `<Card>` 接受 `ModelItemCard`
2. variant 默认 "single",封面 1:1 渲染
3. 底部 author overlay:头像 + 名字
4. 标题截断 h-10
5. click → `nav.openView(viewId, params)` → 跳 `/modelinfo/:id`

---

## 4. 端到端验证(2026-06-27)

```
GET /api/v1/models?limit=1
→ {"items":[{"modelId":"15","title":"动漫游戏CG质感角色V2｜次时代3D漫剧",
             "author":"Dave","authorAvatar":"http://127.0.0.1:9000/eye-trace-assets/...",
             "cover":"http://127.0.0.1:8080/api/v1/assets/106/content",
             "mediaType":"image","type":"template",
             "useCount":22800,"imageCount":389}],
   "nextCursor":"MTc4MjU3MDc4NzAwMHwxNQ","totalHint":1}
```

client 瀑布卡渲染:封面 / 标题 / 作者 / 标签(template)/ CTA(去WebUI使用)/ 头像 全部加载,见 `eye-trace-client/waterfall.png`。

---

## 5. 待办(留作 Phase 2)

- [ ] `ModelItemCard.variant: "preview-deck"` / `"gallery"` — server 端 `model_previews` 翻页 + JSON 列
- [ ] `ModelSummary.typeTag` 从 `models.type_tag` 列填值(本期 importer 不写)
- [ ] `ModelSummary.viewId` 区分 `view: "model.detail"` / `view: "model.run"`(直跳生成)
- [ ] typeTab / subTab 过滤(参 model-summary-card-flow.md 原 §2 / §3,本期未做 SQL 子句)
- [ ] `ModelSummary.actionLabel` 走 server 端 models.action_label(本期写死 "去WebUI使用")
