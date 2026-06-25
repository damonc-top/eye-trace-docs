# Home Recommend 数据流 — 设计-落地-排查-完成

> 时间:2026-06-25
> 范围:`/api/v1/content/home` 的 `recommend` 字段端到端打通
> 涉及子仓库:`eye-trace-docs`、`eye-trace-server`、`tmp/`
> 跟 banner 同构(也走 mid-shape → URLBuilder → openapi marshaling),但块本身更小(1 singleton + N cards)。

---

## 1. 设计

### 1.1 数据形态(源自 `tmp/py-4.html` liblib 推荐块)

```
<div class="py-4">
  <div class="mb-4 flex min-w-0 items-center gap-3 ...">
    <span>一键使用顶尖视频/图片模型</span>     ← 块标题(硬编码在 client,不走 server)
    <div class="min-w-0 shrink">
      <div class="... cursor-pointer shrink" style="background-color: rgb(255, 247, 234);">
        <div class="... h-[22px]">           ← 22px 高 marquee 文案
          <span style="animation-delay: 0s;">S</span>
          <span style="animation-delay: 0.04s;">e</span>
          ...                                ← 每字一个 span,逐字 fade-in
        </div>
        <svg>›</svg>                         ← 右 caret,跳转到 viewId
      </div>
    </div>
  </div>
  <div class="relative overflow-hidden w-[calc(100%+16px)]">
    <div class="flex gap-2 select-none overflow-x-auto pb-2 ...">
      <div class="group ... h-[65px] rounded-[8px] ..." style="width: 248px; background-color: rgb(184, 240, 255);">
        <div class="absolute inset-[1px] ... flex items-center">
          <svg class="absolute left-0 top-0 ...">
            <path d="M46.2754 0C43.0751 0 ..."/>   ← corner accent(accentColor 填色)
          </svg>
          <img src="...seedance.png" class="w-[20px] h-[20px] absolute left-[6px] top-1" />  ← 左上 logo
          <div class="ml-[40px]">
            <div class="text-[14px]">Seedance 2.0</div>
            <div class="text-[12px]">最强视频模型,全能参考,15s音画同出</div>
          </div>
          <div class="absolute right-1 top-1"><img src="...new-badge.png" /></div>   ← 右上 badge(NEW/HOT)
          <div class="absolute right-2 bottom-2 ...">去使用</div>                   ← hover 才出现的 CTA
        </div>
      </div>
      ...8 张
    </div>
  </div>
</div>
```

**两块组成**:
- **announcement(可选)**:per-char 渐显 marquee 文本 + 跳转目标
- **models**:横向滚动卡片,每张 248px 宽 × 65px 高,带 `bgColor`(整卡底色) + `accentColor`(左上 corner accent SVG 填充色)

### 1.2 与 banner 的设计差异

| 维度 | banner | recommend |
|---|---|---|
| 位置 | `flex w-full gap-2 my-4` 容器(左 → 右并排) | `py-4` 容器(banner 下方独立块) |
| 元素数 | 4 个 slot,discriminated 3 种类型 | 1 个 singleton + 1..N 张 card(同构 N) |
| 跳转 | `target: RouteTarget`(viewId+params) | `target: { viewId, params?, externalHref? }`(卡片同时支持 viewId 跳转与外链;announcement 同) |
| 文本字段 | 仅 `imageAlt` | `title` + `desc` + `text` (announcement) |
| 配色 | 单图,无内联颜色 | 每卡独立 `bgColor` + `accentColor` |
| Badge | 无 | 右上角 badge(NEW / HOT / 独家) |
| 资源 key | `home/banner/{kind}/{position}.{ext}` | `home/recommend/icons/{slug}.png`(只 model icon,无图) |
| 版本表 | `home_banner_version` | `home_recommend_version`(独立,各自 bump) |

### 1.3 DB schema(3 张表 + singleton)

```sql
home_recommend          (id PK=1, enabled, announce_text/view_id/params_json/external_href,
                          card_width, per_char_delay, char_fade_ms, hold_ms)   -- 单例
home_recommend_models   (id, position UNIQUE, title, desc_text,
                          bg_color "#RRGGBB", accent_color "#RRGGBB",
                          icon_key, badge)                                       -- 一行一张卡
home_recommend_targets  (id, ref_id FK→models, view_id, params_json, external_href)  -- 一行一卡的跳转
home_recommend_version  (id PK=1, version)                                       -- ETag 单例
```

