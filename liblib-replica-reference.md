# Liblib 仿站信息登记 v1

> 版本: 1.1 | 日期: 2026-06-12 | 状态: 信息登记(非实施)
> 信息源: `https://www.liblib.art/`(2026-06-11 / 2026-06-12 两次抓取)
> 落点: 本仓库 `eye-trace-client/res/liblib-reference/{home,list,detail}.{png,json}` + 镜像 `www.liblib.art/` + 抓取工具 `tools/liblib-grab/`
> 上位文档: [`client-architecture.md`](./client-architecture.md) §10 仿站原语登记(本文为其**原始信息源**)

本文档**只登记从 liblib 抓取到的客观事实**(URL / 视口尺寸 / DOM 几何 / 文案 / 字段 / 资源 URL),不写客户端代码、不写样式。客户端要"95% 仿站"时,本文件是 ground truth,`client-architecture.md` §10 是简表。

---

## 目录

1. 抓取元信息
2. 首页 (`/`)
3. 列表页 (`/image-model`)
4. 模型详情页 (`/modelinfo/:id?versionUuid=...`)
5. 资源样例
6. 与 `liblib-client-function-map.md` 的关系

---

## 1. 抓取元信息

> 2026-06-12 第二轮抓取后,本文升级为 v1.1。原始 2026-06-11 抓取见 `res/liblib-reference/{home,list,detail}.{png,json}` 与 legacy `www.liblib.art/www.liblib.art/` 目录。

### 1.1 两次抓取汇总

| 抓次 | 日期 | 范围 | 工具 | 产物 |
|---|---|---|---|---|
| 第一轮 | 2026-06-11 | 3 个关键页(首页 / 列表 / 详情)+ 静态资源 | Chrome saveAllResource | `www.liblib.art/www.liblib.art/`(legacy,3.6MB)+ `res/liblib-reference/{home,list,detail}.{png,json}` |
| **第二轮** | **2026-06-12** | **14 路由全量**(去 /userpage/me/publish 外加 1 个模型详情样本) | `tools/liblib-grab/grab.mjs`(Playwright) | `www.liblib.art/{hosts,liblibai-*,stubs}/` 共 **126MB**,1177 个 response 写入磁盘,576 个唯一 URL 登记到 manifest |

### 1.2 第二轮 manifest 概览(`www.liblib.art/manifest.json`)

```
grabbedAt:  2026-06-12T10:43:18Z
routes:     14 / 14 全部 ok
resources:  576 unique URLs
            ├─ saved (221)   真资源二进制
            ├─ stubbed (350) 第三方 host 占位(注释头)
            └─ skipped (5)   api2.liblib.art JSON API 无 body
bytes:      126,186,160 (saved) + 143,112 (stubbed)
```

### 1.3 抓取工具与可重放命令

```bash
cd /Users/mac/github/eye-trace/tools/liblib-grab
npm install
npx playwright install chromium
node grab.mjs                  # 全量抓取(本次默认跑了这个)
node grab.mjs --diff           # 对比新旧 manifest 差异
node grab.mjs --no-stubs       # 不 stub,真抓第三方 SDK(不推荐)
node grab.mjs --no-images      # 跳过图片(快速 smoke)
node grab.mjs --screenshots    # 同时输出 14 路由整页截图
```

### 1.4 客观事实(原值)

| 项 | 值 |
|---|---|
| 抓取入口 | `https://www.liblib.art/`(全站 SPA,Next.js 14) |
| 视口 | 1689×1233(home)、1674×5131(list)、详情自适应 |
| Mantine CSS 来源 | `liblibai-web-static.liblib.cloud/.../static/_next/static/css/*.css`(**单独 CSS 文件**,31 个) + Mantine CSS-in-JS runtime 注入 `<style>` 标签 |
| Next.js 客户端 chunk | 134 个 `liblibai-web-static.liblib.cloud/.../chunks/*.js`(本轮全抓到) |
| 字体栈 | `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji"`(与本仓库 `globals.css` 一致) |
| 主题 | `bgBody = #FFFFFF`,`colorBody = #000000`(light mode 默认);暗色模式未抓取 |
| 鉴权依赖 | 跨域 `www.liblib.tv/cross-storage-hub.html` 同步 cookie,本仓库登录后壳以 `auth.login` modal 替代 |

