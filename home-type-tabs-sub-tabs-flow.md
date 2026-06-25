# Home typeTabs + subTabs 数据流 — 设计-落地-排查-完成

> 时间:2026-06-25
> 范围:`/api/v1/content/home` 的 `typeTabs` / `subTabs` 字段端到端打通
> 涉及子仓库:`eye-trace-docs`、`eye-trace-server`、`eye-trace-client`、`tmp/`
> 跟 banner / recommend 同构(单例 + version + ETag hash),但块更小 ——
> 纯文本 id+label,无 MinIO 资源、无 FK、无图片资源

---

## 1. 设计

### 1.1 数据形态(源自 `eye-trace-client/src/config/static/default.yaml` 第 320-358 行)

```yaml
typeTabs:                              # 顶层 4 个固定 tab
  tabs:
    - { id: image, label: "图片模型" }
    - { id: video, label: "视频特效" }
    - { id: ideas, label: "发现灵感" }
    - { id: flows, label: "工作流" }

subTabs:                               # 每个 parent 一桶
  byType:
    image:  [all, weekly, top100, photo, ecommerce, anime, illust, graphic, arch, creative, ip, novel]   # 12
    video:  [all, new, hot, portrait, short]                                                                  # 5
    ideas:  [all, featured, following]                                                                       # 3
    flows:  [all, comfyui, sd]                                                                               # 3
```

**两块组成**:
- **typeTabs** — 顶层 tab strip,首项 `image` = 默认 active
- **subTabs** — 父子两层(`byType` map 按 parent id 分桶),每桶首项 = 默认 active

### 1.2 与 banner / recommend 的设计差异

| 维度 | banner | recommend | **typeTabs** | **subTabs** |
|---|---|---|---|---|
| 位置 | 左→右并排 | banner 下方独立块 | **HomePage 上方 / 各页通用** | **TypeTabs 正下方** |
| 元素数 | 4 slot (discriminated 3 种) | 8 card | **固定 4** | **按 parent 变(3/5/12/3)** |
| 形态 | flat array | flat array + 1 singleton | **flat array** | **`byType: { parent → N-tabs }`**(map) |
| 跳转 | viewId+params | viewId+params+externalHref | **无(viewId 在 SubTab 上)** | **SubTab.viewId 可选(本次全 NULL)** |
| 资源 | 4 张 PNG | 6 张 PNG | **无** | **无** |
| 版本表 | `home_banner_version` | `home_recommend_version` | **`home_type_tabs_version`** | **`home_sub_tabs_version`** |

### 1.3 DB schema(2 singleton + 2 version 表,JSON 列)

```sql
home_type_tabs              (id PK=1, enabled, tabs_json)                              -- 单例
home_type_tabs_version      (id PK=1, version)                                       -- ETag 单例
home_sub_tabs               (id PK=1, enabled, by_type_json)                          -- 单例
home_sub_tabs_version       (id PK=1, version)                                       -- ETag 单例
```

- `enabled = FALSE` → 整块不下发(client 当 null 收)
- singleton 行不存在 → 整块当 null(client 静默不渲染)
- **JSON 列 vs normalized**:纯文本小 N(4 / 4×3-12),无 FK、无图片 → JSON 列(对照 recommend 的 normalized targets/models,本次省 2 张表)
- `byType_json` 的 key 集合 ⊇ `tabs_json[].id`(seed 静态保证)

### 1.4 URLBuilder 不需要扩展

整块无 `image_key` / `icon_key` / `external_href`,`BannerURL` / `RecommendIconURL` 够用。**接口签名不变**,`banner_service.go` 的 `URLBuilder` 接口零改动。

### 1.5 契约位置(openapi.yaml 漂移修复)

**SubTabs 形态对齐 client zod**:`openapi.yaml:316-321` 原本是 `SubTabs.tabs: []`(flat array),与 client zod 的 `byType: record(SubTab[])`(`schemas.ts:378-380`)长期漂移。本次顺手对齐:

