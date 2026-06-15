# Client 数据拆分指导:运营数据 vs 配置数据

> 目标:把 client 现在混在 `default.yaml` 里的"配置"和"运营内容"彻底分家,
> 让渲染组件不关心数据来自 YAML 还是 server API。
> 本文基于对 `src/components/layout/` 现有组件的实地核对,非推测。
>
> 相关文档:`openapi-skeleton.md`(server 契约)、`go-server-structure.md`(server 结构)。

---

## 0. 核心判定标准

一个字段属于哪一类,只问一个问题:**"它变了,要不要重新打包发版?"**

| 类别 | 定义 | 变更方式 | 存放 |
|------|------|---------|------|
| **配置 (config)** | 部署期/平台期决定,几乎不变 | 改 YAML → 重新 build | 永远留 `default.yaml` + 平台覆盖 |
| **分类 (taxonomy)** | 变更低频,但属于"内容结构" | 短期 YAML,长期 server | 过渡期 YAML,后期 API |
| **运营内容 (content)** | 随活动/运营/用户随时变 | **绝不能靠发版**,必须 server | server API,YAML 仅离线兜底 |

**口诀**:URL/超时/平台开关/窗口尺寸 = 配置;模型卡/Banner/优惠/跑马灯 = 运营内容;页签分类 = taxonomy(介于两者)。

---

## 1. 6 个 class 片段 → 组件 → 数据源 实测映射

| # | class 片段(节选) | 组件文件 | 读取的 config 节点 | 迭代源(.map) | 类别 |
|---|---|---|---|---|---|
| 1 | `flex w-full gap-2 my-4 md:mt-0 md:mb-6` | `DynamicBannerCarousel.tsx` | `appConfig.bannerCarousel` | `config.carousel` + `config.sideCards` | **运营** |
| 2 | `relative h-full w-full cursor-pointer` | `DynamicBannerCarousel.tsx`(单卡) | `bannerCarousel.carousel[]` / `sideCards[]` | `slides.map` / `cards.map` | **运营** |
| 3 | `...rounded-lg pl-4 pr-3 ... bg-[#FFF7EA]` | `RecommendBanner.tsx` L111 | `appConfig.recommend.announcement` | `announcement.text` 逐字 `chars.map` | **运营** |
| 4 | `grid w-full` | `ModelItemList.tsx` L402 | `appConfig.modelItems` | `items.map` | **运营(UGC + admin 双上传)** |
| 5 | `sticky z-40 flex gap-2 ... after:...` | `TypeTabs.tsx` L25 | `appConfig.typeTabs` | `tabs.map` | **taxonomy** |
| 6 | `sticky z-40 flex items-center justify-between ...` | `SubTabs.tsx` L57 | `appConfig.subTabs` | `byType[typeId].map` | **taxonomy** |

> "配了几个展示几个" = 数组长度驱动渲染,5 个组件全部已是这个模式:空数组/缺字段则隐藏整块。

---

## 2. 关键现状:双源钩子已存在(好消息)

实地核对发现,**5 个组件已支持 `config?` prop 注入**,采用"默认读 appConfig,可被 prop 覆盖"模式:

```tsx
// 现有写法(DynamicBannerCarousel.tsx 第 17/168 行)
const cfg = appConfig.bannerCarousel;                          // 默认源 = 静态 YAML
export function DynamicBannerCarousel({ config = cfg }: { config?: BannerCarouselConfig }) { ... }
```

| 组件 | 支持 config prop | fallback 到 appConfig |
|------|:---:|:---:|
| DynamicBannerCarousel | ✅ | ✅ |
| RecommendBanner | ✅ | ✅ |
| ModelItemList | ✅ | ✅ `config?.items ?? appConfig.modelItems?.items ?? []` (L385) |
| TypeTabs | ✅ | ⚠️ **缺失** — `config?.tabs ?? []`(L25),不传就空白 |
| SubTabs | ✅ | ⚠️ 半缺 — `config?.byType ?? {}` (L29) |

**结论**:迁移到 server 不需要改组件渲染逻辑,只需在上层统一注入 config prop。
但 **TypeTabs / SubTabs 的 fallback 不一致,迁移前必须补齐**(见 §4.2)。

---

## 3. 目标架构:统一内容加载层 `loadHomeContent()`

在 `src/api/` 下新建内容加载器,作为**所有运营内容的唯一入口**。组件只认类型,不认来源。