### 1.5 抓得到 vs 抓不到

| 资源 | 可抓性 | 落点 |
|---|---|---|
| Next.js HTML 壳 | ✓ | `hosts/www.liblib.art/<path>` |
| Next.js client chunks (134 个) | ✓ | `liblibai-web-static.liblib.cloud/.../chunks/*.js` |
| Next.js 静态 CSS (31 个) | ✓ | `liblibai-web-static.liblib.cloud/.../css/*.css` |
| 图片资源 (banner / 头像 / 封面) | ✓ | `liblibai-online.liblib.cloud/...` + `liblibai-web-static.liblib.cloud/.../static/_next/static/images/` |
| 第三方 SDK (baidu/aliyun/rum/sdk/qlogo) | ✗ → stubbed | `stubs/<host>/<path>`(注释头,0 字节) |
| WebSocket 帧 | ✗ | 跳过 |
| 跨域 cookie 登录态 | ✗ | 跳过 |

**关键观察(修正)**: 之前"raw HTML 7KB 不够克隆"是误读——raw HTML 实际 ~28KB,正文靠 Next.js 客户端 chunk 在浏览器执行;**第二轮抓取补齐了 134 个 JS chunk + 31 个 CSS**,现在 file:// 加载能跑出完整 Mantine 渲染态。

### 1.6 第三方 stub host 清单(本轮 stubbed 350 个 URL,来自 25 个 host)

```
fxgate.baidu.com              hm.baidu.com                fclog.baidu.com
o.alicdn.com                  retcode.alicdn.com          static-captcha.aliyuncs.com
g9m3a0kv73-default-cn.rum.aliyuncs.com  sdk.rum.aliyuncs.com
upload.captcha-open.aliyuncs.com          5469wv.captcha-open.aliyuncs.com
cloudauth-device-dualstack.cn-shanghai.aliyuncs.com
lf3-data.volccdn.com          tab.volces.com              gator.volces.com
www.googletagmanager.com      www.google.com              www.google.com.sg
analytics.google.com          static.howxm.com
passport.liblib.art           bridge.liblib.art           www.liblib.tv
thirdqq.qlogo.cn              thirdwx.qlogo.cn
g.alicdn.com
```

**stub 文件格式**(3 行注释头,内容相同):
```js
// STUB: liblib-grab skipped this third-party host.
// url: <原始 URL>
// host: <原始 host>
```

---

## 2. 首页 (`https://www.liblib.art/`)

> 原始信息源:`res/liblib-reference/home.{png,json}`(1689×1233 视口)

### 2.1 视口与主题

```
title:    "LiblibAI-哩布哩布AI - 中国领先的AI创作平台"
viewport: { w: 1689, h: 1233 }
bgBody:   rgb(255, 255, 255)
colorBody: rgb(0, 0, 0)
fontFamily: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji"
```

### 2.2 区域切分(自顶向下 DOM 顺序)

| # | tag | 区域 | 几何 (x,y,w,h) | 关键内容 |
|---|---|---|---|---|
| 1 | `header` mantine-Header-root z-100 | 顶部活动条 | (0,0,1674,48) | 倒计时 + "618限时直降·折上再减¥618..." + 限时抢购 |
| 2 | `div` mantine-AppShell-root | 主壳 | (0,0,1674,4885) | 含 Navbar / main / Overlay |
| 2.a | `nav` mantine-Navbar-root | 左侧栏 | (0,48,200,1185) | 13 条主链接 + 「前往LibTV创作」底栏(详见 §2.3) |
| 2.b | `main` mantine-AppShell-main | 主区 | (0,0,1674,4885) | TopBar / Banner 轮播 / 推荐模型条 / 类型 tab / 子 tab / 6 列网格 |

