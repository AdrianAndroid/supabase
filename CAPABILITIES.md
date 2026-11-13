# Supabase 能用来干什么：能力与限制分析

> 写在 2026-11-12，基于**本机实际探测到的配置**（不是官方文档的通用介绍）。
> 探测命令见文末附录，配置事实来源 `docker/.env`、`docker/docker-compose.yml`。
> 凭据见 `CREDENTIALS.md`；运维细节见 `OPERATION-LOG.md` / `DEPLOYMENT.md`。

---

## 一、当前实际提供的能力（已实测）

| 能力 | 组件 | 现状 |
|---|---|---|
| 关系库 | Postgres **17.6** | 单实例，数据在 `docker/volumes/db/data` |
| 自动 REST API | PostgREST v14 | 建表即得 `/rest/v1/<表>` CRUD，RLS 控制权限 |
| GraphQL | pg_graphql 扩展 | 可用（未启用，`create extension pg_graphql` 即可） |
| 认证 | GoTrue (auth v2.189) | 邮箱密码注册开启；JWT 有效期 **3600s** |
| 文件存储 | storage-api + imgproxy | 本地文件后端，**单文件 50MB**，支持签名 URL 与图片实时转码 |
| 实时 | Realtime v2.102 | WebSocket 已实测 `101`；表变更订阅 / broadcast / presence |
| Serverless | Edge Functions (Deno) | 示例函数 `hello`、`main`；`FUNCTIONS_VERIFY_JWT=false`（网关鉴权即可） |
| 管理台 | Studio 2026.08 | 表编辑器、SQL 编辑器、RLS 策略、日志、API 文档 |
| 连接池 | supavisor 2.9.5 | 对外 54322；用户名必须带租户后缀 `postgres.your-tenant-id` |
| 网关 | Kong 3.9.3 | `/rest /auth /storage /realtime /functions /project` 统一入口 |

### 可用（已安装、未启用）的 Postgres 扩展

```
vector(pgvector)  postgis  pg_cron  pg_net  pg_trgm  pgmq  pgjwt
pgsodium  supabase_vault  uuid-ossp  hypopg  index_advisor
pg_graphql  pg_jsonschema  rum  wrappers  http
```

已启用：`pg_net`、`pg_stat_statements`、`pgcrypto`、`supabase_vault`、`uuid-ossp`、`plpgsql`。
启用方式：Studio SQL 编辑器执行 `create extension if not exists <name>;`

---

## 二、适合拿它做什么

1. **全栈应用后端**：表 → 前端直连 REST/Realtime，省掉整层后端。适合 H5/小程序/个人 SaaS 的 MVP。
2. **AI / RAG 应用**：`pgvector` 存 embedding + REST 检索 + Edge Functions 调模型；
   `pg_cron` + `pg_net` 做定时任务与外部 HTTP 调用。
3. **实时协作**：聊天、评论流、在线看板、多人编辑提示（postgres_changes + presence）。
4. **内部工具 / 数据后台**：Studio 可视化建表，当低代码后端数据库用。
5. **文件/素材管理**：bucket + 签名 URL + imgproxy 实时缩放/转格式，可做图床、附件服务。
6. **IoT / 采集上报**：脚本 `POST /rest/v1/表` 写数据；`pgmq` 当轻量队列。
7. **地理数据**：`postgis` 做定位/范围查询。

---

## 三、当前真正会卡住的点（务必看）

| # | 问题 | 实际影响 | 修法 |
|---|---|---|---|
| 1 | **SMTP 是假的**（`SMTP_HOST=supabase-mail`、`ENABLE_EMAIL_AUTOCONFIRM=false`） | 邮箱注册后收不到确认邮件 → 真实用户注册流程走不通 | 配真实 SMTP；或临时 `ENABLE_EMAIL_AUTOCONFIRM=true` |
| 2 | 手机注册开着（`ENABLE_PHONE_SIGNUP=true`）但**无短信网关** | 手机登录实际不可用 | 关掉或接短信服务商 |
| 3 | 匿名登录关闭（`ENABLE_ANONYMOUS_USERS=false`） | 匿名试用场景不可用 | 需要时置 true 并重建 auth |
| 4 | **单文件 50MB**（`FILE_SIZE_LIMIT: 52428800`，STORAGE_BACKEND=file） | 大文件/视频上传失败 | 调大限制；大规模上 S3/OSS 后端（`docker-compose.s3.yml`） |
| 5 | JWT 有效期 1 小时 | 前端需实现 refresh token 续期 | 可调 `JWT_EXPIRY`，不建议调太大 |
| 6 | **纯 http 无 TLS + 公网可直连** | 密码/token 明文；`service_role` 泄漏=数据库超管 | 上证书（1Panel 免费证书）或限定内网/IP 白名单 |
| 7 | 单机、无 HA、**无备份** | 家宽断电/硬盘故障 = 数据全丢 | 定时 `pg_dump` + 备份 `docker/volumes/` |
| 8 | 无 WAL 归档 / PITR | 只能恢复到最近一次逻辑备份 | 同上方案 |
| 9 | 跑在家庭宽带上行 | 对外并发/带宽有限、延迟不稳 | 只当测试/个人用；正式业务需上云主机 |
| 10 | 根路径 `/rest/v1/` 403（admin only） | 用 curl 探活会误判 | 探测用 `/auth/v1/health` 或带表的路径 |

