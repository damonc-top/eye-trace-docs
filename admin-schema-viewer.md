# admin-schema-viewer — 内部只读 DB 结构浏览器

> 状态:**已落地**(后端 `cmd/admin-server` + 前端 `eye-trace-server/admin/`)。
> 定位:内部开发工具,只读,不入 `/api/v1` 公开契约。用于**查看 MySQL 表结构是否符合设计预期**,快速纠正 AI 大模型在设计与编码过程中产出的 DB 结构偏差。

## 它是什么 / 不是什么

| 是 | 不是 |
|---|---|
| 只读 schema 浏览器(表结构/列/索引/外键/样本/migration 来源) | 产品级运营后台(那是 Phase 3) |
| SELECT-only 自定义查询控制台(三层写守卫) | DB 数据编辑器(禁止任何写操作) |
| 内部开发者工具,绑 `127.0.0.1`,无鉴权 | 对外服务,带 RBAC/审计 |
| 复用主 server 的 config.yaml + MySQL 连接 | 独立数据源 |

适用场景:AI 大模型生成的 migration / model 结构是否符合预期?跑 `make admin` 打开浏览器直接看,比 `mysql` CLI 直观。

## 启动

```bash
cd eye-trace-server/backend

# 方式 A:只起后端(前端走 Vite dev,热更新,推荐开发用)
make admin                        # 后端 :8091
cd ../admin && npm install && npm run dev   # 前端 :5173,自动 proxy /api → :8091
# 浏览器打开 http://localhost:5173

# 方式 B:构建前端后单进程托管(无需 Vite)
make admin-build                  # npm install + npm run build → ../admin/dist
make admin-run                    # :8091,静态托管 dist + /api
# 浏览器打开 http://127.0.0.1:8091
```

前置条件:
- 项目 MySQL 在跑(`make mysql-start`,端口 3307)。
- `DB_DSN` 或 `config.yaml` 可连(复用主 server 同一份配置)。
- 方式 A 需 `npm install` 过 `eye-trace-server/admin/`;方式 B 由 `admin-build` 自动装。

环境变量(均可选):

| 变量 | 默认 | 说明 |
|---|---|---|
| `ADMIN_ADDR` | `127.0.0.1:8091` | 监听地址,**强制 localhost**,非 loopback 启动即拒 |
| `MIGRATIONS_DIR` | `migrations` | migration 扫描目录(相对工作目录) |
| `ADMIN_DIST_DIR` | `../admin/dist` | 前端构建产物目录;不存在时 `/` 显示构建提示,`/api` 仍可用 |
| `DB_DSN` / `CONFIG_PATH` | — | 同主 server |

## 6 个端点(全部 `/api`,只读)

| 方法 路径 | 用途 |
|---|---|
| `GET /api/schema/tables` | 表清单 + 行数估计 + 大小 |
| `GET /api/schema/tables/{table}` | 列/类型/注释/索引/外键 |
| `GET /api/schema/tables/{table}/sample?limit=50` | 前 N 行(只读 `SELECT *`,limit clamp [1,200]) |
| `GET /api/schema/migrations` | 全部 migration 文件 + table→migration 反向索引 |
| `GET /api/schema/migrations/{table}` | 命中该表的 migration 段落(含设计注释 + DDL 摘要) |
| `POST /api/query` `{"sql":"SELECT..."}` | SELECT-only 自定义查询 |

错误统一 `application/problem+json`,字段 `detail` 给人看。

## 只读保证(三层防御)

`POST /api/query` 的 SQL 守卫:

1. **首关键字白名单**:必须以 `SELECT` / `WITH` / `EXPLAIN` / `DESCRIBE` / `SHOW` 开头。
2. **关键词黑名单**(词边界匹配,大小写不敏感):`INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|REPLACE|GRANT|REVOKE|RENAME|LOAD|CALL|SET|LOCK|UNLOCK|USE|...` 命中即 400。
3. **DB 层兜底**:`BeginTx(ReadOnly:true)` 开 READ ONLY 事务,前两层漏放时 DB 仍拒绝写。

附加保护:无 `LIMIT` 的 SELECT/WITH 自动注入 `LIMIT 200`;结果达到 200 行标记 `truncated:true`。表名入参(样本端点)先校验存在于 `information_schema`,杜绝注入。

样本/查询返回的 `[]byte`(MySQL driver 对 VARCHAR/BLOB 的默认返回)在后端转 string,前端可直接读中文/字符串,无 base64。

## 前端能力(4 个 Tab)

- **结构**:列卡片(类型/可空/键/默认/Extra/注释)+ 索引 + 外键。
- **样本数据**:前 N 行表格(N=20/50/100/200 可选),`NULL` 标灰。
- **Migration 来源**:命中该表的 migration 按时间序列出,展开看「设计意图注释 + DDL 头部」,直接对照设计。
- **SQL 控制台**:只读查询,`Ctrl/Cmd+Enter` 执行;结果单元格点击纯标识符可跳转该表。

## 架构边界(不污染主 server)

- 独立二进制 `cmd/admin-server`,独立端口,独立进程。关掉对线上零影响。
- 复用 `pkg/config.Load` + `internal/db.Open`(同一个 MySQL),**不改** `Config` 结构。
- **不进** `openapi.yaml`(工具不是契约),**不动** `cmd/server`。
- 不挂 JWT/tenant/ratelimit,靠 localhost 绑定网络层隔离。
- 前端 `eye-trace-server/admin/` **独立 package.json**,不共享 client 依赖。

## 文件清单

后端(`eye-trace-server/backend/`):
- `cmd/admin-server/main.go` — 入口,强制 localhost + 静态托管 + CORS。
- `internal/schema/repo.go` — `information_schema` 只读查询(表/列/索引/外键/样本)。
- `internal/schema/migrations.go` — migration 文件反向索引(regex 提取 CREATE/ALTER TABLE)。
- `internal/schema/query.go` — SELECT-only 守卫 + READ ONLY 事务执行器。
- `internal/schema/handler.go` — 6 个 HTTP 端点薄层。

前端(`eye-trace-server/admin/`):
- `src/App.tsx` — 主布局(左侧栏 + 4 Tab)。
- `src/api.ts` — fetch 封装。
- `src/views/{TableList,TableDetail,SampleData,Migrations,QueryConsole}.tsx`。
- `src/styles.css` — 暗色主题,无 UI 框架。

工程:
- `backend/Makefile`:`admin` / `admin-build` / `admin-run`。

## 后续可扩展(本期不做)

- MinIO 对象浏览(`/api/storage/objects`)。
- 多 DB / 多环境切换。
- 与 Phase 3 产品级运营后台合并(届时转为带鉴权的可编辑后台)。