主区 children(自顶向下):

1. 邀请有礼 + 会员超市 + 能量点 + 限时 37 折 + 「会员特惠37折」+ 「WebUI ComfyUI 训练LoRA」3 链接
2. 一键使用顶尖视频/图片模型 标题 + 跑马灯公告 + 横向推荐模型条
3. 类型 tab(图片模型 / 视频特效 / 发现灵感 / 工作流)
4. 子 tab(全部 / 摄影写真 / 电商营销 / ...)
5. 6 列瀑布流模型卡(实际抓到 10 张样本)

### 2.3 完整 navLinks(13 条)

| 文案 | href |
|---|---|
| 首页 | `/` |
| 图片生成 | `/ai-tool/image-generator` |
| 视频生成 | `/ai-tool/video-generator` |
| WebUI | `/sd` |
| ComfyUI | `/comfy` |
| 训练 LoRA | `/pretrain` |
| AI 应用 | `/lib3` |
| 资产 | `/asset` |
| 个人中心 | `/userpage/a218ba5f9a30429ab880c740dc421c5c/publish` |
| 创建团队 | (无 href,弹 confirm 提示待接入) |
| 创作中心 | `https://creator.liblib.art`(外链) |
| 发布 | `/userpage/a218ba5f9a30429ab880c740dc421c5c/publish` |
| 教程 | `/teaching` |
| API | `/apis` |
| 前往LibTV创作 | `https://www.liblib.tv/`(外链,底栏) |

> 注:本仓库 `client-architecture.md` §5 与此清单一致;`/image-model` 是从 `图片模型` 类型 tab 进入的子路由(未在 navbar 一级),实际归属 `/`。

### 2.4 卡片字段(首页瀑布流 6 列网格样本 10 张)

每张卡片(链接到 `/modelinfo/{uuid}?from=feed&versionUuid={vuuid}`)字段:

| 字段 | 示例 |
|---|---|
| `href` | `/modelinfo/36f3dc699a8e4e40bf3451be74d53ab2?from=feed&versionUuid=f09f7e72bbcb4435abe9dc2283e3ef38` |
| `title` | 完整文本(用户 + 数字 + 行动词 + 标签 + 标题拼接) |
| `img` | `https://liblibai-online.liblib.cloud/img/{hash}/{file}.png?x-oss-process=...` |
| `w × h` | 284 × 412.66(6 列 = 6×284 + 5×5 gap ≈ 1606 + main 左 padding) |

`title` 的拼接结构(从样本 1 拆出):

```
铭枫 1.4k 8 使用模型 独家 LORA F.1 F.1 国风原画 | CG | 国漫人物
└─┘ └──┘┘└──┘ └──┘ └──┘ └───┘ │  └─────────────────┘
作者 收藏 评论 行动词  独家  类型  基础算法  模型名
```

**行动词枚举**:`使用模型` | `使用模板` | `去WebUI使用`
**类型枚举**:`LORA` | `Checkpoint` | `模板`
**基础算法枚举**:`F.1` | `F.2` | `XL` | `Qwen-Image` | `ERNIE-Image` | `Seedream 5.0` 等

### 2.5 资源样例(banner / icon / avatar)

34 张抓到的图片,按类别抽样(详见 `res/liblib-reference/home.json` `assets.imgs`):

