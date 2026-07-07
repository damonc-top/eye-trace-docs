# MinIO 对象存储 — 运维手册

> 适用版本:本仓 `master` 分支当前快照,后端位于 `backend/`。
> 目标读者:需要在 dev / staging 手动往 MinIO 上传图片、并让 server / client 立刻能取到的工程师。

---

## 0. 一图看懂流程

```
┌──────────────────────────────────────────────────────────────────────┐
│  1. 工具就绪:MinIO 服务在跑,`mc` CLI 配好 alias                     │
│  2. 准备素材:本地有 PNG/JPG/WebP/GIF,知道宽高和用途                  │
│  3. 上传到 MinIO:`mc cp` 或 Web 控制台                              │
│  4. 写一行 assets 表:asset_id + storage_key + mime + size_bytes + ... │
│  5. 客户端引用:在 slot JSON 或 models 行里用 /api/v1/assets/{id}/content│
│  6. 验证:curl 那个 URL,200 + 正确的 Content-Type                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 1. 一次性环境检查

```bash
# MinIO 在跑?
lsof -nP -iTCP:9000 -sTCP:LISTEN
# 期望:有一行 LISTEN 在 127.0.0.1:9000

# mc 配过 alias 没?
mc alias list
# 期望看到 local → http://127.0.0.1:9000,access=eye-trace-admin

# 没配就配一下
mc alias set local http://127.0.0.1:9000 eye-trace-admin eye-trace-secret-32bytes-1234567890

# bucket 在?
mc ls local/
# 期望有 eye-trace-assets
```

bucket 当前的访问权限是 **download 公开**,意味着浏览器可以直接拉对象(走 MinIO 而不是 server 转发),方便 CDN 化。本文档后续命令都假设这个权限。

> 想关掉直链、改成全部走 server 代理?`mc anonymous set none local/eye-trace-assets`。

---

## 2. 对象 key 命名规范

server 从 `assets.storage_key` 字段直接读对象路径,所以 key 必须和你 INSERT 时写的一致。

| 用途 | key 模板 | 示例 |
|---|---|---|
| 单卡封面 | `cover/{id}.{ext}` | `cover/407.png` |
| 作者头像 | `avatar/{id}.{ext}` | `avatar/408.jpg` |
| Banner / 运营图 | `banner/{id}.{ext}` | `banner/501.jpg` |
| 多图轮播 / gallery | `gallery/{id}.{ext}` | `gallery/201.png` |
| 用户生成结果 | `output/{id}.{ext}` | `output/9001.png` |

`{id}` 必须等于你即将 INSERT 的 `assets.id`(AUTO_INCREMENT 顺序号)。
`{ext}` 是小写扩展名 png / jpg / jpeg / webp / gif。

> 偷懒命名也可以,但**绝不**用 `..`、空格、中文;`storage.LocalDisk` 在 `Open` 里有 path traversal 防护,S3 那边虽然没拦但属于坏实践。

---

## 3. 上传素材到 MinIO

两种方式任选其一。

### 方式 A:用 mc 命令行

```bash
# 上传一张 banner 图,key 临时写成 banner/501.jpg(数字与后面 INSERT 的 id 对齐)
mc cp ~/Downloads/summer-promo.jpg \
   local/eye-trace-assets/banner/501.jpg

# 验证对象到位
mc ls local/eye-trace-assets/banner/
mc stat local/eye-trace-assets/banner/501.jpg
# 期望:Size / ETag / Content-Type 都打印
```

### 方式 B:Web 控制台

1. 浏览器打开 `http://127.0.0.1:9001`(MinIO 自带 console)
2. 登录:user = `eye-trace-admin`,password = `eye-trace-secret-32bytes-1234567890`
3. 左侧 Buckets → `eye-trace-assets` → 上传按钮 → 拖文件 / 选文件
4. 上传完点文件名 → 右下角 Share / Copy path,确认 key 对得上规范

---

## 4. 写一行 assets 表

每个上传好的对象,在 MySQL 里**必须**有一行 assets 记录,否则 server 的 `/api/v1/assets/{id}/content` 不知道对象存在,直接 404。