```yaml
SubTabs:
  required: [byType, defaultActiveByType]
  properties:
    byType:              { type: object, additionalProperties: { type: array, items: { $ref: '#/components/schemas/SubTab' } } }
    defaultActiveByType: { type: object, additionalProperties: { type: string } }

SubTab:
  required: [id, label]
  properties:
    id:           { type: string, maxLength: 32 }
    label:        { type: string, maxLength: 64 }
    viewId:       { type: string, maxLength: 64 }
    params:       { type: object, additionalProperties: true }
    externalHref: { type: string, format: uri }
```

`TypeTab` / `SubTab` 加 `maxLength` 约束(`id 32 / label 64`),对齐 liblib 后端 ID 长度上限;`SubTab` 加 `viewId/params/externalHref` 可选字段(后续接子页路由时填)。

**HomeContent.required 收紧**:`openapi.yaml:112` 原本 `required: [version, bannerSlots, vipPromo, recommend, modelItems, typeTabs, subTabs]`,与 client zod `nullable().optional()` 全部冲突。改为 `required: [version]`,其余字段加 `nullable: true` — `schema.gen.ts` 重生成后类型自动同步,运行时 zod 校验不再漂移。

### 1.6 YAML fallback 边界

`client_data_split.md §4.3` line 152-153 明确把 `typeTabs.tabs` / `subTabs.byType` 分类为 **taxonomy** — "短期留 YAML,长期 → server"。本次只迁 server 真源,**YAML 块保留**(banner / recommend 那种严格 server-only 不适用,因为 typeTabs/subTabs 是导航骨架,server 不可达时必须 fallback)。

`<TypeTabs>` / `<SubTabs>` 组件已有 `config?.tabs ?? appConfig.typeTabs?.tabs ?? []` 兜底链,server 真源优先,YAML 兜底。

### 1.7 默认 active 由 server 决定

**不再让 client 算 `tabs[0]?.id`** — server 在 mid-shape 阶段直接计算 `defaultActive = Tabs[0].ID` + `defaultActiveByType[parent] = ByType[parent][0].ID`,client `HomePage` 优先用 `content.typeTabs.defaultActive`,server 没下发再兜底 `tabs[0]?.id`。

**为啥要这样**:`HomePage` 的 `useState<string>(content?.typeTabs?.tabs[0]?.id ?? "")` 在首帧(content 还没 hydrate)时拿到的是 `""`,即使后续 content 来了 state 也不会重新初始化。server 明确下发 `defaultActive`,client 端 `useEffect` 同步即可。

---

## 2. 落地

### 2.1 文档 & 配置

| 文件 | 变更 |
|---|---|
| `eye-trace-docs/openapi.yaml` | `SubTabs.tabs → byType` + `SubTab` 加可选字段 + `HomeContent.required` 收为 `[version]` |
| `eye-trace-docs/home-type-tabs-sub-tabs-flow.md` | 本文档 |

### 2.2 数据库迁移

- `migrations/000005_home_tabs.up.sql` — 4 表 DDL(2 singleton + 2 version 表,JSON 列)
- `migrations/000005_home_tabs.down.sql` — DROP 顺序(sub_tabs_version → type_tabs_version → sub_tabs → type_tabs)
- `migrations/000006_home_tabs_seed.up.sql` — 1 + 1 行 seed,JSON 长度:4 tabs + 12 image / 5 video / 3 ideas / 3 flows
- `migrations/000006_home_tabs_seed.down.sql` — 反向清 seed + 重置 version

**seed 数据**(对照 `default.yaml:320-358`):

| typeTab id | label |
|---|---|
| image | 图片模型 |
| video | 视频特效 |
| ideas | 发现灵感 |
| flows | 工作流 |

| parent | subTabs 数 | 首项 |
|---|---|---|
| image | 12 | all |
| video | 5 | all |
| ideas | 3 | all |
| flows | 3 | all |

### 2.3 Server(Go)