- `enabled = FALSE` → 整块不下发(client 当 null 收)
- `home_recommend` 单例行不存在 → 整块当 null(client 静默不渲染)
- target 走 LEFT JOIN,允许空(`view_id` 可为 NULL,客户端跳过 `openView` 调用)

### 1.4 MinIO URL 规则(只 icon)

```
DB icon_key:    home/recommend/icons/seedance.png
                       ↓ URLBuilder.RecommendIconURL
client icon:    http://127.0.0.1:9000/eye-trace-assets/home/recommend/icons/seedance.png
```

`URLBuilder` 接口比 banner 多 1 个方法(`RecommendIconURL`);生产模式可代理到 `/api/v1/assets/recommend/{key}`(后续 PR)。

### 1.5 契约位置(对齐 liblib 行为)

openapi.yaml `Recommend` schema:
- `bgColor` / `accentColor` 加 `pattern: ^#[0-9A-Fa-f]{6}$`(防 hex 漂移)
- `icon` 改 `format: uri`(明确是 URL)
- `badge` 加 `maxLength: 8`(对齐 banner `ActionButton.label` 的 a11y 约束)
- `cardWidth` / `perCharDelay` / `charFadeMs` / `holdMs` 全部 required + default(动画参数在 server 端存实际值,而不是依赖 schema default,避免后续微调埋坑)

---

## 2. 落地

### 2.1 文档 & 配置

| 文件 | 变更 |
|---|---|
| `eye-trace-docs/openapi.yaml` | `RecommendModelCard` 加 pattern / format / maxLength 约束 |
| `eye-trace-docs/home-recommend-flow.md` | 本文档 |

### 2.2 数据库迁移

- `migrations/000003_home_recommend.up.sql` — 4 张表 DDL(singleton + models + targets + version)
- `migrations/000003_home_recommend.down.sql` — DROP 顺序(targets → models → singleton → version)
- `migrations/000004_home_recommend_seed.up.sql` — 1 singleton + 8 model card + 8 target,显式 id 1..8
- `migrations/000004_home_recommend_seed.down.sql` — 反向清 seed

**seed 数据**(对照 `default.yaml` `recommend` 块 + `py-4.html`):

| pos | title | bgColor | viewId | icon key |
|---|---|---|---|---|
| 0 | Seedance 2.0 | #B8F0FF | tool.video | seedance.png (NEW) |
| 1 | Seedance 2.0 Fast | #B8F0FF | tool.video | seedance.png (NEW) |
| 2 | Happy Horse 1.0 | #FBE8A5 | tool.video | happy-horse.png |
| 3 | 智能图片V2 | #D5E4FF | tool.image | img-v2.png |
| 4 | 全能图片V2-Flash | #FBE8A5 | tool.image | img-v2-flash.png |
| 5 | Seedream 5.0 lite | #D1E9FF | tool.image | seedream.png |
| 6 | 可灵 3.0 | #B7F8B7 | tool.video | keling.png |
| 7 | 可灵 3.0 Omni | #B7F8B7 | tool.video | keling.png |

announcement:`'Seedance 2.0 低至 0.35元/秒,不排队!会员限时 37 折,即将结�优惠!'` → `viewId: tool.video, params: {}`

### 2.3 Server(Go)

| 文件 | 变更 |
|---|---|
| `internal/content/recommend_repo.go` | `RecommendRepo` 接口 + `MySQLRecommendRepo`(`GetMeta` / `ListCards` / `GetVersion`)。`ListCards` 用 LEFT JOIN 一把拿到 target 行 |
| `internal/content/banner_service.go` | `URLBuilder` 接口加 `RecommendIconURL`;`HomeView.Recommend *RecommendView` 字段;`Service` 收 `recRepo`;`buildRecommend(ctx)` 拼 mid-shape(announcement + 8 card);`computeETag` 把 recommend 内容(announcement text/viewId、动画参数、每卡 title/desc/bg/accent/badge、target viewId/externalHref)纳入 hash |
| `internal/content/banner_service.go` 的 mid-shape | 新增 `RecommendView` / `AnnouncementView` / `RecommendCardView` / `RecommendTargetView`(独立于 banner 的 `TargetView`,因为 recommend target 多了 `externalHref` 平级字段) |
| `cmd/banner-server/main.go` | `NewService(repo, recRepo, urlsAdapter)`;`urlBuilderAdapter` 加 `RecommendIconURL` 方法;新增 `toOpenAPIRecommend(*RecommendView)` 把 mid-shape 转 openapi 形态,enabled=false 或 singleton 不存在时返回 `nil`(对应 zod `nullable`) |