```sql
-- 把刚才上传的 banner/501.jpg 登记成 asset_id = 501(手动指定,跳过 AUTO_INCREMENT)
INSERT INTO assets (
  id, owner_type, purpose, media_type, mime, size_bytes,
  storage_backend, storage_key, width, height
) VALUES (
  501,                -- 与 key 里数字一致;不填就走 AUTO_INCREMENT
  'admin',            -- 'admin' / 'user' / 'system' 三选一
  'banner',           -- 'cover' / 'preview' / 'gallery' / 'avatar' / 'output' / 'banner'
  'image',            -- 'image' / 'gif' / 'webp' / 'video',跟 mime 对齐
  'image/jpeg',
  245678,             -- 文件字节数(mc stat 出来的 Size)
  's3',               -- 永远是 's3' —— 不要再用 'local'
  'banner/501.jpg',   -- 必须跟 mc cp 的目标路径一字不差
  1200,               -- 像素宽(从图片工具或 mc stat 拿)
  400                 -- 像素高
);
```

校验:

```bash
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "SELECT id, purpose, media_type, mime, size_bytes, storage_backend, storage_key FROM assets WHERE id = 501;"
# 期望:1 行,storage_backend='s3',storage_key='banner/501.jpg'
```

---

## 5. 让前端真的能拿到图

根据用途走不同入口。

### 5a. Banner / 运营图 — 改 slot JSON

`home_content_slots.banner_carousel.content_json` 里的 `carousel[].src` 和 `sideCards[].src`:

```sql
UPDATE home_content_slots
SET content_json = JSON_SET(
  content_json,
  '$.carousel[0].src', '/api/v1/assets/501/content'
  -- 再加更多就续行:,'$.carousel[1].src','/api/v1/assets/502/content'
),
version = version + 1
WHERE slot_key = 'banner_carousel';
```

`recommend.models[].icon` 同理:

```sql
UPDATE home_content_slots
SET content_json = JSON_SET(
  content_json,
  '$.models[0].icon', '/api/v1/assets/601/content'
),
version = version + 1
WHERE slot_key = 'recommend';
```

### 5b. 模型封面 / 头像 — 改 models 行

```sql
-- 把模型 id=42 的封面换成刚上传的 asset 501
UPDATE models SET cover_asset_id = 501 WHERE id = 42;
-- 头像
UPDATE models SET author_avatar_id = 502 WHERE id = 42;
```

### 5c. 不需要任何代码改动

`/api/v1/assets/{id}/content` 这个端点 server 已经实现了 — 它从 assets 表查 storage_key,再读 MinIO,流回浏览器。**只要 URL 是这个形式、assets 行存在、MinIO 对象在,前端立刻能用**。前端 React 代码不感知。

---

## 6. 验证

```bash
# 1) MinIO 直链(如果 bucket 是 download 公开)
curl -sI http://127.0.0.1:9000/eye-trace-assets/banner/501.jpg
# 期望:200 OK,Content-Type: image/jpeg,Content-Length 接近你 INSERT 的 size_bytes

# 2) server 代理(永远可用,即使 bucket 改 private)
curl -sI http://127.0.0.1:8080/api/v1/assets/501/content
# 期望:200,Content-Type: image/jpeg,ETag 是 sha256 截 32

# 3) 端到端,浏览器加载
mcpe  'http://localhost:1420/?cb=now'
# 浏览器 devtools Network 过滤 type=image,看到 status=200 的请求 src 是 /api/v1/assets/501/content
```

---

## 7. 常见操作脚本

### 7.1 删一个对象 + 它的 assets 行

```bash
# 先找出 asset_id
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "SELECT id, storage_key, purpose FROM assets WHERE storage_key = 'banner/501.jpg';"

# 删对象
mc rm local/eye-trace-assets/banner/501.jpg

# 删行(先确认没被 models / slots 引用)
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "DELETE FROM assets WHERE id = 501 AND NOT EXISTS (
        SELECT 1 FROM models WHERE cover_asset_id = 501 OR author_avatar_id = 501
      );"
```

### 7.2 替换一个对象但保留 asset_id

```bash
# 改文件但不改 key,前端无感
mc cp ~/Downloads/new-promo.jpg \
   local/eye-trace-assets/banner/501.jpg --force

# 如果扩展名变了(mp4 → webp),需要 UPDATE mime + media_type + storage_key
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "UPDATE assets SET storage_key='banner/501.webp', mime='image/webp', media_type='webp' WHERE id = 501;"
```

### 7.3 看哪些 banner / recommend 当前是空的

