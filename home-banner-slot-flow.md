# Home Banner Slot 数据流 — 设计-落地-排查-完成

> 时间:2026-06-24 ~ 2026-06-25
> 范围:`/api/v1/content/home` 的 `bannerSlots` 字段端到端打通
> 涉及子仓库:`eye-trace-docs`、`eye-trace-config`、`eye-trace-client`、`eye-trace-server`

---

## 1. 设计

### 1.1 三种 slot 类型(源自 liblib.art 首页 DOM)

参考 `tmp/slot.html`(`class="flex gap-2 my-4 b768:mt-0 b768:mb-6 w-full"` 容器),三种变体:

| 类型 | DOM 特征 | 类名 | 行为 |
|---|---|---|---|
| carousel | 子 div 含 `w-[36.42%]` | `relative h-full w-[36.42%] rounded-lg overflow-hidden group` | 多图轮播 + 左右箭头 + 指示点 |
| static | 子 div 含 `flex-1`、无底部按钮 | `relative h-full flex-1 rounded-lg overflow-hidden cursor-pointer` | 单图,整图可点击 |
| staticActions | 子 div 含 `flex-1` + 底部 a 按钮条 | `relative h-full flex-1 ... + bottom-2 absolute flex flex-nowrap gap-0.5 md:gap-1` | 单图 + 底部 1..8 个按钮 |

### 1.2 跳转目标

仅内部路由(viewId + params),**不暴露**外链字段。外部链接由 client 包成 webview viewId。

### 1.3 契约位置(替代旧 BannerCarouselConfig)

| 字段 | 旧(已删) | 新 |
|---|---|---|
| 字段名 | `bannerCarousel` | `bannerSlots`(数组) |
| 内部结构 | `carousel + sideCards + overlayLinks` 单一 config | discriminatedUnion 三种 slot |
| 跳转模型 | `href: string` | `target: { viewId, params }` |
| 命名 | `src` | `imageUrl` |

### 1.4 DB schema(两张表 + JSON 字段,后续可平滑扩)

```sql
home_banner_slots     (id, position, kind, enabled, image_key, image_alt)
home_banner_slides    (id, slot_id, position, image_key, image_alt)
home_banner_actions   (id, slot_id, position, label, icon_key)
home_banner_targets   (id, scope, ref_id, view_id, params_json)
home_banner_version   (id, version) -- 用于 ETag
```

- `scope='banner|slide|action'` + `ref_id` 关联具体元素
- `home_banner_slots.image_key` 给 static / staticActions 用;carousel 此列为 NULL,实际图存 `home_banner_slides`

### 1.5 MinIO URL 规则

```
DB image_key:    home/banner/carousel/0.webp
                     ↓ URLBuilder
client imageUrl: http://127.0.0.1:9000/eye-trace-assets/home/banner/carousel/0.webp
```

两模式:`PublicBase` 直连 MinIO(开发期)/ server 代理 `/api/v1/assets/banner/{key}`(生产)。

---

## 2. 落地

### 2.1 文档 & 配置

| 文件 | 变更 |
|---|---|
| `eye-trace-docs/openapi.yaml` | 新建全量契约,含 `BannerSlot` discriminatedUnion + `RouteTarget` + 6 个端点 |
| `eye-trace-docs/openapi-skeleton.md` | §1.3 标注替代、`§3` 加 schema 片段、`§5` codegen 路径 |
| `eye-trace-config/home/banner-slots.example.yaml` | seed 模板(URL 留 `__PENDING__`) |

### 2.2 数据库迁移

- `migrations/000001_home_banner_slots.up.sql` — 5 张表 DDL
- `migrations/000001_home_banner_slots.down.sql` — DROP 顺序(被 FK 依赖倒置)
- `migrations/000002_home_banner_slots_seed.up.sql` — 4 slot + 6 slide + 3 action + 12 target,显式 id
- `migrations/000002_home_banner_slots_seed.down.sql` — 撤销

### 2.3 Server(Go)

- `internal/api/api.gen.go` — oapi-codegen 产物(1303 行,从 openapi.yaml 生成)
- `internal/content/banner_repo.go` — `MySQLRepo` 实现 `BannerRepo` 接口(ListEnabledSlots / ListSlides / ListActions / GetTarget / GetVersion)
- `internal/content/banner_service.go` — `Service.GetHome` 串联 repo + URLBuilder,产出 mid-shape + ETag
- `cmd/banner-server/main.go` — 最小启动入口,只暴露 `/api/v1/content/home` 公开端点;其余 10 个端点靠 `api.Unimplemented` 兜底(返 501)

