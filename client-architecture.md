# EyeTraceAI Client 架构设计 v1

> 版本: 1.0 | 日期: 2026-06-12 | 状态: 规范(Phase 1 实施蓝本)
> 范围: `eye-trace-client/`(web + desktop 双形态,单一 codebase)
> 上位文档: [`project_guide.md`](./project_guide.md) §2 Phase 1
> 平行文档: [`server_guide.md`](./server_guide.md)、`client_data_split.md`、[`openapi-skeleton.md`](./openapi-skeleton.md)

本文是 Phase 1 客户端仿站完工阶段的**架构唯一真相源**。它不重写既有的 `client-frontend-roadmap.md` 与 `liblib-client-function-map.md`,而是把两份分卷的结论沉淀到一张"目录契约 + 14 路由外壳 + 接入规则"上,作为后续 PR 拆分的依据。

---

## 目录

1. 目标与边界
2. 与既有分卷的契约
3. 形态与发布矩阵
4. 目录骨架与模块边界
5. 14 路由的页面外壳清单
6. AppShell 形态选择器(已落地,固化)
7. 数据获取与回退(已落地,固化)
8. UI 原语与组件层级
9. 设计 token 与断点(本阶段新增)
10. 仿站原语登记(从 liblib 抽取)
11. 平台桥(Desktop vs Web)
12. 命名迁移
13. 测试策略
14. Phase 1 完成判据(细化)
15. 后续 PR 拆分建议

---

## 1. 目标与边界

### 1.1 Phase 1 目标(从 `project_guide.md` §2.1 摘出)

- 产出一个**完整可导航的产品外壳**,14 条路由均可点击进入并展示出有数据感的 UI;
- 全部运营内容由 `default.yaml` 驱动,**除 `/content/home` 外不发起任何真实后端请求**;
- `EyeTraceAI` / `eyetrace://` 品牌与 deeplink 完成替换;
- 6 个生成页(/ai-tool/image-generator、/ai-tool/video-generator、/sd、/comfy、/pretrain、/lib3)的"生成"按钮接入 `fsm` 并能跑通 `idle→selectingModel→configuring→generating→success→error`(进度由 mock 计时器驱动);
- Page Registry / ViewStack / KeepAlive / LayerManager 四件套保持原状,不重写。

### 1.2 本阶段**不**做的事

- 不接任何除 `/content/home` 之外的真实接口;
- 不做鉴权(Authorization 头在 `client.ts` 仍以 TODO-Auth 注释保留);
- 不做多租户、不做配额;
- 不引入新的状态机 / 路由 / UI 库;
- 不动 `go-server-structure.md` 任何 server 端决策(本文只描述 client 如何*准备*对接)。

### 1.3 决策原则

- **不重写既有代码**:`@/app/registry`、`@/app/stack`、`@/app/keepalive`、`@/app/layer`、`@/app/nav` 已落地且有测试覆盖,Phase 1 只消费它们的接口,不修改实现。
- **样式沿用 Tailwind + shadcn/ui primitive 风格**(与 `tailwind.config.js` 的 HSL token、`globals.css` 的 `--color-background` 等保持一致);所有数值类 token(尺寸、间距、颜色)在 §9 收敛,页面层只允许使用 token。
- **数据契约**:运营内容数据形状固化在 `src/config/static/types.ts`;`HomeContent` zod schema 复用同一份类型。改数据时只动 `default.yaml` 和 `types.ts`。
- **品牌**:本轮一次性把 "Game AI Asset Studio" / "gameai" / "studio.gameai.asset" 改为 "EyeTraceAI" / "eyetrace" / "ai.eyetrace.app",与 `project_guide.md` §9 对齐。

---

## 2. 与既有分卷的契约