```bash
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "SELECT slot_key, JSON_LENGTH(content_json, '\$.carousel') AS carousel_len,
             JSON_LENGTH(content_json, '\$.sideCards') AS side_len
      FROM home_content_slots WHERE slot_key = 'banner_carousel';"
```

### 7.4 一键 dump 所有 s3 对象清单 + 对应 assets 行

```bash
mc ls --recursive local/eye-trace-assets/ > /tmp/objects.txt
mysql -ueye_trace -peye_trace -h127.0.0.1 eye_trace \
  -e "SELECT id, purpose, storage_key, size_bytes FROM assets WHERE storage_backend='s3' ORDER BY id;" \
  > /tmp/assets.txt
diff <(awk '{print $NF}' /tmp/objects.txt) <(awk 'NR>1{print $3}' /tmp/assets.txt)
# 输出左-only = 孤儿对象(没登记),右-only = 孤儿行(对象没了)
```

---

## 8. 启动 MinIO(dev 模式)

MinIO 数据目录已统一到仓库内 `eye-trace-res/minio-data/`(与 MySQL 的 `eye-trace-db/mysql-data/` 同模式,被根 `.gitignore` 排除),启动命令以仓库根为 cwd。

如果 MinIO 没跑,装一下:

```bash
brew install minio/stable/minio minio/stable/mc   # macOS
```

起服务:

```bash
cd /Users/mac/github/eye-trace-ai
mkdir -p eye-trace-res/minio-data
MINIO_ROOT_USER=eye-trace-admin \
MINIO_ROOT_PASSWORD=eye-trace-secret-32bytes-1234567890 \
nohup minio server ./eye-trace-res/minio-data \
  --address 127.0.0.1:9000 \
  --console-address 127.0.0.1:9001 \
  > /tmp/minio.log 2>&1 &
disown
```

挂上 server 端的 env,启动 server:

```bash
cd backend
export STORAGE_BACKEND=s3
export STORAGE_S3_ENDPOINT=127.0.0.1:9000
export STORAGE_S3_BUCKET=eye-trace-assets
export STORAGE_S3_ACCESS_KEY=eye-trace-admin
export STORAGE_S3_SECRET_KEY=eye-trace-secret-32bytes-1234567890
export STORAGE_S3_REGION=us-east-1
./bin/server
# 日志期望:s3 storage ready,bucket=eye-trace-assets,endpoint=127.0.0.1:9000
```

---

## 9. 别踩这些坑

1. **storage_key 写错字**(大小写、漏 `/`、多空格)→ server 报 "storage: not found",5xx。`mc ls` 看一下实际对象路径对照。
2. **media_type 与 mime 不匹配** → 浏览器可能拒绝渲染。`media_type='image'` 对应 `mime='image/png'` / `image/jpeg`;`media_type='webp'` 对应 `image/webp`;`media_type='gif'` 对应 `image/gif`。
3. **storage_backend='local' 但 storage_key 是 s3 路径(或反之)** → 同样 5xx。schema 有 ENUM,这里一旦写错,server 不会替你翻译。
4. **删除对象但忘删 assets 行** → 客户端拿到 404 错误对象。最好先按 7.4 对账再删。
5. **忘了 bump slot 的 `version`** → server 端 `home_content_slots` 的 ETag 不会变,客户端的 ETag 缓存 30 天可能拉不到新内容。UPDATE 时务必 `version = version + 1`。
6. **Owner 是 user,但该用户不存在** → 目前 server 不校验,等 Phase 3 多租户上线会强制 owner_id 落在 tenant_members 范围。先在 admin 端检查。
7. **在生产 MinIO bucket 上跑 `mc rm --recursive --force`** → 全删了,没有回收站。建议所有删除走 `mc rm --dry-run` 先演练。

---

## 10. 相关文档

- `eye-trace-server/backend/DEPLOY.md` §1 / §8:MinIO 在生产部署中的位置
- `eye-trace-server/backend/internal/asset/repo.go`:assets 表的 repo
- `eye-trace-server/backend/pkg/storage/s3.go`:MinIO 后端实现
- `eye-trace-server/backend/cmd/importer/liblib`:批量抓 liblib 模型 + 上传 MinIO
- `eye-trace-server/backend/cmd/importer/banner`:批量把 slot 里 liblib URL 抓下来并替换

**写完素材、插完行、跑完验证三个命令,前端刷新就生效,不用动 React 代码。**