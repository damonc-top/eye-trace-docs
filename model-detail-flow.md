# Model Detail 数据设计与迭代依据

> 更新时间: 2026-06-30  
> 范围: 模型详情页 `/modelinfo/:id` 的 detail、author、comments、作品展示数据链路  
> 涉及仓库: `eye-trace-server`、`eye-trace-client`、`eye-trace-docs`  
> 参考样本: `tmp/动漫游戏cg质感-detail.json`、`tmp/动漫游戏cg质感-author.json`、`tmp/动漫游戏cg质感-comments-list.json`、`tmp/动漫游戏cg质感-return-pic.json`

这份文档记录当前已落地的生产级数据设计。后续迭代不要再从页面临摹反推字段,而是按这里的边界改 DB、导入器、API 和 UI。

---

## 1. 数据链路总览

| 页面区域 | 主要数据源 JSON | 落库表 | API 输出 | 前端消费 |
|---|---|---|---|---|
| 顶部标题、封面、版本、标签、描述、参数、计数 | `*-detail.json` | `models`、`model_details`、`model_versions`、`model_cover_images`、`model_counters`、`assets` | `GET /api/v1/models/{modelId}` | `ModelDetailPage` |
| 作者卡 | `*-author.json` + detail 的 `userUuid` | `authors`、`assets`、`models.author_user_uuid` | `author`、`authorAvatar`、`authorStats` | 右侧 `AuthorCard` |
| 评论讨论 | `*-comments-list.json` | `model_comments`、`assets` | `comments[]` | `CommentsSection` |
| 作品展示/返图 | `*-return-pic.json` | `model_user_return_pics`、`model_counters.return_pic_count`、`assets` | detail 首屏 `works[]` + `GET /models/{id}/works` 翻页 | `WorksSection` |

当前 API:

| Method | Path | 用途 | 状态 |
|---|---|---|---|
| `GET` | `/api/v1/models/{modelId}` | 详情首屏数据,含 12 个作品 batch 和 8 条顶层评论 | 已实装 |
| `GET` | `/api/v1/models/{modelId}/works?cursor=&limit=` | 作品展示按 batch 游标分页 | 已实装 |
| `GET` | `/api/v1/models/{modelId}/comments` | 评论深分页 | 未实装,当前首屏由 detail 返回 |

资源访问规则: 模型详情页所有模型图、头像、评论头像、返图区图片都走 `/api/v1/assets/{assetId}/content`。不要在模型详情接口中直接暴露 MinIO bucket URL。

---

## 2. Detail 主数据设计

### 2.1 来源与时间字段

detail 数据来自 `tmp/动漫游戏cg质感-detail.json` 的 `data` 节点。关键字段:

| Liblib 字段 | 当前用途 |
|---|---|
| `data.id` | 写入 `models.civitai_id`,作为外部模型 id |
| `data.uuid` | 写入 `model_details.external_uuid` |
| `data.name` | 写入 `models.name`,API 输出 `title` |
| `data.modelType` | 映射为 `models.type`: `checkpoint/lora/template/workflow/video` |
| `data.userUuid` | 写入 `models.author_user_uuid`,用于 JOIN `authors` |
| `data.createTime` | 写入 `models.created_at`。详情页“已验证”当前使用 `createdAt` |
| `data.updateTime` | 写入 `models.updated_at`、API 输出 `updatedAt/fetchedAt` |
| `data.counter.*` | 写入 `model_counters` |
| `data.versions[]` | 写入 `model_versions` |
| `data.versions[0].imageGroup.images[]` | 写入 `model_cover_images` + `assets` |
| `data.description` | 写入 `model_details.description`,API 双写 `description/descriptionHtml` |
| `data.tagsV2.modelContent2` | 写入 `model_details.liblib_tags_v2` |
| `data.tagsV2.modelactivity` | 写入 `model_details.liblib_activity` |

注意: 用户之前问到的 `ml-0.5 whitespace-nowrap` 时间不是临时前端造的。模型详情时间字段已设计在 DB 中:

- `models.created_at`: 来自 `data.createTime`
- `models.updated_at`: 来自 `data.updateTime`
- `model_versions.liblib_create_time`: 来自 `versions[].createTime`
- `model_versions.liblib_update_time`: 来自 `versions[].updateTime`
- `model_details.fetched_at`: importer 抓取/写入时间,不是模型发布时间