| 既有分卷 | 关系 | 本文档如何对待 |
|---|---|---|
| `client-frontend-roadmap.md`(索引) | 已落地 4 个 UI 原语 + 5 步路线 | 视为已生效结论,本文不复述 |
| `01-tech-stack.md`(Tauri 2 + React 18 + Vite 5 + Zustand + Tailwind) | 基座 | 本文不重新选型 |
| `02-config-isolation.md` | 静态配置 2 层浅覆盖 | §7 沿用,扩展运营内容 yaml 段 |
| `03-platform-layer.md` | `platforms/{web,desktop}` 平台抽象 | §11 沿用 |
| `04-ui-management.md` | Page Registry / ViewStack / KeepAlive / LayerManager / Command Bus | §5 §6 引用接口,不重述内部实现 |
| `05-landing-plan.md` | 6 步落地路线 | 本文是 M5 之后的工作分解,§15 给出 PR 拆分 |
| `liblib-client-function-map.md` | 14 路由 + 6 区域清单 | §5 把每条路由映射到具体 page + 配置段 |
| `client_data_split_guide.md` | 数据分类(运营/契约/管线/隐私) | §7 沿用"server 后置 + YAML 兜底" |
| `project_guide.md` §2 Phase 1 | 本阶段交付集合 | §14 给细化判据 |

---

## 3. 形态与发布矩阵

| 形态 | 命令 | 入口 | `__PLATFORM__` | 平台抽象 |
|---|---|---|---|---|
| web | `npm run dev` / `npm run build` | `index.html` 静态 | `web` | `platforms/web/*`(no-op) |
| macOS 桌面 | `npm run tauri:dev` / `tauri:build -- --target aarch64-apple-darwin` | Tauri WebView 加载 devUrl/`frontendDist` | `macos` | `platforms/desktop/*` |
| Windows 桌面 | 同上,换 `--target x86_64-pc-windows-msvc` | 同上 | `windows` | 同上 |

三形态共用同一份 `src/`,仅 `vite.config.ts` 注入的 `__PLATFORM__` 与 `tauri.conf.json` 资源不同。本轮**不动**发布矩阵,只更新品牌字符串。

---

## 4. 目录骨架与模块边界

```
eye-trace-client/
├── src/
│   ├── api/                  # 运行时数据获取(client.ts + content.ts + sse.ts [新增])
│   ├── app/                  # 框架层 —— 不允许放业务
│   │   ├── auth/             # 鉴权闸 + 受保护视图守卫
│   │   ├── keepalive/        # Selective KeepAlive(已落地,固化)
│   │   ├── layer/            # LayerManager + OverlayHost
│   │   ├── nav/              # useNav + NavLink
│   │   ├── registry/         # Page Registry 真相源
│   │   ├── routes/           # 14 路由的 React 组件
│   │   └── stack/            # ViewStack
│   ├── components/
│   │   ├── icons/            # SVG 图标 primitive
│   │   ├── layout/           # AppShell/NavBar/TopBar/CenterBar/DynamicBannerCarousel/...
│   │   └── ui/               # shadcn-style primitive(本阶段按需补)
│   ├── config/
│   │   ├── i18n/             # i18next 资源
│   │   └── static/           # default.yaml + web/macos/windows 覆盖
│   ├── features/             # 横切特性(generate FSM / models / prompt / cache)
│   ├── hooks/                # 通用 hook
│   ├── lib/                  # 通用工具(cn 等)
│   ├── platforms/
│   │   ├── desktop/          # Tauri 桥(已落地)
│   │   └── web/              # no-op(已落地)
│   ├── stores/               # Zustand store(目前仅 content store)
│   ├── styles/               # 全局 CSS(globals.css + tailwind)
│   └── types/                # 业务类型(目前为空,类型多在 config/static/types)
├── src-tauri/                # Tauri 2 Rust 入口(本轮仅改品牌)
├── public/ + res/            # 静态资源
└── docs/                     # 历史/参考
```

**模块边界硬规则**:

1. `app/` 不允许依赖 `features/`、`components/`、`pages/`(框架与业务反向)。
2. `app/routes/` 单页文件**禁止跨页 import 组件**;跨页复用必须先抽到 `components/layout/` 或 `features/`。
3. `components/ui/` 只放 shadcn 风格无业务 primitive(`Input`、`Button` 等),不加 liblib 文案。
4. `features/generate/fsm.ts` 是**纯函数**,任何页面订阅都用 `useGenerateMachine`。
5. `config/static/types.ts` 是**所有 yaml 段形状的真相源**;`api/content.ts` 的 zod schema 必须复用此类型。

---

## 5. 14 路由的页面外壳清单

路由总清单与 liblib 仿站目标的对应关系。每行都明确:"哪个 viewId × 哪个 page 组件 × 哪个 yaml 段 × Phase 1 是否需要新增"。已存在的文件打 ✓,需要新增/重写的打 NEW。

| # | viewId / path | page 文件 | 状态 | yaml 段 | 备注 |
|---|---|---|---|---|---|
| 1 | `home` / `/` | `HomePage.tsx` | ✓ | `bannerCarousel / recommend / modelItems / typeTabs / subTabs / vipPromo` | `/content/home` 已接 |
| 2 | `image.model.list` / `/image-model` | `ImageModelListPage.tsx` | ✓ | 复用 `modelItems` + 本地 12 chip 列表 | 子分类 chip 列表已硬编码在页面内,候选:提到 `default.yaml.pages.subChips` |
| 3 | `search` / `/search` | `SearchPage.tsx` | ✓ | `pages.search` | 5 tab + 结果卡 + 筛选入口 |
| 4 | `tool.image` / `/ai-tool/image-generator` | `ImageGeneratorPage.tsx`(壳)→ `ToolLandingPage` / `ToolWorkspacePage` | ✓ | `pages.toolWorkspaces.image` | 接入 FSM + 6 个生成页外壳(见 §5.A) |
| 5 | `tool.video` / `/ai-tool/video-generator` | `VideoGeneratorPage.tsx`(壳)→ `ToolLandingPage` | ✓ | `pages.toolWorkspaces.video` | 同上 |
| 6 | `sd` / `/sd` | `WebUIPage.tsx`(壳)→ `ProtectedWorkspacePage`(`pageKey="sd"`) | ✓ | `pages.protectedWorkspaces.sd` | 走 `SdRouteShell`(登录前/后两态) |
| 7 | `comfy` / `/comfy` | `ComfyPage.tsx`(壳)→ `ProtectedWorkspacePage`(`pageKey="comfy"`) | ✓ | `pages.protectedWorkspaces.comfy` | 通用工作区外壳(SD 专属编辑器组件已写,Comfy 复用) |
| 8 | `pretrain` / `/pretrain` | `PretrainPage.tsx`(壳)→ `ProtectedWorkspacePage`(`pageKey="pretrain"`) | ✓ | `pages.protectedWorkspaces.pretrain` | 通用工作区外壳 |
| 9 | `lib3` / `/lib3` | `AiAppsPage.tsx` | ✓ | `pages.aiApps` | 应用分类 + 右侧 Runner 面板(已落地) |
| 10 | `asset` / `/asset` | `AssetsPage.tsx` | ✓ | `pages.assets` | 产品 tab + 类型 + 收藏 + 排序(已落地) |
| 11 | `model.detail` / `/modelinfo/:id` | `ModelDetailPage.tsx` | ✓ | `pages.modelDetails[id]` | 详情页(`destroyOnLeave`) |
| 12 | `image.detail` / `/imageinfo/:id` | `ImageDetailPage.tsx` | ✓ | `pages.imageDetails[id]` | 详情页(`destroyOnLeave`) |
| 13 | `teaching` / `/teaching` | `TeachingPage.tsx` | ✓ | `pages.teaching` | 教学列表 + 右侧文章预览 |
| 14 | `userpage.publish` / `/userpage/me/publish` | `UserPublishPage.tsx` | ✓ | 无 yaml(本地 mock 列表) | 用户中心(已落地) |
| 15 | `apis` / `/apis` | `ApiLandingPage.tsx` | ✓ | `pages.apiLanding` | API 落地页(已落地) |