- **Banner**(5 张,1400×382):`https://liblibai-online.liblib.cloud/banner/{id}.png?x-oss-process=image/resize,w_1400,m_lfit/format,webp`
- **小 Banner**(3 张,800×400):同 host `/banner/`
- **模型封面**:社区资源走 `community-img/genius_playground/image/...`;普通资源走 `/img/...`;OSS 处理 `?image_process=format,webp&x-oss-process=image/resize,w_600,m_lfit/format,webp/ignore-error,1`
- **头像**:100×100 圆角,`/web/avatar/avatar{1..5}.png` 是占位
- **VIP 角标**(1 张 78×32):`/web/e43bfc9f08aa4776b7c1647006be562d/...`

---

## 3. 列表页 (`https://www.liblib.art/image-model`)

> 原始信息源:`res/liblib-reference/list.{png,json}`(1674×5131 视口)

### 3.1 顶部壳与首页一致

- 顶部活动条
- Navbar(同首页)
- TopBar(同首页):logo / nav / search(占位文案 "搜索视频特效/模型/AI应用/工作流/创作者/图片") / user actions
- 左侧**无**侧栏(本路由专用壳,200px 宽 logo 区域让位给 promo)

### 3.2 主页结构(自顶向下)

| # | 组件 | 几何 / 行为 |
|---|---|---|
| 1 | **Promo Carousel** | 顶部 1400×382 横幅轮播(slick 风格);包含「推荐 / ComfyUI / WebUI / 训练LoRA」3 个 CTA 卡 + Seedance 2.0 / Seedance 2.0 Fast / Happy Horse 1.0 / 智能图片V2 / 全能图片V2-Flash / Seedream 5.0 lite / 可灵 3.0 / 可灵 3.0 Omni 模型横幅 |
| 2 | **MainCategoryTabs** | 4 项,横排;active 无下划线但加深;第一项默认 active。枚举:图片模型 \| 视频特效 \| 发现灵感 \| 工作流 |
| 3 | **SubCategoryChips** | 12 项,横排 chip 样式;active=`bg-[#F2F3F5] font-medium cursor-default`(无 hover);inactive=`cursor-pointer bg-[var(--color-background)] text-[#666666] hover:bg-[#F6F6F6]` |
| 4 | **ActionPills** | 右上 2 个,32px 高,「推荐」dropdown(sort:最新/最热) + 「筛选」popover(z-300,~600px 宽) |
| 5 | **CardGrid** | 6 列瀑布流,`role=grid`;card 尺寸 284×~413;列间距 5px;**滚动到底自动加载下一页**(infinite scroll,无 pager) |

### 3.3 SubCategoryChips 完整列表(12 项)

```
全部 (默认 active)
🤩每周新选·图片模板
🔥爆款Top100模型
摄影写真
电商营销
动漫游戏
风格插画
平面设计
建筑及室内设计
创意玩法
文创周边
小说推文
```

### 3.4 筛选 Popover 结构(6 段,全部 checkbox)

| 段 | 子段 | 选项 |
|---|---|---|
| 发布时间 | — | 全部 / 1天 / 1周 / 1个月 / 1年 |
| 会员专属 | 会员专属 | 仅会员可下载 |
| | 免费模型 | (空) |
| | 商用许可 | 生成图片可商用 / 模型或融合模型可转售 |
| 模型类型 | — | Checkpoint / Textual Inversion / Hypernetwork / Aesthetic Gradient / 模板 / LoRA / LyCORIS / Controlnet / Poses / Wildcards / Other |
| 基础算法 | 图片模型 | Seedream 5.0 / Seedream 4.5 / Qwen-Image / Qwen-Edit / 全能图片模型 / 智能图片V2 / 基础算法 F.1 / 基础算法F.2 Pro / 基础算法 F.2 Flex / 基础算法 F.2 Dev / Z-Image / Z-Image-Base |
| | 图片编辑模型 | (空) |
| | (其它基础模型) | 基础算法 XL / 基础算法 v1.5 / Kolors / HiDream / 混元DiT v2.1 / 混元DiT v1.2 / 混元DiT v1.1 / Mj V7 / LTX 2.3 / ERNIE-Image / Anima / 基础算法 v2.1 / 基础算法 v3 / 基础算法 v3.5M / 基础算法 v3.5L |
| 模型使用范围 | — | 可下载 / 可融合 |
| 平台特权 | — | 独家 |

