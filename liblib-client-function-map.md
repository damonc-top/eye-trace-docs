# Liblib Client Function Map

> Updated: 2026-06-11  
> Reference: live inspection of `https://www.liblib.art/` and key child routes.  
> Goal: 1:1 copy the visible client function surface first, then reverse backend APIs from the client contract.

## 1. Product Shell

Liblib is a single creator workstation, not a marketing-only website. The client shell has these persistent areas:

| Area | Visible Function | Client Requirement | Backend Later |
|---|---|---|---|
| Left nav | Home, creation group, asset, user center, creator center, publish, teaching, API, LibTV | Page Registry entries and external-link escape hatch | Navigation/content permissions |
| Top bar | Search, invite, member mall, credits, VIP promo, notifications, account/login | Search route, promo data, auth modal | Search API, user wallet, membership, notifications |
| Activity banner | Time-limited campaign strip | Configurable promo copy/countdown | Campaign API |
| Center generation bar | Collapsed prompt pill, expanded generation form, fullscreen input | Global quick-create overlay | Generate task API |
| Layer system | Login modal, filter/sort dialogs, protected tool gates | LayerManager/Dialog primitive | Auth/session, entitlement |

## 2. Routes And Pages

| Route | Page | Function Scope |
|---|---|---|
| `/` or `/inspiration` | Inspiration home | Banner carousel, recommended model strip, type tabs, category chips, sort/filter entry, waterfall cards |
| `/search?keyword=` | Search results | Keeps keyword in search box, resource-type tabs: image model, video effect, AI app, workflow, work; filter entry; result cards |
| `/ai-tool/image-generator` | Image generation | Mode tabs, prompt textarea, model selector, style/upload chip, aspect ratio/count, disabled/enabled generate, "do same" inspiration cards |
| `/ai-tool/video-generator` | Video generation | Same tool frame with video/image/audio uploads, video model, reference mode, ratio/resolution/duration/audio params |
| `/sd` | WebUI | Protected professional workspace; unauthenticated login gate; authenticated demo shows model/params/task panels |
| `/comfy` | ComfyUI | Protected node workflow workspace; unauthenticated login gate; authenticated demo shows workflow/actions/task panels |
| `/pretrain` | Train LoRA | Protected training workspace; dataset upload, base model, trigger words, training progress |
| `/lib3` | AI apps | App category grid, selected app runner, upload controls, sliders/inputs, one-click generate, result/empty state |
| `/asset` | Assets | Product tabs, asset-type select, date range, favorite checkbox, empty/list state |
| `/modelinfo/:id` | Model detail | Gallery, title/stats, versions, description, author card, CTA, favorite/follow, version params, license, comments, showcase |
| `/imageinfo/:id` | Work detail | Work media, author/follow, like/share, date/tool, tags, prompt/params, related model links, comments, do-same CTA |
| `/teaching` | Teaching home | Teaching category tabs, latest articles, video tutorials, paid courses entry, footer info |
| `/teaching/:id` | Teaching detail | Article template, author/date, rich content, related links |
| `/apis` | API landing | Hero CTA, model capability sections, comparison tabs, pricing plans, FAQ |
| `/userpage/me/publish` | User center | User works/assets/favorites/tasks shell; auth required later |

## 3. Core Interaction Inventory

| Interaction | Frontend Behavior Now | Backend Contract Later |
|---|---|---|
| Search submit | Navigate to `/search?keyword=value` and render configured results | `GET /search?keyword=&type=&filters=` |
| Type/category chip | Local tab/chip active state and filtered demo card lists | taxonomy + model feed query |
| Sort/filter | Dialog entry; if auth required show login gate | `GET /filters`, saved user preferences |
| Model card click | In-app route to model detail | `GET /models/:id` |
| Work card click | In-app route to image/work detail | `GET /works/:id` |
| "Do same" | Fill/route into generator with demo state | generation template clone API |
| Generate | Disabled until prompt/upload requirement is satisfied; demo task card | `POST /generate/jobs`, `GET /generate/jobs/:id` |
| Professional tools | Login gate if unauthenticated | auth/session + entitlement |
| Favorite/follow/like/comment | Visible buttons, local demo state only | social/action APIs |
| Asset filters | Local demo filtering by product/type/favorite/date | `GET /assets` |
| API purchase/contact | Visible CTA only | billing/order/contact APIs |

## 4. Demo Data Strategy

All visible operational content should be configurable from `src/config/static/default.yaml` while the backend is absent. The YAML remains an offline fallback after server APIs land.

Keep in YAML permanently:

- `apiBaseUrl`, `features`, `request`, platform/window config
- UI hard limits and static tokens such as `maxPromptLength`, `supportedSizes`, `inputRadius`

Move to server later, but keep YAML fallback:

- home banners, recommended models, waterfall cards, VIP promo
- tool workspaces and inspiration cards
- AI app gallery and runner definitions
- asset demo rows and filters
- search result cards
- model/work detail records
- teaching lists/articles
- API landing sections, plans and FAQs

## 5. First Development Milestone

The first client milestone is not real generation. It is a complete navigable product shell:

1. Home content moves into `HomePage`; `AppShell` keeps only persistent shell, topbar, nav, keepalive and centerbar.
2. Topbar search navigates to `/search?keyword=...`.
3. Replace placeholder routes with data-driven demo pages.
4. Add YAML sections for `toolWorkspaces`, `protectedWorkspaces`, `aiApps`, `assets`, `search`, `modelDetails`, `imageDetails`, `teaching`, `apiLanding`.
5. Preserve current Page Registry, ViewStack and KeepAlive rules.
6. Leave all mutating actions as visible local/demo controls until backend contracts are built.

## 6. Live Page Capture Rule

`https://www.liblib.art/ai-tool/image-generator` proves that raw HTML source is not enough for 1:1 cloning:

- `curl` source is a ~7KB Next.js shell with `__NEXT_DATA__`, static CSS/JS chunks and no full page body.
- The clone baseline must use three layers: raw HTML shell, browser-rendered DOM, and screenshot/geometry sampling.
- Store sampled layout facts in docs before implementation, then express data in YAML wherever the value is operational content.

Captured facts for `/ai-tool/image-generator`:

| Area | Observed Structure |
|---|---|
| Shell | Narrow 68px icon sidebar, no normal search topbar, no CenterBar; page owns its tool header/actions |
| Campaign | Fixed cyan activity strip at top, text campaign with countdown/price highlights and `限时抢购` button |
| Top actions | Right aligned `灵感库`, `会员超市`, `会员中心`, `登录/注册` |
| Hero | Vertically centered title `图片创作`; subtitle `百变风格 超多模型供你选择`; small colorful model/style icons |
| Creator card | ~880px wide, rounded white card, soft shadow; top mode radios; 40px prompt textarea; upload/style tiles; model and parameter chips; disabled gray generate button until prompt exists |
| Inspiration | Bottom section `来试试一键做同款`; six 3:4 cards around 145×193px on 1200px viewport; `查看更多` right aligned |
| Login | Global full-screen modal, not embedded inside pages. Dark overlay, centered wide card, blue promo banner, left phone-code login, right WeChat QR login, QQ fallback, agreement links |

Logged-in capture for `/ai-tool/image-generator` and `/ai-tool/video-generator`:

| Area | Observed Structure |
|---|---|
| Auth dependency | Tool pages load `www.liblib.tv/cross-storage-hub.html`; video login state needs cross-domain storage/cookies beyond the document request cookie |
| Top actions | `灵感库`, `会员超市`, credit/VIP pill, notification bell, avatar dropdown; no `会员中心` or `登录/注册` button |
| Main content | The hero title/inspiration landing section disappears; the upper scroll region becomes a generation history feed |
| History feed | Left-aligned records with prompt, model/style/size/ratio chips, generated preview, actions `重新编辑`, `再次生成`, `...`, and `去LibTV，体验更多智能编辑能力` |
| Filters | Floating top-right filter card: `最近三个月` and `生成类型` |
| Composer | Fixed bottom center white rounded card, 880px wide; same mode tabs and controls as public page, but anchored at viewport bottom |
| Image composer | Active `图片生成`; uploads `风格模型`, `图片`; model `智能图片V2`; params `1:1 | 1张`; disabled credit button shows `18` |
| Video composer | Active `视频生成`; uploads `视频`, `图片`, `音频`; model `Seedance 2.0 VIP`; params `全能参考 | 智能比例 | 720p | 5s | 有配音`; disabled credit button shows `135` |

## 7. Backend Reverse Map

The client function surface implies these server modules:

| Module | Required Endpoints |
|---|---|
| Content CMS | `GET /content/home`, `GET /content/pages/:key` |
| Auth | `POST /auth/sms-code`, `POST /auth/login`, `GET /me`, `POST /auth/logout` |
| Search | `GET /search` with type/filter/sort/page |
| Models | `GET /models`, `GET /models/:id`, `POST /models`, favorite/use/follow actions |
| Works | `GET /works`, `GET /works/:id`, like/comment/do-same actions |
| Generate | `POST /generate/jobs`, `GET /generate/jobs/:id`, `GET /generate/jobs` |
| Assets | `POST /assets/upload`, `GET /assets`, favorite/delete/download actions |
| AI Apps | `GET /apps`, `GET /apps/:id`, `POST /apps/:id/jobs` |
| Teaching | `GET /teaching`, `GET /teaching/:id` |
| Billing | wallet/credits, membership plans, API plans, orders |
| Notifications | `GET /notifications`, mark-read |