### 2.4 Client(零改动)

`HomePage.tsx` 早就支持 `content?.recommend`,`RecommendBanner` / `RecommendModelList` 也是 server 数据流驱动。store `content.recommend` 字段在 `schema.ts` 第 344-351 行(`RecommendSchema`)+ `HomeContentSchema.recommend: RecommendSchema.nullable().optional()` 早就就绪。**这次任务完全没碰 client 代码**,只是把 server 端的 `null` 占位换成真数据。

### 2.5 MinIO 抓取脚本

`tmp/fetch_recommend_icons.sh`(一次性):
- 6 个 liblib 模型 logo URL,严格按"card 出现顺序"排序
- 重试 4 次 + 字节数校验(避免空 body 假成功)
- 输出 `tmp/recommend-icons/{seedance,happy-horse,img-v2,img-v2-flash,seedream,keling}.png`(24-72px RGBA PNG)

上传:`mc cp recommend-icons/*.png local/eye-trace-assets/home/recommend/icons/`

### 2.6 部署链路

```
MySQL eye_trace (4 表)              MinIO eye-trace-assets (6 obj)
  home_recommend                       home/recommend/icons/seedance.png
  home_recommend_models                home/recommend/icons/happy-horse.png
  home_recommend_targets               home/recommend/icons/img-v2.png
  home_recommend_version               home/recommend/icons/img-v2-flash.png
                                       home/recommend/icons/seedream.png
                                       home/recommend/icons/keling.png
            │                                  │
            └─────────────┬────────────────────┘
                          ▼
            cmd/banner-server (Go 1.25)
            GET /api/v1/content/home  :8080
                          │
                          ▼
            RecommendBanner / RecommendModelList 组件
            (chrome @ http://localhost:1420/)
```

---

## 3. 排查

### 3.1 现象(接续 banner 任务)

banner slot 渲染 OK 后,首页"一键使用顶尖视频/图片模型"标题出现但 marquee + 卡片区**全空**。
Store 显示 `home: server · etag "dd36f414..."`,banner 块正常,只有 recommend 块缺数据。

### 3.2 排查步骤

1. **curl server 直击**:`GET /api/v1/content/home` → 200 + JSON,recommend 字段为 `null`
2. **看 `banner-server/main.go` 的 `toOpenAPIHome`**:`recommend: nil` 是硬编码的占位
3. **看 `banner_service.go` `HomeView`**:`HomeView` 没有 Recommend 字段 → `Service.GetHome` 也没拼
4. **看 `api.gen.go` `Recommend` struct**:`Announcement`、`Models`、`CardWidth`、`PerCharDelay`、`CharFadeMs`、`HoldMs` 字段已存在(早就在 openapi.yaml 定义)
5. **看 zod `RecommendSchema`**:`recommend: RecommendSchema.nullable().optional()` 早就支持 nullable → server 返 null 时不会 parse 失败(吸取 banner 任务教训)

**结论**:server 端没拼,client 端早就准备好。

### 3.3 根因 / 修复

| 问题 | 根 | 修复 |
|---|---|---|
| `cmd/banner-server/main.go` 把 recommend 硬编码为 null | 一期 Server 只实现了 banner 块,其余字段占位 null | 新增 recommend repo + service 字段,`toOpenAPIRecommend` 把 mid-shape 转 openapi |
| `RecommendTargetView` 在 recommend_repo 和 banner_service 同时定义 | 两个文件都命名 `RecommendTargetView`,类型不同(repo 层带 sql.NullString,service 层纯 string) | repo 改用 `RecommendTargetRow`,service 用 `RecommendTargetView`,显式区分 repo 行 vs marshal view |
| 第一个 build 报 `RecommendTargetView redeclared in this block` | 同上 | 同上 |
| MinIO icon URL `happy-horse.png` 客户端 200 但内容是 webp | liblib URL 带 `?image_process=format,webp` 查询参数,服务端响应 webp 流 | 重新拉 URL 不带查询参数,得到原生 PNG |
| 浏览器 cached 304(etagged `dd36f414` 与 server 新 hash 不同) | 老 etag 是 banner-only hash,recommend 拼上后 hash 全变 | bump `home_recommend_version` 到 `2026-06-25.2`,触发 client SWR 重拉 |

### 3.4 教训(对比 banner 任务)