| 文件 | 变更 |
|---|---|
| `internal/content/tabs_repo.go` | `TypeTabsRepo` + `SubTabsRepo` 接口 + `MySQL*Repo`(`GetTabs` 单行扫 + JSON 反序列化 + `GetVersion`)。`SubTabsRow` 用临时 struct + `*string` pointer 处理 `viewId` / `externalHref` 的 nullable |
| `internal/content/banner_service.go` | `URLBuilder` 接口不变;`HomeView` 加 `TypeTabs *TypeTabsView` / `SubTabs *SubTabsView` 字段;`Service` 收 4 个 repo;`NewService(repo, recRepo, typeTabsRepo, subTabsRepo, urls)`(5 参数);`buildTypeTabs(ctx)` / `buildSubTabs(ctx)` 拼 mid-shape;`computeETag` 纳入 typeTabs(按数组顺序)+ subTabs(`sort.Strings(byType keys)` 保证 hash 稳定) |
| `internal/content/banner_service.go` 的 mid-shape | 新增 `TypeTabsView{ Tabs + DefaultActive }` / `TypeTabView` / `SubTabsView{ ByType + DefaultActiveByType }` / `SubTabView{ id, label, viewId, params, externalHref }` |
| `cmd/banner-server/main.go` | `content.NewMySQLTypeTabsRepo(sqlDB)` + `NewMySQLSubTabsRepo(sqlDB)` 构造,`NewService(...)` 5 参数;`toOpenAPIHome` 把 `typeTabs: nil` / `subTabs: nil` 换成 `toOpenAPITypeTabs(v.TypeTabs)` / `toOpenAPISubTabs(v.SubTabs)`;marshaler 直接 map 透传 mid-shape,无字段转换(无 MinIO 资源) |

### 2.4 Client(改动 2 个文件,共 ~6 行)

| 文件 | 变更 |
|---|---|
| `src/app/routes/HomePage.tsx` | `useState` 优先 `content.typeTabs.defaultActive`;加 `useEffect` 在首帧空 activeType 时同步到 `defaultActive` 或 `tabs[0]?.id`(保证 subTabs 拿到合法 typeId 渲染) |
| `src/components/layout/SubTabs.tsx` | `useNav()` 调用从 `if (tabs.length === 0) return null` 之后移到之前 — **修一个潜在 hooks-order 错误**:之前 server 返回 null 时永远 early-return 不触发 useNav,server 真源下发后第二次 render 才调 useNav → "Rendered more hooks than during the previous render" |
| `src/api/schema.gen.ts` | `make gen` 重生成(SubTabs.tabs → byType + SubTab.viewId/params/externalHref) |

### 2.5 Tmp 脚本

`tmp/apply_type_tabs_migrations.sh`(沿用 recommend 同款结构):
- 跑 `000005` + `000006`
- `SHOW TABLES LIKE 'home_type_tabs%'` / `LIKE 'home_sub_tabs%'`
- 行数 1/1/1/1
- JSON 长度校验 4 + 12 + 5 + 3 + 3

`tmp/bump_type_tabs_version.sh` + `tmp/bump_sub_tabs_version.sh`:UPDATE 各 version singleton,SELECT 校验。

### 2.6 部署链路

```
MySQL eye_trace (4 表)              MinIO(无 — 纯文本)
  home_type_tabs
  home_sub_tabs
  home_type_tabs_version
  home_sub_tabs_version
              │
              ▼
  cmd/banner-server (Go 1.25)
  GET /api/v1/content/home  :8080
              │
              ▼
  HomePage / ImageModelListPage
  TypeTabs strip + SubTabs strip
  (chrome @ http://localhost:1420/)
```

---

## 3. 排查

### 3.1 现象(接续 recommend 任务)

banner / recommend 都正常后,首页 "一键使用顶尖视频/图片模型" 标题出现,recommend 8 张 card 渲染,但 **typeTabs strip 完全不显示**,store badge `home: server · etag "..."`。

