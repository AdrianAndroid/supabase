# 测试环境全部凭据汇总（可外传中暴露，均为测试用途）

> 生成时间：2026-11-12　用途：下一次接手直接用，无需翻文件找密码。
> 内容：服务器登录、frp、Supabase 全部密钥与连接串、验证命令。
> 提示：这是**测试环境**，凭据可公开；一旦转生产请全部轮换。

---

## 1. 服务器登录

### 内网机（跑 Supabase 的"台式服务器"，192.168.2.2）
```
ssh -p 50122 zhaojian@8.152.204.58      # 密码：zhaojian（本质是经阿里云 frp 转发到内网机 22）
ssh zhaojian@192.168.2.2                 # 同机局域网直连（若在同网段）
```
- 用户 `zhaojian`，密码 `zhaojian`
- 有 docker 组权限：直接 `docker ...` 无需 sudo
- sudo 需要密码（未在本记录中确认，通常同登录密码）
- 已配置免密公钥（macOS 本机 `~/.ssh/id_rsa`）

### 阿里云服务器（8.152.204.58，公网入口 + frps + nginx）
```
ssh root@8.152.204.58                   # 密码：ZHAOjian199149.
```
- 用户 `root`，密码 `ZHAOjian199149.`

---

## 2. frp 内网穿透

| 项 | 值 |
|---|---|
| frps 服务端 | 阿里云容器 `frps`（`/opt/frps/frps.toml` + `/opt/frps/docker-compose.yml`） |
| frps 控制端口 | `7000` |
| frp 认证 token | `token 123456` |
| frps Dashboard | `http://8.152.204.58:7500` 用户名 `admin` 密码 `zhaojian` |
| frpc 客户端 | 内网机容器 `frpc`（host 网络，配置 `~/ubuntu/frpc/frpc.toml`） |
| 允许的远程端口段 | `50000-59999` |

已占用/常用端口：

| 端口 | 用途 |
|---|---|
| 50122 | SSH（内网机 22） |
| 50322 | guacamole 远程桌面 |
| 50071 | filebrowser（域名 file.zhaojian.xin） |
| 50222/50223 | 历史服务 |
| 58000 | 内网机 8000 |
| 58096 | jellyfin（域名 jellyfin.zhaojian.xin） |
| 58888 | custom-port |
| **54321** | **Supabase Kong API** |
| **54322** | **Supabase Postgres（supavisor）** |
| **54323** | **Supabase Studio 直连** |

---

## 3. Supabase 访问入口

| 用途 | 地址 |
|---|---|
| API 基址（域名，推荐） | `http://supabase.zhaojian.xin` |
| API 基址（IP 直连） | `http://8.152.204.58:54321` |
| Studio（经域名） | `http://supabase.zhaojian.xin/project/default` |
| Studio（直连） | `http://8.152.204.58:54323` |
| REST | `<API 基址>/rest/v1/` |
| Auth | `<API 基址>/auth/v1` |
| Storage | `<API 基址>/storage/v1` |
| Realtime(WS) | `ws://supabase.zhaojian.xin/realtime/v1/websocket` |
| Functions | `<API 基址>/functions/v1` |

Studio 登录：

| 项 | 值 |
|---|---|
| 用户名 | `supabase` |
| 密码 | `ac95a5aa5f03251a44be3e00` |

---

## 4. Supabase 密钥（全部）

### 4.1 对外 API 密钥
```
ANON_KEY
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXUyJ9.eyJyb2xlIjoiYW5vbiIsImlzcyI6InN1cGFiYXNlIiwiaWF0IjoxNzk0Mzk2NTczLCJleHAiOjIxMDk3NTY1NzN9.9IV3lu2Y29Ttwg4LH1rdk8XI3-y7U5iNNhp7rNj8DRs

SERVICE_ROLE_KEY
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXUyJ9.eyJyb2xlIjoic2VydmljZV9yb2xlIiwiaXNzIjoic3VwYWJhc2UiLCJpYXQiOjE3OTQzOTY1NzMsImV4cCI6MjEwOTc1NjU3M30.MK4cxBThAC637SZB8UUk5hf_ODMediJkt8qENnykv8M
```
- 两者均为 HS256 JWT，`iss=supabase`，有效期 10 年（至 2036-11）
- `ANON_KEY` 是公开可用的客户端 key（受 RLS 约束）
- `SERVICE_ROLE_KEY` 权限等同超级权限，只能放服务端