### 2.4 Client

| 文件 | 变更 |
|---|---|
| `src/config/static/types.ts` | 删 4 老类型,新增 `RouteTarget` / `SingleBanner` / `CarouselSlide` / `ActionButton` / `BannerSlot*` |
| `src/api/schemas.ts` | 新 `BannerSlotSchema` discriminatedUnion + `HomeContentSchema.bannerSlots`(max 6) + `vipPromo.nullable()` |
| `src/api/content.ts` | `fallback.bannerSlots = null`(无 YAML 兜底) |
| `src/api/schema.gen.ts` | 从 openapi.yaml 重新生成 |
| `src/components/layout/DynamicBannerCarousel.tsx` | 重写三种 slot 块;`slots` 必填,无 client 兜底 |
| `src/config/static/default.yaml` | 删 `bannerCarousel` 种子;新增 bannerSlots 也删除(全 server) |
| `src/app/routes/HomePage.tsx` / `ImageModelListPage.tsx` | `showBanner = !!content.bannerSlots && length > 0` |
| `src/stores/content.ts` | import 路径从 `schema.gen` 切到 `@/api/schemas` |
| `src/components/layout/CenterBar.test.tsx` / `api/__tests__/content.test.ts` | 同步字段名 bannerSlots |

### 2.5 MinIO 抓取脚本

`tmp/scrape_slot.py`(一次性,留档):
- stdlib `html.parser.HTMLParser` 切片最外层 strip 容器,逐 slot 提取 img src
- 重试 4 次应对 OSS 偶发 read timeout
- 输出 `tmp/slot-images/{carousel,static,staticActions}/{position}.webp` + `manifest.json`
- 6 张 carousel + 2 张 static + 1 张 staticActions + 3 个 action label(WebUI / ComfyUI / 训练 LoRA)

### 2.6 部署链路

```
MySQL eye_trace (5 表)              MinIO eye-trace-assets (9 obj)
  home_banner_slots                    home/banner/carousel/{0..5}.webp
  home_banner_slides                   home/banner/static/{0,1}.webp
  home_banner_actions                  home/banner/staticActions/0.webp
  home_banner_targets
  home_banner_version
            │                                  │
            └─────────────┬────────────────────┘
                          ▼
            cmd/banner-server (Go 1.25)
            GET /api/v1/content/home  :8080
                          │
                          ▼
            eye-trace-client fetch
            zod parse → 渲染
                          │
                          ▼
            Chrome @ http://localhost:1420/
```

---

## 3. 排查

### 3.1 现象

- Server `curl /api/v1/content/home` 返 200 + 1992 字节 JSON(4 slot,完整 data)
- 浏览器 console 看不到 `GET /content/home` 请求记录(实际有,但被 ETag 命中逻辑吞)
- DOM 里 `.flex.gap-2.my-4` 不存在
- 右下角显示 `home: fallback` —— store 走了 fallback 路径

### 3.2 排查步骤

1. **curl 直击 server**:200 + JSON 完整,问题不在 server
2. **curl 通过 vite proxy `localhost:1420/api/v1/content/home`**:200 + JSON,代理 OK
3. **mcp chrome-devtools Network**:确认 `GET /api/v1/content/home` 实际发出 200
4. **DOM 检查**:`slotExists: false`、`home: fallback`
5. **怀疑 zod parse 失败**:把 `api/content.ts` 的 catch 加上 `console.error(JSON.stringify(e))`
6. **console 暴露错误**:
   - 第一次:`{ issues: [{ path: ["vipPromo"], expected: "object", received: "null" }] }` — `vipPromo` 没 nullable
   - 第二次(修了 vipPromo 后):`{ path: ["bannerSlots"], ... max=3 }` — 数组上限 3 不够

### 3.3 根因

| 错因 | 根 | 修复 |
|---|---|---|
| `vipPromo: null` 拒收 | `VipPromoSchema` 无 `nullable` | `VipPromoSchema.nullable().optional()` |
| `bannerSlots` 4 个 slot 被拒 | `z.array(...).max(3)` 与 liblib 实际不符 | `max(6)`(留余量) |

### 3.4 教训