### 3.5 卡片 DOM 结构

```html
<a href="/modelinfo/{uuid}?from=feed&versionUuid={vuuid}">
  <img src="{cover}?x-oss-process=..." />   <!-- cover -->
  <img src="{avatar}?x-oss-process=..." />  <!-- author avatar -->
  <h6>{title}</h6>
  <!-- 内嵌叶子节点顺序:作者 / 收藏(k) / 评论 / 行动词 / 独家? / 类型 / 基础算法? / 标题 -->
</a>
```

### 3.6 React 副本数据模型(原 JSON 自带 `reactReplicaNotes`)

```ts
interface Card {
  id: string;          // uuid from href
  title: string;       // from h6
  coverUrl: string;
  author: { name: string; avatarUrl: string };
  stats: { favorites: string; downloads: string };
  actionLabel: '使用模型' | '使用模板' | '去WebUI使用';
  isExclusive: boolean;
  modelType: 'LORA' | 'Checkpoint' | '模板' | ...;
  baseModel: 'F.1' | 'XL' | 'Qwen-Image' | 'ERNIE-Image' | ...;
  href: string;        // model detail link with versionUuid
}

interface State {
  selectedMainCategory: string;
  selectedSubCategory: string;
  sortBy: '推荐' | '最新' | '最热';
  filters: {
    publishTime: string;
    memberOnly: string[];
    free: boolean;
    commercial: string[];
    modelTypes: string[];
    baseModels: string[];
    usageScope: string[];
    platform: string[];  // 独家
  };
  page: number;        // infinite scroll triggers page++
}
```

本仓库当前 `ImageModelListPage.tsx` 用本地 12 chip 列表 + `content.modelItems` 数据 + 6 列 `ModelItemList`;筛选 / sort / 无限滚动**未实现**(Phase 1 不需要)。

---

## 4. 模型详情页 (`/modelinfo/:id?versionUuid=...`)

> 原始信息源:`res/liblib-reference/detail.{png,json}`(2 个样本)

### 4.1 样本 1:`CHEN-极致肤质-消除AI感-FLUX.1`

```
url:     /modelinfo/41e061cedf4c4eb39a06876b8e758b4c?versionUuid=b12da90829ea4f6d9dc90ed2eed111ab
title:   "CHEN-极致肤质-消除AI感-FLUX.1"
type:    LORA
baseModel: F.1
tags:    [女生, 消除AI感, 皮肤增强, 独家]
stats:   { onlineGenerations: "46.7k", downloads: "386", favorites: 148, comments: 91, rawLabel: "48.2k / 469 / 91 / 148" }
```

### 4.2 完整字段表(从 detail.json 抽)