### 4.2 签名与内部密钥
```
JWT_SECRET
sb_secret_ec4446c67bd5647e6e52a391d1d75c71420cb88621b8bc5a

POSTGRES_PASSWORD
c47af637fa032f8862710f56cb37b159

SECRET_KEY_BASE
rEkG8PAqWG2Qqz0GJFd6mTwe10xi05zb0t0K8bZXdaHEkp5B728PXNYGG+K6iJPI

REALTIME_DB_ENC_KEY (必须 16 字符)
7fbda43dea72bbe7

VAULT_ENC_KEY
261a38fd33606178d3a27f5b30158f79

PG_META_CRYPTO_KEY
34e6ebe484b7ffac96842803e84149ba

LOGFLARE_PUBLIC_ACCESS_TOKEN
lg_public_4e81238d24a27259dbc5523341147c92

LOGFLARE_PRIVATE_ACCESS_TOKEN
lg_private_ac6537e2b5465be3b34c7a7e779eaf7c

S3_PROTOCOL_ACCESS_KEY_ID
625729a08b95bf1b7ff351a663f3a23c

S3_PROTOCOL_ACCESS_KEY_SECRET
6d00f8b39d543da0eea986ae8dfec30df3fe28ba7fbdb64233f7be28b0139b32

MINIO_ROOT_USER / PASSWORD
supa-storage / secret1234

POOLER_TENANT_ID（PG 用户名后缀）
your-tenant-id

OPENAI_API_KEY（占位符，未配置）
sk-proj-xxxxxxxx
```

> 单一事实来源：`~/ubuntu/supabase/docker/.env`（600）。本文件是它的快照，
> 改 `.env` 后记得同步这里。

---

## 5. 数据库连接串

域名方式：
```
postgresql://postgres.your-tenant-id:c47af637fa032f8862710f56cb37b159@supabase.zhaojian.xin:54322/postgres
```
IP 方式：
```
postgresql://postgres.your-tenant-id:c47af637fa032f8862710f56cb37b159@8.152.204.58:54322/postgres
```
- 必须带租户后缀：用户名 `postgres.your-tenant-id`（supavisor 要求）
- 池化（transaction）模式端口 `6543` 仅本机可达，未对外
- 容器内直连（无池化）：`docker exec -it supabase-db psql -U postgres`

---

## 6. 环境变量中的 URL 配置

```
SUPABASE_PUBLIC_URL=http://supabase.zhaojian.xin
API_EXTERNAL_URL=http://supabase.zhaojian.xin/auth/v1
SITE_URL=http://supabase.zhaojian.xin
KONG_HTTP_PORT=54321
KONG_HTTPS_PORT=54324
POSTGRES_PORT=5432   # 内部用，勿改
```

---

## 7. 常用验证命令（复制即用）

```bash
# 在内网机上
cd ~/ubuntu/supabase/docker
A=$(grep '^ANON_KEY=' .env | cut -d= -f2-)
curl -s "$(grep '^SUPABASE_PUBLIC_URL=' .env | cut -d= -f2-)/auth/v1/health" -H "apikey: $A"   # 200
curl -s -o /dev/null -w '%{http_code}\n' "http://supabase.zhaojian.xin/rest/v1/" -H "apikey: $A"  # 403 属预期（根是 admin only）
docker compose ps              # 11 个 healthy

# 数据库
docker exec -e PGPASSWORD="$POSTGRES_PASSWORD" supabase-db \
  psql -h supabase.zhaojian.xin -p 54322 -U postgres.your-tenant-id -d postgres -tAc 'select 1'
```

客户端接入示例（JS）：
```js
import { createClient } from '@supabase/supabase-js'
const supabase = createClient('http://supabase.zhaojian.xin', '<ANON_KEY>')
```

---

## 8. 其他已有服务凭据（不在本文件范围）

guacamole / filebrowser / newapi / 1Panel / openviking 等历史服务的凭据在各自目录：
`~/ubuntu/1panel`、`~/ubuntu/newapi`、`~/ubuntu/openviking/SECRETS.md`、
`~/ubuntu/docker/guacamole`。需要时去对应目录查。

---

## 9. 注意事项

1. 全链路 http 明文（无 TLS），只有测试用途才建议这样暴露。
2. `REALTIME_DB_ENC_KEY` 必须 16 字符，改了要重建 realtime。
3. `.env` 的 `POSTGRES_PORT` 不要改（内部连接依赖 5432）。
4. 新增公网端口：阿里云 `/opt/frps/docker-compose.yml` 加 `ports` → 重建 frps。
5. `docker restart frpc` 会断开当前 SSH。
6. 完整运维细节见 `OPERATION-LOG.md` / `DEPLOYMENT.md`。