- **设计契约时的 `maxItems` 来自 UI 假设,需要先采样真实数据再定**:liblib 实际是 4 块,不是 3 块
- **server 兜底 null 字段要先对齐 client zod**:`vipPromo/recommend/modelItems/typeTabs/subTabs` 这些**未来才接**的字段,server 先返 `null` 时,client 端 zod 必须先标 `nullable().optional()`,否则契约漂移会让 parse 失败 → fallback,但 fallback 又无数据 → UI 静默不渲染
- **fallback 链路吞所有错误**对生产友好但对调试不友好:加临时 `console.error(e)` 后能立刻看到 zod issue path
- **5xx/4xx 与 zod parse fail 必须分通道**:HTTP 200 + parse 失败时,console 没任何报错,极易让人误判"接口挂了"

---

## 4. 完成

### 4.1 浏览器最终态(`tmp/home-final.png`)

| slot | 类型 | 渲染 |
|---|---|---|
| 第 1 块 | carousel | "5月收益排行榜已更新" 等 6 张轮播图 |
| 第 2 块 | static | "图片视频生成" |
| 第 3 块 | static | "LibTV" |
| 第 4 块 | staticActions | "开源工具" + 底部按钮 WebUI / ComfyUI / 训练 LoRA |

store badge 显示 `home: server etag "dd36f414..."` — 来自 server,非 fallback。

### 4.2 数字

| 维度 | 数 |
|---|---|
| DB 表 | 5 |
| DB 行(slots/slides/actions/targets) | 4 / 6 / 3 / 12 |
| MinIO 对象 | 9 个 webp,总 335 KiB |
| 契约 schema | `BannerSlot` + 3 个 variant + 3 个共享片段,共约 130 行 yaml |
| Server LOC | `banner_repo.go` 130 + `banner_service.go` 160 + `cmd/banner-server/main.go` 270 |
| 端点 | 1 个真实现(`/content/home`),10 个 unimplemented 返 501 |

### 4.3 后续(server 重构那条线接手)

| 项 | 备注 |
|---|---|
| `internal/api/handler/{auth,asset,content,model}.go` + middleware | 当前 cmd/banner-server 用 oapi-codegen 兜底,正式 handler 重构时补完整 |
| `GET /api/v1/assets/banner/{key}` 代理 | 当前 PublicBase 直连 MinIO,生产可改成 server 代理 |
| `internal/api/router.go` | cmd/server/main.go 老装配链需补回 |
| 其余 10 个端点(workflow-runs / models / auth / assets / me) | 全是 unimplemented,需要逐个补 |
| vipPromo / recommend / modelItems / typeTabs / subTabs | server 暂返 null,client zod 已支持 nullable,后续接表 |

### 4.4 文件清单(全部交付)

```
eye-trace-docs/
├── openapi.yaml                                            # 全量契约 + BannerSlot discriminatedUnion
├── openapi-skeleton.md                                     # §1.3/§3/§5 标注 + schema 片段

eye-trace-config/
└── home/banner-slots.example.yaml                          # seed 模板

eye-trace-server/backend/
├── migrations/000001_home_banner_slots.{up,down}.sql       # 5 表 DDL
├── migrations/000002_home_banner_slots_seed.{up,down}.sql  # 显式 id seed
├── internal/api/api.gen.go                                 # oapi-codegen 产物(1303 行)
├── internal/content/banner_repo.go                         # MySQL 实现
├── internal/content/banner_service.go                      # Service.GetHome + ETag
└── cmd/banner-server/main.go                               # 最小启动入口

eye-trace-client/src/
├── api/schema.gen.ts                                       # 重生
├── api/schemas.ts                                          # BannerSlot discriminatedUnion
├── api/content.ts                                          # fallback.bannerSlots=null
├── api/__tests__/content.test.ts                           # 同步字段名
├── config/static/types.ts                                  # 删 4 老类型 + 新 5 类型
├── components/layout/DynamicBannerCarousel.tsx             # 重写三种 slot 块
├── components/layout/CenterBar.test.tsx                    # mock 同步
└── app/routes/HomePage.tsx, ImageModelListPage.tsx         # showBanner 读 bannerSlots

tmp/
├── slot.html                                               # liblib DOM 切片
├── scrape_slot.py                                          # 一次性抓取脚本
├── slot-images/                                            # 本地副本
├── slot-images/manifest.json                               # 元数据
└── home-final.png                                          # 端到端验证截图
```