| 分类 | 字段 | 类型 / 示例 |
|---|---|---|
| 顶部 | `url` | 详情页 URL(含 versionUuid) |
| | `title` | 模型名 |
| | `pageTitle` | "模型名-类型-作者-LiblibAI" |
| | `type` | `LORA` \| `Checkpoint` \| `LoRA` ... |
| | `baseModel` | `F.1` \| `XL` ... |
| | `tags` | string[] |
| 统计 | `stats.onlineGenerations` | 在线生成数(带 k 后缀) |
| | `stats.downloads` | 下载数 |
| | `stats.favorites` | 收藏数(int) |
| | `stats.comments` | 评论数 |
| | `stats.rawLabel` | 原始拼接串 "48.2k / 469 / 91 / 148" |
| 作者 | `author.name` | 作者昵称 |
| | `author.isVerified` | "Lib认证作者" |
| | `author.stats` | ["303 作品", "77 关注", "70.2k 粉丝", "1.4k 获赞"] |
| | `author.hasFollowButton` | true |
| 行动 | `actions[]` | ["去图片生成", "去WebUI生成", "加入收藏", "下载加密模型", "下载Lib客户端"] |
| 版本 | `versions[]` | ["v4.0", "v3.0", "v2.0", "v1.0"] |
| | `versionMeta.lastUpdate` | "2025/08/26" |
| | `versionMeta.firstRelease` | "2025/03/20" |
| 画廊 | `sampleImages[]` | { src, alt, w, h }[],首张最大 1000×1005 |
| 描述 | `description` | 多段长文本(含「V4.0更新」「核心亮点」段) |
| 侧栏 | `sidePanel.verified` | "2025/03/29" |
| | `sidePanel.aiContent` | true |
| | `sidePanel.downloadSize` | "292.23MB" |
| | `sidePanel.encrypted` | true |
| 版本详情 | `versionDetails.type` | LORA |
| | `versionDetails.isOriginal` | "原创" |
| | `versionDetails.baseModel` | F.1 |
| | `versionDetails.function` | "打造无瑕肌肤,还原真实感" |
| | `versionDetails.triggerWord` | "skin-v1" |
| | `versionDetails.recommendedParams` | { 搭配Checkpoint, 推荐权重, CFG, VAE, 高清放大算法 } |
| 授权 | `versionDetails.license` | { commercial, noResell, creation, liblibOnline, merge } |
| 评论 | `comments.title` | "讨论(2)" |
| | `comments.sortTabs` | ["最热", "最新"] |
| | `comments.total` | 2 |
| | `comments.samples[]` | { user, text, date, replies, isAuthor? } |
| 作品 | `works.title` | "作品展示(63)" |
| | `works.sortTabs` | ["最热", "最新"] |
| | `works.showcaseButton` | "我要晒图" |
| | `works.sampleUsers[]` | 6 个用户名 |
| 布局 | `layout.container` | "main with top fixed banner" |
| | `layout.leftColumn` | "preview images + description + version meta" |
| | `layout.rightColumn` | "author card + action buttons + version details + license" |
| | `layout.belowFold` | "讨论 + 作品展示 瀑布流" |
| | `layout.versionTabs` | 水平 tab,选中高亮 |
| | `layout.h2Structure` | ["H3:讨论(2)", "H3:作品展示(63)"] |

### 4.3 推荐参数样本

```yaml
搭配Checkpoint: F.1基础算法模型-哩布在线可运行_F.1-dev-fp8
推荐权重: 0.8
CFG: 3.6
VAE: 无
高清放大算法: 4x-UltraSharp
```

### 4.4 授权字段样本(5 项)

```yaml
commercial: 生成内容仅会员可商用
noResell: 不可转售模型或出售融合模型
creation: 可
liblibOnline: 可LiblibAI在线生图
merge: 不可进行融合
```

### 4.5 当前 client 实现对照

`ModelDetailPage.tsx` 已支持:`gallery[]` / `typeTag` / `author` / `authorAvatar` / `badges[]` / `stats[]` / `versions[]` / `activeVersion` / `description[]` / `recommendedPrompts[]` / `actions[]` / `meta[]`(用作"模型参数") / `license[]` / `comments[]`。

**Phase 1 不需要扩展**:1) `comments.total` 与 `sortTabs` 已可由 yaml 静态填充;2) `works` 段暂未在 `ModelDetailConfig` 中,Phase 1 不展示;3) `versionDetails.recommendedParams` 已合并到 `meta[]`;4) `sidePanel.encrypted/downloadSize/verified/aiContent` 在 UI 上以"下载加密模型"按钮替代。

---

## 5. 资源样例

### 5.1 现场抓取目录结构(节选)