### 3.2 排查步骤

1. **curl server 直击**:`GET /api/v1/content/home` → 200 + JSON,`typeTabs: null`(老 binary 硬编码)
2. **看 `banner-server/main.go` 的 `toOpenAPIHome`**(line 187-188):`typeTabs: nil, subTabs: nil` 硬编码
3. **看 `banner_service.go` `HomeView`**:`HomeView` 没有 TypeTabs / SubTabs 字段
4. **看 `recommend_repo.go`**:`GetMeta` / `ListCards` 模式 → 镜像写 `tabs_repo.go`
5. **看 zod `SubTabsSchema`**(`schemas.ts:378`):`byType: z.record(z.array(SubTabSchema))` — 跟 server 下发要对齐 → `openapi.yaml:316-321` 当前是 `tabs: []` 漂移,**顺手对齐契约**
6. **看 `schema.gen.ts` `SubTabs`**:openapi-typescript 按 yaml 生成,所以也是 `tabs: []` → `make gen` 重生成才能让 client 类型对得上
7. **Server 端编译通过 + DB 迁移跑通 + 重启 binary → curl 200 typeTabs 数据 OK,但 client 仍不渲染**
8. **DOM 排查**:用 React fiber 抓 `HomePage` 的 `useHomeContent()` 结果 → `typeTabs.tabs[0].id === "image"` ✓;再抓 `activeType` state → `null`
9. **看 `HomePage.tsx:30`**:`useState<string>(content?.typeTabs?.tabs[0]?.id ?? "")` — 首帧 `content=undefined`,useState 锁死成 `""`,后续 content 来了也不重新初始化 → `SubTabs` 收到 `typeId=""` 不渲染
10. **加 `useEffect` 同步 defaultActive** → reload → 报 "Rendered more hooks than during the previous render"
11. **追栈**:`at SubTabs (SubTabs.tsx:95)` — `useNav()` 在 `if (tabs.length === 0) return null` 之后调用 → 之前 server 永远返 null 时 SubTabs early-return,不触发 useNav;现在 server 真源下发 → 第二次 render 调用 useNav → hooks 数变化 → React 报错
12. **修**:`useNav()` 上移到 early return 之前 → client 渲染完整 typeTabs + 12 项 subTabs

### 3.3 根因 / 修复表

| 问题 | 根 | 修复 |
|---|---|---|
| `cmd/banner-server/main.go` 把 typeTabs/subTabs 硬编码 null | 一期 Server 只实现 banner + recommend | 新增 2 个 repo + `buildTypeTabs/buildSubTabs` + `toOpenAPITypeTabs/toOpenAPISubTabs` |
| `openapi.yaml` 的 `SubTabs.tabs` 是 flat | 早于 subTab bucket 设计落地的预存漂移 | `tabs → byType`,`SubTab` 加 `viewId/params/externalHref`,`make gen` 重生成 `schema.gen.ts` |
| `HomeContent.required` 7 个字段必填 | 与 zod `nullable().optional()` 冲突 | `required` 收为 `[version]`,其余加 `nullable: true` |
| `HomePage` `useState` 锁死空 activeType | 首帧 content 还没 hydrate,后续 useState 不重新初始化 | 加 `useEffect` 在 `activeType === ""` 时同步到 `defaultActive` 或 `tabs[0]?.id` |
| `SubTabs.tsx` `useNav()` 在 early return 之后 | server 一直返 null 时永远不触发,真源下发后 hooks 数变化 | `useNav()` 上移到 early return 之前 |
| Server ETag 304 缓存 old binary | 老 binary 在 `/tmp/banner-server`,未替换 | 重新编译 `/tmp/banner-server.new` → cp 覆盖,重启 |
| `go build` RTK wrapper 不写文件 | RTK 对某些 output path 有 sandbox 限制 | 直接 `cp /tmp/et/banner-server /tmp/banner-server`,启动用 env var(避免 inline DSN 触发密码暴露规则) |