> 注:`/teaching/:id` 教学详情页在 liblib 清单中是子路由,Phase 1 在 `TeachingPage` 内通过右侧 `ArticlePreview` 模拟(`articles[id]` 段已支持),后续再拆独立路由。是否拆独立路由在 Phase 2 验收时再定。

### 5.A 6 个生成页外壳("做同款"路径)

6 个生成页共享同一 `ToolWorkspaceConfig` 形状(已定义在 `types.ts`):title / subtitle / activeMode / modes / promptPlaceholder / uploadChips / modelLabel / modelName / parameterChips / inspirations[] / history[]。

Phase 1 模板对照:

| 路由 | `toolWorkspaces.key` | 是否需要登录(`toolShell.loggedIn`) | 备注 |
|---|---|---|---|
| `/ai-tool/image-generator` | `image` | 否,登录后切换为底部固定条 | `ToolLandingPage` 已分流 `loggedIn ? ToolLoggedInWorkspace : ToolLandingWorkspace` |
| `/ai-tool/video-generator` | `video` | 同上 | 同上 |
| `/sd` | `sd` | 是(走 `SdRouteShell`) | 登录前展示 `LoginRequiredState` + 登录后展示 `SdLoggedInWorkspace` |
| `/comfy` | `comfy` | 是(走 `protectedNav.ts`) | 登录后展示通用工作区预览 |
| `/pretrain` | `pretrain` | 是(走 `protectedNav.ts`) | 同上 |
| `/lib3` | — | 否 | `AiAppsPage` 不消费 `toolWorkspaces` |

`/sd` 单独成壳(`SdLoggedInWorkspace` 已实现编辑器头部 + tabs + prompt 双框 + 参数面板 + 队列面板 + 灵感库),Comfy / Pretrain 用通用壳(`GenericWorkspacePage`)。Phase 1 不需要为 Comfy / Pretrain 写专属 UI。

---

## 6. AppShell 形态选择器(已落地,固化)

`AppShell` 已根据当前 path 在三套壳中切换(`AppShell.tsx:467-472`):

| 路由集合 | 壳 |
|---|---|
| `/ai-tool/image-generator`, `/ai-tool/video-generator` | `ToolRouteShell`(窄 68px 图标侧栏 + 顶部 12px 青色活动条 + 顶部右上 actions) |
| `/sd` | `SdRouteShell`(登录前=活动条+登录按钮;登录后=顶部 68px 工作区头部) |
| 其他 | 默认壳(NavBar + TopBar + CenterBar) |

**新增路由时不需要再改 `AppShell.tsx`**,除非新路由需要完全不同的 chrome(例如全屏绘图)。若新增全屏工作区,先在 `AppShell.tsx:22-26` 的 `TOOL_SHELL_ROUTES` / `SD_SHELL_ROUTES` 集合里加路径,再写一套壳。

---

## 7. 数据获取与回退(已落地,固化)

`api/content.ts` 的 `loadHomeContent` 是当前唯一的"远端 + 回退"模式。Phase 1 其它页面**继续**走 `useHomeContent()` + `default.yaml` 的方式,**不再发起任何 fetch**。每个非 home 页面用局部 `useState` 模拟"我已选中哪个 tab / 哪个子分类"等瞬时 UI 状态,不上 Zustand。

本阶段唯一的新增文件:`api/sse.ts`(stub)。详见 `project_guide.md` §6。本 stub 在 Phase 1 返回 mock 事件流(计时器模拟 `progress→succeeded`);`useGenerateMachine` 通过 `subscribeJobEvents` 间接消费,UI 形态与 Phase 2 真实 SSE 接入完全一致。

---

## 8. UI 原语与组件层级

### 8.1 既有 4 原语(已落地,固化)