详情页“已验证”现在取 `detail.createdAt ?? detail.fetchedAt`。如果后续 UI 要严格显示 Liblib 更新时间,应改为取 `versionMeta.lastUpdate` 或 `updatedAt`。

### 2.2 DB 表职责

| 表 | 职责 |
|---|---|
| `models` | 模型主身份、标题、类型、基础模型、作者冗余、封面、状态、创建/更新时间 |
| `model_details` | 详情快照: HTML 描述、版本 uuid/name、Liblib 标签树、推荐提示词、raw payload |
| `model_versions` | 版本级信息: version id/uuid/name、base type、versionIntro 推荐参数、trigger、original 标记、Liblib 版本时间 |
| `model_cover_images` | 详情页大图轮播/模型样例图。每行一张图,按 `sort` 升序 |
| `model_counters` | 详情页计数: 生图、图片、返图作品、评论、下载、热度 |
| `assets` | 对象存储索引。`purpose` 区分 `model_cover/user_return_pic/author_avatar` 等;`format_kind` 区分 `original/webp` |

### 2.3 API 契约

`GET /api/v1/models/{modelId}` 的主字段:

```ts
interface ModelDetail {
  id: string;
  title: string;
  type: "checkpoint" | "lora" | "template" | "workflow" | "video";
  mediaType: string;
  cover: string;
  baseModel?: string;
  author?: string;
  authorAvatar?: string;
  authorUserUuid?: string;
  typeTag?: string;
  typeTagSuffix?: string;
  exclusive?: boolean;
  versions?: Array<{ id?: string; label?: string }>;
  activeVersion?: string;
  versionMeta?: { lastUpdate?: string; firstRelease?: string };
  tags?: Array<{ tagKind?: "modelactivity" | "modelContent2"; tagLabel: string }>;
  gallery?: Array<{ assetId?: string; imageUrl: string; src?: string; width?: number; height?: number; alt?: string }>;
  stats?: { runCount?: number; downloadCount?: number; imageCount?: number; likeCount?: number };
  headCounters?: unknown[];
  authorStats?: AuthorStats;
  commentCount?: number;
  comments?: CommentItem[];
  workCount?: number;
  works?: WorkItem[];
  params?: Array<{ name: string; default?: string }>;
  isOriginal?: boolean;
  openAccess?: boolean;
  openMix?: boolean;
  createdAt?: string;
  updatedAt?: string;
  fetchedAt?: string;
  description?: string;
  descriptionHtml?: string;
}
```

关键派生规则:

- `gallery[]` 来自 `model_cover_images`,完整列表返回,前端按两张一页轮播。
- `tags[]` 优先使用 `liblib_tags_v2/modelactivity`;缺失时由 `type/typeTag/baseModel/typeTagSuffix` 兜底。
- `params[]` 从 `model_versions.version_intro` 的 `templateInfo[]` 摊平。
- `stats.imageCount` 是模型总产图/图片数,不是作品展示数量。
- `workCount` 才是作品展示标题里的数量,来自 `model_counters.return_pic_count`。

---

## 3. Author 数据设计

### 3.1 来源

作者数据来自 `tmp/动漫游戏cg质感-author.json`,对应 Liblib `user/getByUuid`。同时 detail JSON 的 `data.userUuid` 作为 author 文件缺失时的 fallback。

| Liblib 字段 | DB 字段 | API 字段 |
|---|---|---|
| `userDetail.uuid` | `authors.user_uuid` | `authorUserUuid` |
| `userDetail.nickname` | `authors.nickname` | `author` |
| `userDetail.headPic` | `assets(purpose=author_avatar)` + `authors.head_asset_id` | `authorAvatar` |
| `counter.followCount` | `authors.follow_count` | `authorStats.followCount` |
| `counter.fanCount` | `authors.fan_count` | `authorStats.followerCount` |
| `counter.workCount` | `authors.work_count` | `authorStats.uploadCount` |
| `counter.likeCount` | `authors.like_count` | `authorStats.likeCount` |
| `counter.runCount` | `authors.run_count` | `authorStats.runCount` |
| `counter.downloadCount` | `authors.download_count` | `authorStats.downloadCount` |