```
src/
├── config/static/
│   └── default.yaml      # 只剩:apiBaseUrl/features/ui(非vipPromo)/request/window
│                         # bannerCarousel/recommend/modelItems/tabs 降级为 fallback 数据
├── api/
│   ├── client.ts         # fetch 封装(base url、超时、鉴权 header)
│   ├── content.ts        # ★ loadHomeContent():API 优先,失败回退 YAML
│   └── schema.gen.ts     # openapi-typescript 生成(勿手改)
└── stores/
    └── content.ts        # Zustand:持有 HomeContent,组件订阅
```

`content.ts` 核心逻辑:

```ts
import { appConfig } from "@/config/static";
import type { HomeContent } from "./schema.gen";
import { HomeContentSchema } from "./schema.gen";   // zod 校验

// 离线兜底:复用 YAML 现有 demo 数据,形状与 API 响应完全一致
const fallback: HomeContent = {
  bannerCarousel: appConfig.bannerCarousel,
  recommend:      appConfig.recommend,
  modelItems:     appConfig.modelItems,
  typeTabs:       appConfig.typeTabs,
  subTabs:        appConfig.subTabs,
  vipPromo:       appConfig.ui.vipPromo,
};

export async function loadHomeContent(): Promise<HomeContent> {
  try {
    const res = await apiClient.get("/content/home");  // GET /api/v1/content/home
    return HomeContentSchema.parse(res);               // 运行时校验,防契约漂移
  } catch {
    return fallback;                                   // server 不可达 → demo 数据
  }
}
```

上层注入(AppShell 或首页):

```tsx
const content = useHomeContent();   // 从 Zustand store 读
<DynamicBannerCarousel config={content.bannerCarousel} />
<RecommendBanner       config={content.recommend} />
<TypeTabs              config={content.typeTabs} onTypeChange={setActiveType} />
<SubTabs               config={content.subTabs} typeId={activeType} />
<ModelItemList         config={content.modelItems} />
```

**收益**:server 建好后只改 `loadHomeContent()` 一处,渲染层零改动;server 未建时一切照旧走 YAML。

---

## 4. 逐字段拆分清单

### 4.1 永远留在 YAML 的"真配置"

```yaml
apiBaseUrl              # 平台相关,已有 website/macos/windows 覆盖,编译期注入
features:               # 平台开关(showCachePanel/nativeMenu/trayIcon/...)
ui.maxPromptLength      # 表单输入约束
ui.defaultImageSize     # 默认画布尺寸
ui.supportedSizes       # 合法生成尺寸(同时进 eye-trace-config/shared 枚举)
ui.inputRadius          # 纯样式 token
request.timeoutMs       # HTTP 超时
request.pollIntervalMs  # 轮询间隔
window.minWidth/Height  # 桌面窗口约束
```

### 4.2 迁出到 server 的"运营内容"

| 现 YAML 节点 | 目标端点 | 谁改 | 变更频率 |
|---|---|---|---|
| `ui.vipPromo` | `GET /content/home` → `vipPromo` | admin 后台 | 随活动变,高频 |
| `recommend.announcement` | `GET /content/home` → `recommend.announcement` | admin | 高频 |
| `recommend.models[]` | `GET /content/home` → `recommend.models` | admin | 中频 |
| `bannerCarousel.*` | `GET /content/home` → `bannerCarousel` | admin | 中频(广告位) |
| `modelItems.items[]` | `GET /models` + `GET /content/home` → `modelItems` | admin 上架 + 用户 UGC | 高频 |

### 4.3 taxonomy(过渡期留 YAML,后期迁 server)

| 现 YAML 节点 | 短期处理 | 长期目标 |
|---|---|---|
| `typeTabs.tabs` | 留 YAML | `GET /content/home` → `typeTabs` |
| `subTabs.byType` | 留 YAML | `GET /content/home` → `subTabs` |

### 4.4 必须立即修复的 fallback 不一致(TypeTabs / SubTabs)

```tsx
// TypeTabs.tsx L25 — 现在
const tabs = config?.tabs ?? [];                        // ❌ 不传就空白

// 修复为
const tabs = config?.tabs ?? appConfig.typeTabs?.tabs ?? [];  // ✅ 与其它组件对齐

// SubTabs.tsx L29 — 现在
const byType = config?.byType ?? {};                    // ❌ 不传就空

// 修复为
const byType = config?.byType ?? appConfig.subTabs?.byType ?? {};  // ✅
```