### 3.4 教训(对比 banner / recommend 任务)

- **契约是单一真相源,任何漂移都是埋雷**:`openapi.yaml` 与 client zod / `schema.gen.ts` 长期不一致导致本次必须重生成 + 类型补全。后续任何 home content 字段改动必须改 openapi → `make gen` → zod 三者同时。
- **JSON 列 vs normalized 表**:纯文本小 N + 无 FK 的 taxonomy 类配置(枚举、分类、tab)直接 JSON 列,省 2-3 张表 + N 行 INSERT。后续若每桶 > 100 项或要做 AB 实验 → 反规范化。
- **ETag hash 必须 `sort.Strings(map_keys)`**:Go map 迭代无序,不排序两次请求 hash 可能不同,触发客户端不必要的 200 重拉而不是 304(recommend 任务里 sub-tabs 那块没出问题是 map 还没存在)。
- **mid-shape 直接 map 透传 vs 中间转换**:无 MinIO 资源时,mid-shape 字段名(openapi)与 view 字段名一致 → `toOpenAPITypeTabs` 直接 `map[string]any{"tabs": v.Tabs, "defaultActive": v.DefaultActive}`,无需 `json.Unmarshal` 反序列化(对照 recommend 的 `json.Unmarshal` 解 params,那是 extra 一层)。
- **hooks 顺序必须在 early return 之前**:`useNav()` / `useNavigate()` / `useRef()` 等 React hook 在组件任何 `if (cond) return null` 之后调用会触发 "Rendered more hooks" 错误。当 server 从"永远返 null"变成"正常下发"时,组件从 early-return path 切到正常 path → hooks 数差异立刻爆炸。这是 server 数据驱动 UI 的隐藏地雷。
- **client useState 默认值要看真实首帧**:`useState(content?.X?.y ?? "")` 在 content 还没 hydrate 时锁死空值。server 明确下发 `defaultActive` 等"派生默认值"字段比 client 计算更稳。
- **RTK 编译路径限制 + 密码内嵌规则**:Go 二进制必须落在 `/tmp/` 类 sandbox 可见路径;DB 密码不能 inline 在命令行 → 用 env var 注入(`MYSQL_PWD` 或 `DB_DSN` 由外部脚本导出)。

---

## 4. 完成

### 4.1 浏览器最终态(`tmp/after-type-tabs-final.png` / `tmp/clicked-video-tab.png`)

**默认载入(图片模型 active)**:
```
[LibLibAI 顶部 bar]
[banner 4 块:作者收益 / 图片视频生成 / LibTV / 开工工具]
一键使用顶尖视频/图片模型  [Seedance 2.0 低至 0.35元/秒…] ›     ← 推荐标题 + marquee + caret
  [Sd] Seedance 2.0       [Sd] Seedance 2.0 Fast  [H] Happy Horse 1.0  [▣] 智能图片V2   →
       最强视频模型…             最强视频模型…          阿里最新视频…           最新图片模型…
       NEW 角标                NEW 角标

图片模型 | 视频特效 | 发现灵感 | 工作流     ← 顶层 typeTabs(图片模型蓝色下划线)
全部  🤩每周新选·图片模板  🔥爆款Top100模型  摄影写真  电商营销  动漫游戏  风格插画  平面设计  建筑及室内设计  创意玩法  文创周边  小说推文  →    ← 12 项 subTabs(全部灰底 active)
                                                                    推荐 ▾ 筛选 ▾

[加载失败 GET /models?limit=12&sort=latest]   ← 不在本任务范围

store badge: home: server · etag "0bf4fb83a..."
```

**点击 "视频特效"**:
```
图片模型 | 视频特效 | 发现灵感 | 工作流     ← 视频特效蓝色下划线 active
全部  🆕最新上线  🔥热门视频模型  人像写真  短剧特效                              ← 5 项 video subTabs
```

12 项 ↔ 5 项 ↔ 3 项 ↔ 3 项 的 subTabs swap 工作正常,首项 "全部" 由 `defaultActiveByType[parent]` 自动重置。