| 原语 | 实现 | 入口 |
|---|---|---|
| Page Registry | `app/registry/pages.ts` | `ViewHost` 通过 `useViewStackSync` 读 |
| ViewStack | `app/stack/stack.ts` | `useNav` / `useViewStackSync` |
| Selective KeepAlive | `app/keepalive/KeepAlive.tsx` | `<KeepAliveHost/>` 在 AppShell 层 |
| LayerManager | `app/layer/manager.ts` + `OverlayHost.tsx` | 任何业务通过 `useLayerStore` 触发 |

### 8.2 组件层级(自顶向下)

```
AppShell
├── NavBar                 (liblib 200/68px 侧栏)
├── TopBar / ToolPromoBar / SdTopBar  (按壳切换)
├── main
│   ├── TypeTabs           (sticky 80px)
│   ├── SubTabs            (sticky 134px)
│   ├── ModelItemList      (6 列网格)
│   ├── DynamicBannerCarousel / RecommendBanner+List
│   ├── CenterBar          (3 态底栏,展开式 prompt)
│   └── <路由内容>          (14 routes)
└── OverlayHost            (modal/drawer/preview/confirm/toast)
```

### 8.3 本阶段新增 / 补强

- `components/ui/Button.tsx` — shadcn 风格,变体 `default / primary / secondary / ghost / danger`;`RouteActionButton` 在本阶段迁过去消费(避免每个页面重复写 `cn`)。**非必须,若 PR 容量紧可推迟**。
- `components/ui/Card.tsx` — 圆角 2xl 卡片 + 阴影 + 暗色 token 收敛,详情页/工作区复用。
- `features/generate/` 保持原样,`useGenerateMachine` 已被 `ToolLandingWorkspace` 间接消费(本期补 onClick 接线)。

---

## 9. 设计 token 与断点(本阶段新增)

为了"95% 仿站"在长尾迭代时不被样式碎片化拖垮,本阶段在 `styles/globals.css` 与 `tailwind.config.js` 各引入一组 token,所有页面**只允许**消费 token,不允许硬编码 hex。

### 9.1 颜色 token(在 `globals.css` 扩展)

| token | light | dark | 用途 |
|---|---|---|---|
| `--eyetrace-bg` | `#FAFAFA` | `#0D0D0D` | AppShell 主背景(已部分使用) |
| `--eyetrace-bg-soft` | `#F2F3F5` | `#171717` | 卡片底、chips |
| `--eyetrace-bg-card` | `#FFFFFF` | `#171717` | 白色卡片底 |
| `--eyetrace-border` | `#EBEBEB` | `#2A2D3D` | 卡片描边 |
| `--eyetrace-text` | `#18191C` | `#FFFFFF` | 主文本 |
| `--eyetrace-text-muted` | `#666666` | `#FFFFFFB2` | 副文本 |
| `--eyetrace-text-faint` | `#999999` | `#FFFFFF80` | 三级文本 |
| `--eyetrace-primary` | `#1F6DFF` | `#1F6DFF` | 主蓝(已大面积使用) |
| `--eyetrace-primary-soft` | `#EAF2FF` | `#1F6DFF29` | 蓝色 chip 背景 |
| `--eyetrace-accent-orange` | `#FF8000` | `#FFB264` | 邀请有礼 / 营销橙 |
| `--eyetrace-accent-pink` | `#FF4D27` | `#FF4D27` | VIP 折扣角标 |

> 历史 hex 在 14 路由的源文件里大量存在(`#18191C` / `#1F6DFF` / `#EBEBEB` / `#F2F3F5` / `#FAFAFA` 等),本阶段**不**强制迁移,但新增 / 改动的样式必须走 token。批量迁移在 Phase 2 视觉走查时统一处理。

### 9.2 断点

- 已有:`b768`(768px,镜像 liblib `b768:*` 媒体查询)。
- 新增:`b1024`(1024px,主内容容器切换 2 列→3 列的分界);`b1280`(6 列网格完整展现的最小宽度)。
- 在 `tailwind.config.js` 的 `screens` 中按需追加,避免改写既有断点。

### 9.3 尺寸 token(在 tailwind extend 中)