```
www.liblib.art/www.liblib.art/
├── favicon.ico (15KB)
├── index.html / index (1).html / image-model.html  (各 ~28KB,Next.js 壳)
├── modelinfo/
│   ├── 41e061cedf4c4eb39a06876b8e758b4c.html  (77KB)
│   └── ab5ce555692d4a4da514d03654418424.html  (80KB)
└── static/
    ├── vipModel/
    │   └── diamond.png
    └── actionFollow/
        ├── actionModel1.gif / actionModel2.gif
        ├── pro.png / std.png
```

### 5.2 资源 URL 模式(从 fixtures 总结)

| 用途 | URL 模式 |
|---|---|
| 头像(占位) | `https://liblibai-online.liblib.cloud/web/avatar/avatar{1..5}.png` |
| 用户头像(真) | `https://liblibai-online.liblib.cloud/img/{hash}/{file}.{ext}?x-oss-process=image/resize,w_100,m_lfit/format,webp` |
| 模型封面 | `https://liblibai-online.liblib.cloud/{img,community-img}/.../...{png,jpg,jpeg}?x-oss-process=image/resize,w_600,m_lfit/format,webp/ignore-error,1` |
| Banner | `https://liblibai-online.liblib.cloud/banner/{id}.{png,jpg}?x-oss-process=image/resize,w_1400,m_lfit/format,webp` |
| 静态图(UI 元素) | `https://liblibai-online.liblib.cloud/web/{hash}/{file}.{svg,png,jpeg}` |
| 训练产物 | `https://liblibai-online.liblib.cloud/train/model_images/{uuid}/flux-lora_e000007.{png}?image_process=format,webp&x-oss-process=image/resize,w_600,m_lfit/format,webp/ignore-error,1` |
| 详情画廊 | `https://liblibai-online.liblib.cloud/{img,community-img}/{hash}/{file}?x-oss-process=image/resize,w_1000,m_lfit/format,webp` |
| 微信头像 | `https://thirdwx.qlogo.cn/mmopen/vi_32/...` |
| QQ 头像 | `https://thirdqq.qlogo.cn/...` |
| 实时分析 SDK | `lf3-data.volccdn.com/obj/data-static/log-sdk/collect/5.0/collect-rangers-v5.1.12.js`(火山引擎) |
| 阿里验证码 | `o.alicdn.com/captcha-frontend/aliyunCaptcha/AliyunCaptcha.js` |
| Next.js 静态资源 | `liblibai-web-static.liblib.cloud/liblibai_v4_online/static/_next/static/chunks/*.js` |

### 5.3 依赖与第三方 SDK(站点全局)

- **Next.js 14** + **Mantine 7** UI 库(class 前缀 `mantine-`)。
- **react-query** / **zustand**(从 className 推测,未抓 JS 验证)。
- **Sentry**: `g9m3a0kv73-default-cn.rum.aliyuncs.com`(阿里云 RUM)。
- **GrowingIO 兼容埋点**:`howxm.com` / `hm.baidu.com`。
- **阿里云验证码**:`AliyunCaptcha.js` + `static-captcha.aliyuncs.com`。
- **火山引擎 Gator SDK**:`gator.volces.com`。
- **Google Tag Manager**:`www.googletagmanager.com` / `analytics.google.com`。
- **Baidu Tongji**:`hm.baidu.com` / `fclog.baidu.com` / `fxgate.baidu.com`。

**仿站取舍**:本仓库**不集成**以上任何第三方 SDK / 验证码 / 埋点;登录态用 `auth.login` modal 替代,UI chrome 完全本地化。

---

## 6. 与 `liblib-client-function-map.md` 的关系

| 关注点 | 本文 | `liblib-client-function-map.md` |
|---|---|---|
| 14 路由的**功能面** | (无) | ✓(§2 路由表 + §3 交互清单 + §4 demo 数据策略) |
| 3 关键页的**客观事实**(几何/字段/资源) | ✓(§2-4) | §6 仅"已登录前后捕获",无字段 |
| 仿站**实施蓝本** | `client-architecture.md` §10(简表) | (无) |