- **同构迁移可以零 client 改动**:这次 0 行 client 代码就能渲染,关键是 banner 任务已经先把 store / zod / 组件都按"server-only"做完了。后续 modelItems / typeTabs / subTabs 也是这个模式。
- **`externalHref` 平级字段要让 mid-shape 显式建模**:`RecommendTargetView` 与 banner 的 `TargetView` 不同(多 `externalHref`),不要图省事共用一个 type,handler 转 openapi 时会乱。
- **URL 查询参数会偷换资源类型**:`?image_process=format,webp` 让 PNG endpoint 返回 webp,扩展名依然是 `.png` 但 content-type / 字节数与 native PNG 不同(`file` 命令能识别),首次 MinIO 上传后就埋坑了——liblib 的 OSS 处理路径对自动化脚本不友好。
- **server 占位 null 要让 client zod 同时 nullable** banner 任务踩过(留 VipPromo.nullable),recommend 任务直接照搬,没有重复犯错。

---

## 4. 完成

### 4.1 浏览器最终态(`tmp/home-recommend-icons.png`)

```
[LibLibAI 顶部bar]
[banner 4 块:5月收益排行榜 / 图片视频生成 / LibTV / 开源工具]
一键使用顶尖视频/图片模型  [Seedance 2.0 低至 0.35元/秒…] ›   ← 标题硬编码 + marquee 文本 + caret
  [Sd] Seedance 2.0       [Sd] Seedance 2.0 Fast  [H] Happy Horse 1.0  [▣] 智能图片V2   →  ←
       最强视频模型…              最强视频模型…            阿里最新视频…         最新图片模型…
       NEW 角标                NEW 角标
[加载失败 GET /models?limit=12&sort=latest]   ← 不在本任务范围(后续 PR)
```

store badge:`home: server · etag "af3c51e..."`

### 4.2 数字

| 维度 | 数 |
|---|---|
| DB 表 | 4(singleton + models + targets + version) |
| DB 行(singleton/models/targets) | 1 / 8 / 8 |
| MinIO 对象 | 6 PNG,总 ~4.7 KiB(都是 24-72px RGBA,UI 用 20×20 渲染) |
| 契约 schema 改动 | `RecommendModelCard` 加 3 个约束(pattern × 2 + format + maxLength) |
| Server LOC 新增 | `recommend_repo.go` 130 行 + `banner_service.go` +90 行 + `banner-server/main.go` +60 行 |
| 客户端代码改动 | **0 行**(全靠 banner 任务预先铺好的 store / zod / 组件) |

### 4.3 后续(server 重构那条线接手)

| 项 | 备注 |
|---|---|
| `RecommendIconURL` 接入 server 代理模式 | 当前 PublicBase 直连 MinIO,生产改 `/api/v1/assets/recommend/{key}` 代理 |
| `modelItems` / `typeTabs` / `subTabs` server 端实现 | 沿用本任务同构路径(singleton + cards + version + ETag hash) |
| `vipPromo` server 端实现 | 字段已 nullable,只需要单表(类似 singleton,但带图片资源) |
| icon 上传 workflow | 当前 `mc cp` 一次性脚本,生产应迁 `eye-trace-server/backend/cmd/importer/` 类似 banner 那条线,支持幂等 + 版本化 + 自动 sync 到 DB |
| `cardWidth` 等动画参数走实验 | 当前硬编码 248/0.04/600/1000,后续若要做 AB,server 端可以根据 user 群体返回不同值 |

### 4.4 文件清单(全部交付)

```
eye-trace-docs/
└── home-recommend-flow.md                                  # 本文档

eye-trace-server/backend/
├── migrations/000003_home_recommend.{up,down}.sql           # 4 表 DDL
├── migrations/000004_home_recommend_seed.{up,down}.sql      # 显式 id seed
├── internal/content/recommend_repo.go                       # RecommendRepo + MySQL 实现
├── internal/content/banner_service.go                      # +RecommendView / +buildRecommend / +ETag hash
└── cmd/banner-server/main.go                               # +recRepo 注入 / +toOpenAPIRecommend

tmp/
├── py-4.html                                               # liblib DOM 切片(输入)
├── fetch_recommend_icons.sh                                # 6 个 logo 抓取脚本
├── recommend-icons/                                        # 6 PNG 本地副本
├── apply_recommend_migrations.sh                           # DB migrate 辅助脚本
├── dump_recommend_icons.sh                                 # DB ↔ MinIO 校验
├── bump_recommend_version.sh                               # ETag bump 辅助
└── home-recommend-icons.png                                # 端到端验证截图(icons 已加载)
```