### 4.5 进 `eye-trace-config/shared` 的共享枚举(client + server 双读)

```yaml
# eye-trace-config/shared/enums.yaml
supportedImageSizes:       # client 表单校验 + server 拒绝非法尺寸
  - [512, 512]
  - [768, 768]
  - [1024, 1024]

typeTabIds:                # client 路由 + server 按此归类模型
  - image
  - video
  - ideas
  - flows

jobStatus:                 # 见 openapi-skeleton.md
  - pending
  - running
  - succeeded
  - failed
  - cancelled
```

---

## 5. 模型数据统一格式(UGC + admin 双上传)

### 5.1 两种形状不要混淆

| 形状 | 用途 | 谁持有 |
|------|------|--------|
| `ModelUploadRequest` | **上传时提交**,资源元数据 | 用户/admin → server |
| `ModelItemCard`(现有 types.ts) | **渲染时展示**,含 variant/cover/previews 等 | server → client |

**不要让用户直接提交 `ModelItemCard`**。展示卡是 server 从 `ModelRecord` 派生的,client 不应感知派生逻辑。

### 5.2 统一上传格式(加入 openapi-skeleton.md 的 components.schemas)

```yaml
ModelUploadRequest:
  type: object
  required: [title, modelType, assetId]
  properties:
    title:
      type: string
    modelType:
      type: string
      enum: [checkpoint, lora, template, workflow, video]
    assetId:
      type: string                   # 先 POST /assets 上传文件,拿到 assetId
    coverAssetId:
      type: string                   # 封面图,同走 POST /assets
    previewAssetIds:
      type: array
      maxItems: 3
      items: { type: string }        # 预览图(最多 3 张)
    description:
      type: string
    tags:
      type: array
      items: { type: string }
    baseModel:
      type: string                   # 底模标识:sd15 / sdxl / flux / ...
    params:
      type: object
      additionalProperties: true     # 默认生成参数,由模型决定字段
    # source 字段由 server 根据 token 角色注入,客户端传了也会被覆盖
    # source: user | admin
```

### 5.3 同一端点,靠 token 角色区分来源

| 来源 | 端点 | 鉴权 | server 行为 |
|------|------|------|------------|
| 前端用户 UGC | `POST /api/v1/models` | 用户 JWT | 注入 `source=user`,进审核队列 |
| admin 后台 | `POST /api/v1/models` | admin JWT | 注入 `source=admin`,可跳过审核 |

**一套 `ModelUploadRequest` + 一个端点 + token 角色鉴权** = 格式天然一致,不开两套接口。

### 5.4 server 派生展示卡的规则

server 将 `ModelRecord` 映射成 client `ModelItemCard` 时的 variant 决策:

| 条件 | 派生 variant |
|------|-------------|
| `previewAssetIds.length >= 1` | `preview-deck` |
| `assetId` 指向视频(mp4/webm) | `single` + `mediaType: video` |
| `assetId` 指向 gif | `single` + `mediaType: gif` |
| gallery 多图模型(特殊标记) | `gallery` |
| 其他 | `single`(默认) |

`cover` / `previews` / `gallery[].src` 全部解析为 `/api/v1/assets/{id}/content` 的可访问 URL。
client 渲染层完全复用现有 `ModelItemList`,无需感知来源是 UGC 还是 admin。

---

## 6. 落地顺序

按依赖关系,不要跳步:

1. **YAML 注释分区**(不改代码):在 `default.yaml` 顶部加注释 `# === 配置(勿动) ===` 和 `# === 运营内容(离线 demo,将迁 server) ===`,让边界可见。
2. **修 TypeTabs/SubTabs fallback**(§4.4,2 行改动):消除 5 个组件的行为不一致。
3. **建 `src/api/content.ts`**:实现 `loadHomeContent()`,fallback 指向现有 YAML。server 未建时功能完全不变。
4. **改 AppShell 注入方式**:从 `useHomeContent()` 取数据,显式传 `config` prop(组件已支持,改调用方即可)。
5. **server `/content/home` 就绪后**:把 YAML 里的 demo 数据移入 server DB,`loadHomeContent()` 开始返回真数据,YAML 永久降级为离线兜底。
6. **模型上传**:补 `POST /models` 端点 + `ModelUploadRequest` schema,admin 后台和前端 UGC 接同一个接口。