**使用方式**:

- 设计/产品/视觉走查 → 读 `liblib-client-function-map.md`(功能面);
- 实施时需要具体数值/字段/URL → 读本文件(客观事实);
- 写代码前对齐 → 读 `client-architecture.md` §10(简表 + 实施位)。

---

## 附录 A. 现场材料坐标

| 用途 | 路径 |
|---|---|
| 首页 DOM 几何 + 文案 | `eye-trace-client/res/liblib-reference/home.json` |
| 首页截图(1689×1233) | `eye-trace-client/res/liblib-reference/home.png` |
| 列表 DOM 几何 + 字段 | `eye-trace-client/res/liblib-reference/list.json` |
| 列表截图(1674×5131) | `eye-trace-client/res/liblib-reference/list.png` |
| 详情字段 | `eye-trace-client/res/liblib-reference/detail.json` |
| 详情截图 | `eye-trace-client/res/liblib-reference/detail.png` |
| 首页复刻对比(自截) | `eye-trace-client/res/liblib-reference/replica-home.png` |
| 列表复刻对比(自截) | `eye-trace-client/res/liblib-reference/replica-list.png` |
| 原始 HTML 壳(legacy) | `www.liblib.art/www.liblib.art/{index,image-model}.html` |
| 原始详情 HTML(legacy) | `www.liblib.art/www.liblib.art/modelinfo/{uuid}.html` |
| 原始静态资源(legacy) | `www.liblib.art/www.liblib.art/static/{vipModel,actionFollow}/` |
| **第二轮抓取 manifest** | `www.liblib.art/manifest.json` |
| **第二轮抓取 liblibai-online 资产** | `www.liblib.art/liblibai-online.liblib.cloud/`(~720MB) |
| **第二轮抓取 liblibai-web-static chunks+CSS** | `www.liblib.art/liblibai-web-static.liblib.cloud/`(~26MB) |
| **第二轮抓取 hosts 目录(非 liblibai-* 主域)** | `www.liblib.art/hosts/`(~1.4MB) |
| **第二轮抓取第三方 stub** | `www.liblib.art/stubs/<host>/`(368KB) |
| 抓取脚本 + README | `tools/liblib-grab/{grab.mjs,package.json,README.md}` |
| favicon | `eye-trace-client/res/liblib/favicon.ico` |
| VIP badge | `eye-trace-client/res/vip-badge.png` |
| 登录 banner 图 | `eye-trace-client/src/app/layer/OverlayHost.tsx:182-183` 内联 URL |
| 登录 QR 图 | `eye-trace-client/src/app/layer/OverlayHost.tsx:184-185` 内联 URL |

## 附录 B. 关键数值速查

| 类别 | 数值 |
|---|---|
| 首页卡片宽 × 高 | 284 × 412.66 |
| 列表卡片宽 × 高 | 284 × 413 |
| 卡片列数 | 6 |
| 列间距 | 5px |
| 左侧栏宽(展开 / 折叠) | 200px / 68px |
| 顶部活动条高 | 48px |
| Banner 尺寸(主) | 1400 × 382 |
| Banner 尺寸(侧) | 800 × 400 |
| 详情大图最大边 | 1000px |
| SubCategoryChips 数 | 12 |
| MainCategoryTabs 数 | 4 |
| Navbar 主链接数 | 14(含「前往LibTV创作」) |
| 筛选 Popover 段数 | 6 |
| 基础算法枚举数(图片模型) | 12 |
| 基础算法枚举数(其它) | 15 |
| 第三方 SDK 域(stubbed) | 25(本轮) |
| **第二轮抓取路由数** | 14 / 14 ok |
| **第二轮 unique URL 数** | 576 |
| **第二轮真资源 saved** | 221(JS 134 / CSS 31 / image 90+ / json 9) |
| **第二轮 stub 资源** | 350 |
| **第二轮产物总字节** | 126MB |