### 3.2 JOIN 与兜底

`models.author_user_uuid = authors.user_uuid` 是作者卡的主 JOIN。渲染优先级:

1. `authors.nickname/head_asset_id` 存在时,用作者表。
2. 作者表缺失时,回退 `models.author_name/author_avatar_id`。
3. 评论和返图区有自己的头像时,优先用各自的 `head_asset_id`;缺失时再回退作者头像。

作者卡认证状态当前由 `AuthorProfile.IsAuthorized=true` 固定输出。后续如接 Liblib 认证字段,应新增明确列,不要继续前端硬编码。

---

## 4. Comments 评论数据设计

### 4.1 来源与导入

评论来自 `tmp/动漫游戏cg质感-comments-list.json` 的 `data.commentList`。导入器 `cmd/importer/liblib-deep/comments.go` 先删除当前模型旧评论,再深度优先写入 `model_comments`。

| Liblib 字段 | DB 字段 | API 字段 |
|---|---|---|
| `commentId` / `id` | `model_comments.liblib_comment_id` | `commentId` |
| `parentId` / `replyTo` | `parent_id` | 用于组装 `replies[]` |
| 顶层评论 id | `root_id` | 暂不输出 |
| 祖先链 | `path` | 暂不输出 |
| `userUuid` | `user_uuid` | `userUuid` |
| `nickName` | `nickname` | `nickname/author` |
| `headPic` | `assets(author_avatar)` + `head_asset_id` | `headPic/avatar` |
| `content` | `content` | `content` |
| `likeCount` | `like_count` | `likeCount/likes` |
| `isAuthor` | `is_author` | `isAuthor` |
| `topFlag` | `top_flag` | `topFlag` |
| `createTime` / `time` | `liblib_time` | `createdAt/date` |

Liblib 样本中 `commentId` 可能为空。导入器用 `modelID|userUUID|content` 的 SHA1 截断值生成稳定 `liblib_comment_id`,避免多个空 id 在唯一索引上冲突。

### 4.2 查询与 API 形态

当前 detail 首屏查询:

- 顶层评论: `WHERE parent_id = 0 ORDER BY top_flag DESC, like_count DESC, liblib_time DESC LIMIT 8`
- 回复: 对每条顶层评论再查一层 `WHERE parent_id = top.liblib_comment_id ORDER BY liblib_time ASC`

API 输出:

```ts
interface CommentItem {
  commentId?: number;
  userUuid?: string;
  nickname?: string;
  author?: string;
  headPic?: string;
  avatar?: string;
  content: string;
  createdAt?: string;
  date?: string;
  likeCount?: number;
  likes?: number;
  isAuthor?: boolean;
  topFlag?: boolean;
  replies?: CommentItem[];
}
```

双写字段说明:

- `nickname` 与 `author` 同值,兼容旧 UI 与新 UI。
- `createdAt` 与 `date` 同值,都是 RFC3339。
- `likeCount` 与 `likes` 同值。
- `headPic` 与 `avatar` 同值。

后续如果实现 `/comments` 分页,必须保留顶层排序和一层回复组装规则。不要让前端用扁平列表自己推树。

---

## 5. Return Pics 作品展示数据设计

### 5.1 关键语义

Liblib 返图数据来自 `tmp/动漫游戏cg质感-return-pic.json`,对应 `returnPicList/getReturnPic`。这里有两个容易混淆的数量:

- `data.returnNum`: 作品展示标题数量。当前样本是 100,落到 `model_counters.return_pic_count`,API 输出 `workCount`。
- `data.dataList[].pics[]`: 实际图片数量。一个作品 batch 可以有多张图。当前全量样本是 100 个 batch、108 张图片。

UI 必须按 batch 渲染作品卡,不是按单张图片渲染列表。一个 batch 多张图时,卡内用轮播。

### 5.2 DB 设计

表 `model_user_return_pics` 是“一图一行”,但 service 查询时按 `batch_id` 聚合成“一作品多图”。