| token | 值 | 用途 |
|---|---|---|
| `rounded-card` | 16px(`rounded-2xl`) | 卡片 |
| `rounded-card-lg` | 24px(`rounded-3xl`) | 大区块(API landing hero) |
| `rounded-chip` | 9999px(`rounded-full`) | chips |
| `shadow-card` | `0 12px 40px rgba(0,0,0,0.04)` | 卡片 |
| `shadow-card-hover` | `0 14px 36px rgba(0,0,0,0.08)` | 卡片 hover |

---

## 10. 仿站原语登记(从 liblib 抽取)

按 `liblib-client-function-map.md` §6 摘出的"对 liblib 关键页面的可观察结构"逐项登记到 client 端的实现位置。便于后续验证"95% 仿站"的覆盖度。

| 仿站原语 | 来自 liblib | 现状 | 改造任务 |
|---|---|---|---|
| 68px 窄侧栏(无搜索) | `/ai-tool/*` 已登录态 | `NavBar` 折叠态 68px,`ToolRouteShell` 正确接入 | ✓ |
| 顶部青色活动条(限时抢购) | 同上 | `ToolPromoBar` 已落地 | ✓ |
| 右上 灵感库 / 会员超市 / 会员中心 / 登录注册 | 同上 | `ToolShellTopActions` 已落地(登录前后两态) | ✓ |
| Hero 居中标题 + 副标题 + 风格点 | 同上 | `ToolLandingWorkspace` 标题+7 风格点+副标题已落地 | ✓ |
| 880px 创作卡 + 40px prompt + chips + 灰色禁用按钮 | 同上 | `ToolComposer` 880px 卡 + chips + disabled button | ✓ |
| 一键做同款 6 张 3:4 卡 | 同上 | `InspirationsSection` 已落地 | ✓ |
| 登录模态:全屏暗遮罩 + 720px 卡 + 手机号 + 微信扫码 + QQ | 通用 | `OverlayHost.LoginModal` 已落地(720×454,含 banner) | ✓ |
| WebUI 工作区:左 prompt+参数 / 右队列 / 灵感库抽屉 | `/sd` | `SdLoggedInWorkspace` 已落地(checkpoint/VAE/tabs/编辑器/队列/灵感库) | ✓ |
| 6 列模型卡网格 + aspect 277/408 | `/image-model` | `ModelItemList` 6 列 + 277/408 已落地 | ✓ |
| 横向跑马灯(per-char cascade) | 首页公告 | `RecommendBanner` 已落地 | ✓ |
| 推荐模型条 + 65px 高色卡 | 首页 | `RecommendModelList` 已落地 | ✓ |
| 顶部 type tabs + 下方 sub tabs(双 sticky) | 首页 | `TypeTabs`(top:80) + `SubTabs`(top:134) 已落地 | ✓ |
| 历史生成流(已登录 creator) | `/ai-tool/*` 已登录 | `ToolLoggedInWorkspace.HistoryRecord` 已落地 | ✓ |
| 视频/图片/音频三上传 + 比例/分辨率/时长/配音 | `/ai-tool/video-generator` | `toolWorkspaces.video` 字段已定义 UI 壳 | ✓ |
| 教学分类 tab + 列表 + 右侧文章预览 | `/teaching` | `TeachingPage` 已落地 | ✓ |
| API landing hero + plans + FAQ | `/apis` | `ApiLandingPage` 已落地 | ✓ |
| 用户中心:头像 + 4 指标 + 列表 + 任务栏 | `/userpage/me/publish` | `UserPublishPage` 已落地 | ✓ |

**结论**:14 路由外壳**已经基本到位**,Phase 1 余下要做的是:(a) `useGenerateMachine` 的按钮接线;(b) `api/sse.ts` mock stub;(c) 命名迁移;(d) 6 个生成页的"做同款"点击行为统一到一个 `doSame()` 助手。

---

## 11. 平台桥(Desktop vs Web)