### 4.2 数字

| 维度 | 数 |
|---|---|
| DB 表 | 4(2 singleton + 2 version)|
| DB 行 | 1 / 1 / 1 / 1(全是 singleton)|
| typeTabs 项 | 4 |
| subTabs 项 | 12 + 5 + 3 + 3 = 23 |
| MinIO 对象 | **0**(纯文本,无资源)|
| 契约 schema 改动 | `HomeContent.required` 收为 `[version]` + `SubTabs.tabs → byType` + `SubTab` 加 3 可选字段 + `TypeTab/SubTab` 加 `maxLength` |
| Server LOC 新增 | `tabs_repo.go` 178 行 + `banner_service.go` +95 行 + `main.go` +18 行 |
| 客户端代码改动 | **2 个文件 ~6 行**(`HomePage.tsx` + `SubTabs.tsx`)+ `make gen` 自动同步 `schema.gen.ts` |

### 4.3 后续(server 重构那条线接手)

| 项 | 备注 |
|---|---|
| 给 SubTab 填 viewId(目前全 NULL) | 接入 "本周新选" / "摄影写真" 等真实子页路由时填进 `home_sub_tabs.by_type_json.*[].viewId` |
| normalized 反规范化 | 若后续每桶 > 100 项或要做 AB 实验 → 拆 `home_sub_tab_items(parent_type_id, position)` |
| admin UI 写接口 | 当前运维 SQL 手动,后续接 content admin(server refactor line)|
| YAML fallback 评估 | server 稳定 2 周后,删 `default.yaml` 的 `typeTabs/subTabs` 块,走 server-only(与 banner / recommend 一致)|
| `schema.gen.ts` vs zod 一致性 | 本次修复一次,但 openapi-typescript 仍按 yaml 自动生成 — 后续任何 home content 字段必须 `make gen` 后再交 zod。CLAUDE.md 加铁律:`新增 / 修改 HomeContent 字段 → 必须 openapi.yaml → make gen → zod 三者同步,缺一步会被契约漂移打到` |
| JSON 列 upgrade 路径 | `home_sub_tabs.by_type_json` 在 mysqldump / replication 下保持稳定,但 5.7 → 8.0 升级建议做一次 roundtrip 校验 |

### 4.4 文件清单(全部交付)

```
eye-trace-docs/
├── openapi.yaml                                        # HomeContent.required + SubTabs/SubTab
└── home-type-tabs-sub-tabs-flow.md                     # 本文档

eye-trace-server/backend/
├── migrations/000005_home_tabs.up.sql                  # 4 表 DDL
├── migrations/000005_home_tabs.down.sql                # 反向 DROP
├── migrations/000006_home_tabs_seed.up.sql             # JSON 种子
├── migrations/000006_home_tabs_seed.down.sql           # 反向清 seed
├── internal/content/tabs_repo.go                       # TypeTabsRepo + SubTabsRepo + MySQL 实现
├── internal/content/banner_service.go                  # +TypeTabsView/SubTabsView + buildTypeTabs/buildSubTabs + ETag sort.Strings
└── cmd/banner-server/main.go                           # +2 repo 注入 + toOpenAPITypeTabs/toOpenAPISubTabs

eye-trace-client/
├── src/app/routes/HomePage.tsx                         # useState 优先 defaultActive + useEffect 同步
├── src/components/layout/SubTabs.tsx                   # useNav() 上移到 early return 之前(修 hooks order)
└── src/api/schema.gen.ts                               # make gen 自动重生成(SubTabs.tabs → byType + SubTab.viewId)

tmp/
├── apply_type_tabs_migrations.sh                       # 迁移辅助
├── bump_type_tabs_version.sh                           # ETag bump
├── bump_sub_tabs_version.sh                            # ETag bump
├── after-type-tabs-final.png                           # 默认 active 截图
└── clicked-video-tab.png                                # 点击视频特效后截图
```