| Liblib 字段 | DB 字段 | API 字段 |
|---|---|---|
| `imageBatch` | `batch_id` | `batchId` |
| `userUuid` | `user_uuid` | `userUuid` |
| `nickName` | `nickname` | `nickname/author` |
| `headPic` | `head_asset_id` | `headPic/avatar` |
| `title` | `title` | `title` |
| `remark` | `remark` | `remark` |
| `likeCount` | `like_count` | `likeCount/likes` |
| `commentCount` | `comment_count` | `commentCount` |
| `isAuthor` | `is_author` | `isAuthor` |
| `topFlag` | `top_flag` | `topFlag` |
| `createTime` | `liblib_create_time` | `createdAt` |
| `pics[].id` | `liblib_image_id` | `pics[].liblibImageId` |
| `pics[].uuid` | `liblib_image_uuid` | `pics[].liblibImageUuid` |
| `pics[].imageUrl` | `assets(format_kind=original)` | `pics[].imageUrl/src` |
| `pics[].webpUrl` | `assets(format_kind=webp)` | 当前优先 original asset URL |
| `pics[].width/height` | `width/height` | `pics[].width/height` |

重要约束: `pics[].id` 在 Liblib 返图接口里不是全局唯一。生产 upsert key 必须是 `pics[].uuid`。迁移 `000015_model_return_pics_image_uuid` 已新增:

- `model_user_return_pics.liblib_image_uuid`
- `UNIQUE KEY uk_murp_liblib_uuid (tenant_id, liblib_image_uuid)`
- `idx_murp_liblib_id` 仅保留调试/兼容查询用途

导入器在发现 payload 有 UUID 时会清理当前模型历史 `liblib_image_uuid IS NULL` 的旧行,防止早期用 numeric id 导入造成只能刷新 20 多张的问题复发。

### 5.3 API 与分页

详情接口首屏返回前 12 个 batch:

```ts
interface WorkItem {
  batchId?: number;
  userUuid?: string;
  nickname?: string;
  author?: string;
  headPic?: string;
  avatar?: string;
  title?: string;
  remark?: string;
  likeCount?: number;
  likes?: number;
  commentCount?: number;
  topFlag?: boolean;
  isAuthor?: boolean;
  createdAt?: string;
  pics?: Array<{
    assetId?: number;
    imageUrl: string;
    src?: string;
    width?: number;
    height?: number;
    liblibImageId?: number;
    liblibImageUuid?: string;
  }>;
}
```

滚动到底部后前端调用:

```text
GET /api/v1/models/{modelId}/works?cursor={batchOffset}&limit=12
```

响应:

```ts
interface ModelWorksResponse {
  items: WorkItem[];
  nextCursor?: string | null;
  hasMore: boolean;
}
```

分页规则:

- `cursor` 是 batch offset,不是图片 offset。
- service 每次查 `limit + 1` 个 batch 判断 `hasMore`。
- 下一页 `nextCursor = offset + len(items)`。
- limit 上限 24,默认 12。
- 查询排序: `top_flag DESC, like_count DESC, sort_time DESC, batch_id DESC`。
- 前端合并时按 `batchId` 去重。

UI 规则:

- 容器最大宽度 1204,四列等宽瀑布流。
- 分列向下渲染,不是 CSS grid 行对齐;每列宽度一致,高度按图片比例自然增长。
- `workCount` 显示标题 `作品展示({workCount})`。
- 有更多数据时底部 sentinel 触发刷新。
- 无更多数据时显示 `暂时没有更多～`。

---

## 6. Importer 规范

入口: `eye-trace-server/backend/cmd/importer/liblib-deep`

期望文件命名:

| 文件 | 作用 |
|---|---|
| `<stem>-detail.json` | 模型详情、版本、样例图、计数 |
| `<stem>-author.json` | 作者卡、作者头像、作者计数 |
| `<stem>-comments-list.json` | 评论树 |
| `<stem>-return-pic.json` | 作品展示返图 |

导入顺序:

1. `upsertAuthor`: 作者与作者头像。
2. `upsertModel`: 主模型行,写 `created_at/updated_at`。
3. `upsertVersions`: 版本信息与 `version_intro`。
4. `upsertCovers`: 样例图/详情轮播。
5. `upsertDetails`: HTML 描述、tagsV2、suggestedPrompts、author_user_uuid。
6. `upsertReturnPics`: 返图 batch 与图片资产。
7. `upsertComments`: 评论树与评论头像。
8. `updateModelCoverAndCounters`: 封面与计数,含 `return_pic_count`。