`platforms/` 已落地,Phase 1 不动实现,只把以下 4 件事在命名迁移时一并改:

- `gameai` → `eyetrace`(Tauri 插件 scheme + Rust 解析);
- `studio.gameai.asset` → `ai.eyetrace.app`(bundle identifier);
- `Game AI Asset Studio` → `EyeTraceAI`(productName + window.title);
- `platforms/desktop/deep-link.ts` 内部字符串注释同步改。

`platforms/web/deep-link.ts` 维持 no-op;`apiBaseUrl` 在 web/macos/windows 三个 yaml 中分别指向相对路径或 Tauri 后端端口(已有 `website.yaml` / `macos.yaml` / `windows.yaml`,本轮只校对品牌字段)。

---

## 12. 命名迁移

与 `project_guide.md` §9 完全对齐,落地点细化到本仓库:

| 文件 | 字段 | 现值 | 新值 |
|---|---|---|---|
| `src-tauri/tauri.conf.json` | `productName` | `Game AI Asset Studio` | `EyeTraceAI` |
| 同上 | `identifier` | `studio.gameai.asset` | `ai.eyetrace.app` |
| 同上 | `app.windows[0].title` | `Game AI Asset Studio` | `EyeTraceAI` |
| 同上 | `plugins.deep-link.desktop.schemes[]` | `["gameai"]` | `["eyetrace"]` |
| `src-tauri/src/lib.rs` | `parse` 内部字符串 | `gameai://` / `gameai:` | `eyetrace://` / `eyetrace:` |
| 同上 | `register("gameai")` | `register("gameai")` | `register("eyetrace")` |
| `src/platforms/desktop/deep-link.ts` | 注释 + 事件 payload | 引用 gameai | 引用 eyetrace |
| `package.json` | `name` | `client` | 保持(私有包无需改);但 `description` 改 "EyeTraceAI desktop + web client" |
| `src-tauri/Cargo.toml` | `name` / `description` | 见 `Cargo.toml` | 改为 `eyetrace-desktop` / EyeTraceAI |
| `src/config/static/default.yaml` | `appName` | 需校对 | `EyeTraceAI` |
| 所有 `.tsx` / `.ts` 中 `Game AI` 字符串 |  | 散落 | 替换为 `EyeTraceAI` / `眸迹AI` |

**范围控制**:命名迁移只动品牌标识与 scheme;`apiBaseUrl`、路由表、组件命名不动。

---

## 13. 测试策略

既有测试覆盖(`vitest run` 1.9s 内)已验证四件套 + 路由注册,Phase 1 不重写。补充:

- `api/sse.ts` mock stub:**纯单测**,不需要起 Tauri/RR6;验证定时器节奏与回调顺序即可。
- `useGenerateMachine` 接线:沿用 `fsm.test.ts` 已有的状态机断言;新增一个**集成测试**(用 `@testing-library/react` 渲染 `ToolWorkspacePage` + 模拟 `subscribeJobEvents`),断言"点击生成 → 看到 success 卡片"端到端跑通。
- 命名迁移:用 `rg` 确认 `Game AI` / `gameai` 命中为 0,作为 PR 合入门。
- 14 路由可达性:`vitest` 写一个最小 smoke(渲染 `<App/>` + 跑 `useNav().openView(viewId)` 到每个 viewId),验证不抛错即可(不校验样式)。

---

## 14. Phase 1 完成判据(细化)

满足以下**全部**条件视为 Phase 1 完工:

1. **可达性**:`npm run dev` 启动后,从首页点击 + deeplink `eyetrace://home` / `eyetrace://model/<id>` 都能进入 14 路由,无白屏、无控制台 error。
2. **品牌**:仓库内 `rg -i "game ai|gameai|studio\.gameai"` 命中数 = 0。
3. **数据**:除 `/content/home` 外,Network 面板不出现 `localhost:1420/api/...` 之外的任何 XHR(`/sse` mock 在 `EventSource` polyfill 上,不算真请求)。
4. **FSM**:进入 `/ai-tool/image-generator` 输入 prompt → 点生成 → 4 秒内出现 success 卡片(含 mock 图 + 模型名 + 「再次生成」按钮);`fsm.test.ts` 仍全绿。
5. **保护视图**:未登录态访问 `/sd` → 弹出 `LoginModal`;登录回调后跳到 `SdLoggedInWorkspace`。
6. **样式**:所有新增 / 改动代码不引入新的硬编码 hex(允许遗留,但 lint 期不允许 *新增* hex 字面量——可加 `eslint-plugin-no-restricted-syntax` 检查 `Literal[value=/^#[0-9a-f]{3,8}$/i]`)。
7. **保持**:Page Registry / ViewStack / KeepAlive / LayerManager / Command Bus 5 个原语**无破坏性改动**(`git diff --stat` 这几个目录行数 < 5)。
8. **测试**:`npm test` 全绿,新增 1 个集成测试覆盖"生成 → success"。

---

## 15. 后续 PR 拆分建议

按 §14 判据反向拆解,每 PR ≤ 400 行,顺序执行:

1. **PR-1 命名迁移**:§12 全表,**独立 PR**,不混样式。
2. **PR-2 `api/sse.ts` mock stub** + `subscribeJobEvents` 类型导出,补单测。
3. **PR-3 `useGenerateMachine` 接线**:在 `ToolLandingWorkspace` 与 `ToolWorkspacePage` 的"生成"按钮上接 `SUBMIT` + 替换 `success` 渲染为 `resultUrl` 卡片。
4. **PR-4 设计 token**:§9 的 CSS 变量 + tailwind extend + 替换 1 个页面(选 `HomePage`)做示范迁移。
5. **PR-5 14 路由 smoke 测试**:新文件 `__tests__/navigable.test.tsx`,遍历 viewId,验证不抛错。
6. **PR-6 README 收口**:在 `eye-trace-client/README.md` 加 Phase 1 入口 + 仿站映射表链接到本文件。

**不在本轮范围**(回退到 phase 2 计划):
- 真实生成接通;
- 鉴权(`Authorization` 头)解注;
- Comfy / Pretrain 专属工作区 UI(目前通用壳已够);
- 教学详情页独立路由;
- 视觉批量 token 迁移(只保新增不引入新 hex)。

---

## 附录 A. 仿站参考资源

- `www.liblib.art/`(已抓 HTML/CSS/JSON)→ `res/liblib-reference/{home,list,detail}.{png,json}`。
- `res/liblib/` 真实静态资源(favicon、登录 banner 等)。
- `docs/design/reissue_website/` 历史设计稿(本仓库 `client/docs/` 内有副本,`docs.zip` 84.6MB)。

## 附录 B. 关键文件坐标速查

| 关注点 | 文件 |
|---|---|
| 14 路由注册 | `src/app/registry/pages.ts` |
| 路由分发 | `src/app/routes/index.tsx` |
| 持久化导航 | `src/app/stack/stack.ts` + `useViewStackSync.ts` |
| 路由缓存策略 | `src/app/keepalive/KeepAlive.tsx` |
| 弹层 / 模态 | `src/app/layer/manager.ts` + `OverlayHost.tsx` |
| 鉴权闸 | `src/app/auth/protectedNav.ts` |
| AppShell | `src/components/layout/AppShell.tsx` |
| 侧栏 | `src/components/layout/NavBar.tsx` |
| 顶栏 | `src/components/layout/TopBar.tsx` |
| 底栏 | `src/components/layout/CenterBar.tsx` |
| 数据契约 | `src/api/content.ts` + `src/config/static/types.ts` |
| 运营内容 yaml | `src/config/static/default.yaml` |
| 静态配置加载 | `src/config/static/index.ts` |
| 生成 FSM | `src/features/generate/fsm.ts` + `useGenerateMachine.ts` |
| Tauri 入口 | `src-tauri/src/lib.rs` |
| Tauri 桥 | `src/platforms/desktop/*.ts` |