---

## 四、结论

这是一套**开箱即用的完整 BaaS**：数据库 + 认证 + 文件 + 实时 + Serverless + 管理台。

- 适合：个人项目、内部系统、AI 应用原型、IoT 采集、自动化任务。
- 不适合直接承载有真实用户注册的正式线上业务——缺 **SMTP、TLS、备份** 三件。
- 补齐这三项（+ 按需调文件大小与 S3 后端）即可支撑小规模生产。

---

## 五、常见需求的改造指引

```bash
cd ~/ubuntu/supabase/docker

# 1) 临时放开发确认（测试环境最省事）
sed -i 's/^ENABLE_EMAIL_AUTOCONFIRM=.*/ENABLE_EMAIL_AUTOCONFIRM=true/' .env

# 2) 配真实 SMTP（示例：企业邮箱/QQ/163 都可）
sed -i 's/^SMTP_HOST=.*/SMTP_HOST=smtp.example.com/' .env
sed -i 's/^SMTP_PORT=.*/SMTP_PORT=465/' .env
sed -i 's/^SMTP_USER=.*/SMTP_USER=no-reply@example.com/' .env
sed -i 's/^SMTP_PASS=.*/SMTP_PASS=<授权码>/' .env
sed -i 's/^SMTP_ADMIN_EMAIL=.*/SMTP_ADMIN_EMAIL=no-reply@example.com/' .env

# 3) 放宽单文件上限（原位：docker-compose.yml 的 FILE_SIZE_LIMIT，单位字节；此处示例 500MB）
#    FILE_SIZE_LIMIT: 524288000
#    同时 nginx 侧 client_max_body_size 已设 1G

# 4) 关掉手机注册（无短信网关时）
sed -i 's/^ENABLE_PHONE_SIGNUP=.*/ENABLE_PHONE_SIGNUP=false/' .env

# 应用改动（只重建受影响的容器）
docker compose up -d

# 5) 开启 pgvector（做 embedding/检索）
docker exec -it supabase-db psql -U postgres -d postgres -c 'create extension if not exists vector;'

# 6) 备份（建议 cron，每天一次，保留 7 天）
#    pg_dumpall 走容器，存到 ~/backup：
#    /usr/bin/docker exec supabase-db pg_dumpall -U postgres | gzip > ~/backup/supa-$(date +\%F).sql.gz
```

---

## 附录：本分析用的探测命令

```bash
cd ~/ubuntu/supabase/docker
docker exec supabase-db psql -U postgres -d postgres -tAc \
  "select name from pg_available_extensions where name in ('vector','pg_cron','pg_net','postgis','pgcrypto','uuid-ossp','pg_stat_statements','pgsodium','pgjwt','pg_graphql','hypopg','index_advisor','supabase_vault','rum','pg_trgm','pgmq','pg_jsonschema','http','wrappers') order by name;"
docker exec supabase-db psql -U postgres -d postgres -tAc "select extname from pg_extension order by 1;"
grep -E '^(ENABLE_EMAIL_SIGNUP|ENABLE_EMAIL_AUTOCONFIRM|ENABLE_PHONE_SIGNUP|ENABLE_ANONYMOUS_USERS|JWT_EXPIRY|PGRST_DB_SCHEMAS|FUNCTIONS_VERIFY_JWT)=' .env
grep -E 'STORAGE_BACKEND|FILE_SIZE' docker-compose.yml
docker stats --no-stream
```