资产写入:

- 使用 `assets.purpose + source_id + format_kind` 做幂等。
- 模型样例图: `purpose=model_cover`。
- 作品展示图片: `purpose=user_return_pic`。
- 作者/评论/返图区头像: `purpose=author_avatar`。
- 详情页 API 输出时都经 `assets.URL(assetID)` 生成 `/api/v1/assets/{id}/content`。

---

## 7. 前端消费边界

文件:

- `eye-trace-client/src/api/models.ts`
- `eye-trace-client/src/app/routes/ModelDetailPage.tsx`

前端不再使用 YAML mock 或 zod fallback。`getModelById` 直接消费 server `ModelDetail`。缺失字段只能做空态,不能自行构造假数据。

关键 UI 映射:

| UI 区域 | 字段 |
|---|---|
| 标题/标签/计数 | `title`、`tags[]`、`headCounters` 或 `stats` |
| 双图轮播 | `gallery[].imageUrl` |
| 描述 | `description` |
| 评论区 | `comments[]`、`commentCount` |
| 作者卡 | `author`、`authorAvatar`、`authorUserUuid`、`authorStats` |
| 版本详情卡 | `type`、`isOriginal`、`stats`、`baseModel` |
| 推荐参数 | `params[]` |
| 许可范围 | `openAccess`、`openMix` |
| 作品展示 | `works[]`、`workCount`、`/works` 分页 |

WorksSection 当前使用自实现瀑布流分列算法:

- `distributeWorks(works, 4)` 按估算高度把卡片放入当前最短列。
- 图片比例来自 `pics[0].width/height`,缺失时默认 `3/4`。
- 多图 batch 在 `WorkCardCarousel` 内渲染卡内轮播。

---

## 8. 验证清单

数据正确性:

```sql
SELECT COUNT(*) rows_count,
       COUNT(DISTINCT batch_id) batches,
       COUNT(DISTINCT liblib_image_uuid) uuids
FROM model_user_return_pics
WHERE model_id = 15;

SELECT return_pic_count, image_count, comment_count
FROM model_counters
WHERE model_id = 15;
```

API 翻页正确性:

```bash
curl 'http://localhost:8080/api/v1/models/15/works?limit=12'
curl 'http://localhost:8080/api/v1/models/15/works?limit=12&cursor=12'
```

浏览器正确性:

- `作品展示(100)` 与 DB `return_pic_count` 一致。
- 滚动到底部能按 12 个 batch 追加。
- 最终展示 100 个 batch 后显示 `暂时没有更多～`。
- 评论区显示作者徽标、头像、回复、点赞数和 RFC3339 时间格式化结果。
- 首页/详情页图片没有 `naturalWidth=0` 的破图。

---

## 9. 后续迭代约束

1. 不要用 Liblib numeric `pics[].id` 做返图区唯一键。只能用 `pics[].uuid`。
2. 不要把 `model_counters.image_count` 当作品展示数量。作品展示用 `return_pic_count/workCount`。
3. 不要在前端 hardcode 评论/返图 mock 数据。没有 server 数据就显示空态。
4. 评论树由 server 组装,前端只渲染 `comments[].replies[]`。
5. 模型详情资源统一走 `/api/v1/assets/{id}/content`,不要混入 MinIO 直链。
6. 新增字段时先落 DB/导入器/API 文档,再改 UI。否则会再次出现“看起来渲染了,但数据不是生产字段”的问题。

---

## 10. 待办

- [ ] `GET /api/v1/models/{id}/comments` 分页端点。
- [ ] `versionMeta.lastUpdate/firstRelease` 改为优先使用 `model_versions.liblib_update_time/liblib_create_time`。
- [ ] `isOriginal` 从 `model_versions.is_original` 输出真实值。
- [ ] `openAccess/openMix` 从生产授权字段派生,不要长期固定 false。
- [ ] `webp_asset_id` 作为返图区图片输出优先源的策略确认:当前 API 仍输出原图 asset URL。
- [ ] 作者认证状态接入真实字段。
