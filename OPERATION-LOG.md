# Supabase 自托管：完整操作记录（交接用）

> 目的：让下一个 Agent / 运维无需重新踩坑，直接接手。
> 记录时间：2026-11-12　执行人：myflicker（远程操作，从 macOS 经 SSH 进入内网机）
> 无密钥（密钥见 `docker/.env` 与 `../SECRETS.md`）

---

## 0. 环境事实（先看这里）

| 项 | 值 |
|---|---|
| 内网机（本机） | Ubuntu 22.04，内核 6.8，用户 `zhaojian`（在 docker 组，无需 sudo 跑 docker） |
| 内网 IP | `192.168.2.2`（br0），出口是北京联通家宽 |
| 磁盘/内存 | 219G（用 35%）/ 15G（可用 12G） |
| Docker / Compose | 29.6.1 / Compose v5.3.1 |
| 阿里云 | `8.152.204.58`，root 账号（密码在 `../AGENTS.md`） |
| 公网入口 | 只能经阿里云 frps 转发（本机无公网 IP） |
| 已有服务 | 1Panel（自管容器）、guacamole(50322)、filebrowser(40071)、newapi(53000/55173/55174)、openviking(1933, systemd) —— **本次均未改动** |

关键机制：**frp 内网穿透**
- 阿里云跑 `frps`（容器 `frps`，镜像 `snowdreamtech/frps:latest`），配置 `/opt/frps/frps.toml`
  - `bindPort=7000`，`allowPorts = 50000..59999`，token `token 123456`（低强度，历史配置，未改）
  - **docker 层面是逐个 `-p` 发布端口的**（不是 host 网络），所以"frps 上允许"≠"公网可达"，
    新增对外端口必须在 `/opt/frps/docker-compose.yml` 里加 `ports` 再重建
- 本机跑 `frpc`（容器，host 网络，`restart: unless-stopped`），配置 `~/ubuntu/frpc/frpc.toml`
- 链路：公网 → 阿里云:remotePort → frps → frpc → 本机 127.0.0.1:localPort

⚠️ 两个易踩的坑：
1. 阿里云 frps 容器不发布端口 → 外部连不上（frpc 日志仍显示 `start proxy success`，具有误导性）
2. **`docker restart frpc` 会瞬间切断所有经 frp 的隧道，包括当前 SSH 会话**（SSH 自己就走 frp）。
   预期行为，等 5-10 秒重连即可。不要在无人值守时重启 frpc 后再跑关键操作。

---

## 1. 本次部署最终形态

```
公网 http://8.152.204.58:54321  →  frp  →  本机 127.0.0.1:54321  →  supabase-kong:8000
公网 8.152.204.58:54322         →  frp  →  本机 127.0.0.1:54322  →  supabase-pooler:5432
公网 http://8.152.204.58:54323  →  frp  →  本机 127.0.0.1:54323  →  supabase-studio:3000
```

| 公网端口 | 服务 | 说明 |
|---|---|---|
| 54321 | Kong API 网关 | `/rest/v1` `/auth/v1` `/storage/v1` `/realtime/v1` `/functions/v1` `/project/default`(Studio) |
| 54322 | supavisor 连接池 | 用户名必须带租户：`postgres.your-tenant-id` |
| 54323 | Studio 直连 | 备用入口（主入口走 Kong 的 `/project/default`，有 basic auth） |

容器共 11 个（studio/kong/auth/rest/realtime/storage/imgproxy/meta/functions/db/pooler），
全部 `restart: unless-stopped` 且 healthy。

---

## 2. 操作时间线（按顺序）

### 2.1 登录方式
本机已可直接免密 SSH（公钥已就绪）：
```bash
ssh -p 50122 zhaojian@8.152.204.58      # = 阿里云 frp 入口 → 本机 22
```
（本机密码为 `zhaojian`；macOS 无 sshpass，需交互时可 `expect`。）

### 2.2 克隆 supabase（指定稳定 tag）
```bash
cd ~/ubuntu
git clone --depth 1 -b v1.26.08 -c advice.detachedHead=false https://github.com/supabase/supabase.git supabase
```
选 tag 而非 main：避免上游随 HEAD 漂移，升级时按 `docker/upgrades.json` 走。

### 2.3 生成密钥并写 .env
在 `~/ubuntu/supabase/docker` 下：
```bash
cp .env.example .env && chmod 600 .env
```
替换为强随机值：`POSTGRES_PASSWORD`(hex16)、`JWT_SECRET`(sb_secret_+hex24)、
`SECRET_KEY_BASE`、`REALTIME_DB_ENC_KEY`、`VAULT_ENC_KEY`、`PG_META_CRYPTO_KEY`、
`LOGFLARE_PUBLIC/PRIVATE_ACCESS_TOKEN`、`S3_PROTOCOL_ACCESS_KEY_SECRET`、`DASHBOARD_PASSWORD`。

`ANON_KEY` / `SERVICE_ROLE_KEY` 用 python3 标准库手工签 HS256 JWT：
```python
# 头 {"alg":"HS256","typ":"JWS"}；载荷 {"role":"anon|service_role","iss":"supabase","iat":now-1d,"exp":now+10y}
# 签名 hmac.new(JWT_SECRET, f"{b64(header)}.{b64(payload)}", sha256)
```
（没有用 supabase CLI，因为只需标准库即可复现。）

**坑：`REALTIME_DB_ENC_KEY` 必须恰好 16 字符。** 首次误给 32 字符 →
realtime 日志 `(ErlangError) Erlang error: {:badarg, {~c"api_ng.c", 244}, ~c"Bad key size"}`
→ 容器无限 `Restarting (1)`。改成 `openssl rand -hex 8` 后 `docker compose up -d realtime` 恢复。

### 2.4 端口改动（docker-compose.yml，相对上游的 diff）

上游原始 → 现状：

| 原行 | 改为 |
|---|---|
| `- ${KONG_HTTP_PORT}:8000/tcp` | `- 127.0.0.1:${KONG_HTTP_PORT}:8000/tcp` |
| `- ${KONG_HTTPS_PORT}:8443/tcp` | `- 127.0.0.1:${KONG_HTTPS_PORT}:8443/tcp` |
| `- ${POSTGRES_PORT}:5432` | `- 127.0.0.1:54322:5432` |
| `- ${POOLER_PROXY_PORT_TRANSACTION}:6543` | `- 127.0.0.1:${POOLER_PROXY_PORT_TRANSACTION}:6543` |
| studio 服务 | 新增 `ports: ["127.0.0.1:54323:3000"]` |

⚠️ **绝对不要改 `.env` 里的 `POSTGRES_PORT`**：auth/rest/storage/meta/functions/pooler
内部连接串都用它连 `db:5432`，改了会全员挂掉。宿主对外端口直接写死 54322。

`.env` 追加/调整：
```
KONG_HTTP_PORT=54321
KONG_HTTPS_PORT=54324
SUPABASE_PUBLIC_URL=http://8.152.204.58:54321
API_EXTERNAL_URL=http://8.152.204.58:54321/auth/v1
```
全部端口只绑 `127.0.0.1`，公网暴露完全交给 frp，本机局域网不可直接访问。

### 2.5 镜像拉取（重点坑）
`docker compose pull` 用上了 daemon 里配的镜像加速源
（`/etc/docker/daemon.json`: docker.1panel.live / docker.1panel.dev / docker.1ms.run），
但 `supabase/storage-api` 在 1panel.live 上卡死在 `Pulling fs layer` 20+ 分钟，
整体 pull 还出现过 `short read: expected N bytes but got M: unexpected EOF`。

有效做法：**逐镜像显式指定 1ms.run 源拉取再打 tag**（1 分钟内完成）：
```bash
docker pull docker.1ms.run/supabase/storage-api:v1.60.4 \
 && docker tag docker.1ms.run/supabase/storage-api:v1.60.4 supabase/storage-api:v1.60.4
# studio / supavisor 同理
```
（外网 jsr.io / esm.sh / deno.land 从本机可直连，不是网络全阻，是单源不稳。）

### 2.6 启动
```bash
cd ~/ubuntu/supabase/docker && docker compose up -d
```
（`docker compose config --services` 可先校验 10 个服务名；
本机 Compose v5 **不支持** `docker compose ps --state`，请用 `ps -q` + `docker inspect`。）

### 2.7 frpc 增加转发（本机侧）
`~/ubuntu/frpc/frpc.toml` 末尾追加三条：
```toml
[[proxies]]
name = "supabase-kong"      # 54321 -> Kong
type = "tcp"; localIP = "127.0.0.1"; localPort = 54321; remotePort = 54321
[[proxies]]
name = "supabase-db"        # 54322 -> supavisor
type = "tcp"; localIP = "127.0.0.1"; localPort = 54322; remotePort = 54322
[[proxies]]
name = "supabase-studio"    # 54323 -> Studio
type = "tcp"; localIP = "127.0.0.1"; localPort = 54323; remotePort = 54323
```
生效：`docker restart frpc`（会断 SSH，属预期）。验证日志含
`proxy added: [... supabase-db supabase-kong supabase-studio]` + 各自 `start proxy success`。

### 2.8 阿里云 frps 发布新端口（关键一步，否则公网 000）
原本 frps 是手工 `docker run` 的（无 compose 管理的字面编排）。本次固化为
`/opt/frps/docker-compose.yml`，**保留原有全部端口**并新增 3 个：

```yaml
services:
  frps:
    image: snowdreamtech/frps:latest
    container_name: frps
    restart: always
    ports:
      - "7000:7000"      # frp 控制
      - "7500:7500"      # dashboard
      - "9119:9119"
      - "50022:50022"
      - "50071:50071"
      - "50122:50122"    # SSH（⚠️ 重建期间 SSH 会断）
      - "50222-50223:50222-50223"
      - "54321:54321"    # 本次新增
      - "54322:54322"    # 本次新增
      - "54323:54323"    # 本次新增
      - "58000:58000"
      - "58096:58096"
      - "58888:58888"
    volumes:
      - /opt/frps/frps.toml:/etc/frp/frps.toml:ro
```
执行（在阿里云上）：
```bash
cd /opt/frps && docker rm -f frps && docker compose up -d
ss -tln | grep -E ':(5432[123])'     # 应看到三行 LISTEN
```
阿里云安全组已放开 5xxxx 段（历史 50022/58000 等可直接访问，也据此推断本次三端口无需再改安全组）。


### 2.9 阿里云 nginx 增加域名（2026-11-12 追加）

域名 `supabase.zhaojian.xin` 已解析到 `8.152.204.58`（DNS 侧无需改动）。

阿里云实际对外的是 **docker 容器 `nginx`**（`nginx:latest`，发布 80），配置目录挂载
`/opt/nginx/conf.d`（容器内 `/etc/nginx/conf.d`），每个子域名一个 `.conf`，反代目标写
`172.17.0.1:<frps 端口>`。新增 `supabase.zhaojian.xin.conf`：

```nginx
server {
    listen 80;
    server_name supabase.zhaojian.xin;
    client_max_body_size 1G;            # storage 上传
    location / {
        proxy_pass http://172.17.0.1:54321;   # frps 发布的 Kong
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;      # Realtime WebSocket
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_read_timeout 600s;
    }
}
```
生效：`docker exec nginx nginx -t && docker exec nginx nginx -s reload`
（⚠️ reload 后立刻打请求可能命中正在退出的旧 worker，出现一次 502；稍等或重试即可。）

同时把本机 `.env` 的 `SUPABASE_PUBLIC_URL` / `API_EXTERNAL_URL` / `SITE_URL`
改为 `http://supabase.zhaojian.xin`，`docker compose up -d` 重建了
auth/storage/edge-functions/studio 四个容器。

验证（全部通过）：
```
auth/v1/health (apikey)      → 200
rest/v1/<table> (anon)       → 404 PGRST205（路由正常，表不存在）
storage/v1/bucket (anon)     → 200
/project/default (basic auth)→ 200
/realtime/v1/websocket       → HTTP/1.1 101 Switching Protocols
http://8.152.204.58:54321/…  → 200（IP 直连仍可用）
```

---

## 3. 验证结果（部署当日实测）

从**外部 Mac**：
```
API 54321 无 key → 401          （Kong 在，鉴权生效）
Studio 54323     → 307          （重定向到登录）
PG 54322         → TCP OPEN
```
从本机侧（完整业务链路，均通过 8.152.204.58 公网回绕）：
```
/auth/v1/health + apikey                → 200 {"name":"GoTrue",...}
/rest/v1/<table> + anon key             → 404 PGRST205（说明已过鉴权到 PostgREST，表不存在属正常）
/rest/v1/ + anon key                    → 403 "You cannot consume this service"（**预期**，见下）
/project/default + dashboard basic auth → 200（无凭据 401）
psql "postgresql://postgres.your-tenant-id:...@8.152.204.58:54322/postgres" → 成功，17.6
```
**`/rest/v1/` 根是 admin only**：本版本上游把 PostgREST OpenAPI 根（`rest-v1-openapi`）
的 acl 设为 `allow: [admin]`，因此 anon 访问根路径恒 403；anon 访问 `/rest/v1/<table>`
正常（路由 `rest-v1-all` 允许 anon/service_role）。不要误判为故障。

健康探测一行命令（宿主）：
```bash
A=$(grep '^ANON_KEY=' ~/ubuntu/supabase/docker/.env | cut -d= -f2-)
curl -s -o /dev/null -w '%{http_code}\n' 'http://8.152.204.58:54321/auth/v1/health' -H "apikey: $A"   # 200
```

Linux 无密探针（本机没装 psql，临时用 db 容器里的 psql，注意 hive 网关细节）：
```bash
cd ~/ubuntu/supabase/docker
P=$(grep '^POSTGRES_PASSWORD=' .env | cut -d= -f2-)
docker exec -e PGPASSWORD="$P" supabase-db \
  psql -h 8.152.204.58 -p 54322 -U postgres.your-tenant-id -d postgres -tAc 'select 1'
```
（`psql -h 127.0.0.1` 在 db 容器内是容器自身，不可用来测宿主端口；必须走公网地址或宿主。）

---

## 4. 开机自启（已核查，未做真实整机重启）

- `systemctl is-enabled docker containerd` → enabled；`systemctl is-active docker` → active
- 11 个 supabase 容器 + `frpc`：`RestartPolicy=unless-stopped`
- 阿里云 `frps`：`restart: always`（compose 固化后）
- `~/ubuntu/power-schedule/` 有历史断电/开机计划脚本，本次未改；重启后容器按策略自动起。
- 之前 `start_all_services.sh` 只覆盖 guacamole/filebrowser 等历史服务，**不含 supabase**，
  也不含阿里云 frps；supabase 与 frp 都靠 restart 策略自愈，无需加入该脚本。

真实重启验证建议（有维护窗口时）：`sudo reboot`，等 3-5 分钟，
`ssh -p 50122 ...` 重连后执行 `docker compose ps` 确认 11 个 healthy，并复查三端口。

---

## 5. 本次"未做/留给后面"的事

1. 全站 http 明文，无 TLS。要上生产需前置 Nginx（本机及阿里云都有 openresty）或 Cloudflare。
2. `POOLER_TENANT_ID` 仍是示例值 `your-tenant-id`（PG 用户名需带此后缀）。
   若要改：改 `.env` 后 `docker compose up -d` 会重建 pooler，客户端连接串同步改。
3. `OPENAI_API_KEY` 仍是占位符，Edge Functions 用 OpenAI 需替换。
4. frps `auth.token = "token 123456"` 强度低（历史配置），如需安全加固需同步改 frps/frpc。
5. Supabase 版本停在 v1.26.08；升级后**必须重新做** 2.4 的 compose 端口修改与 studio 端口。
6. 未配置任何备份策略。数据在 `~/ubuntu/supabase/docker/volumes/db/data`
   （另有 storage 等 volume 目录），建议加 cron 备份。

---

## 6. 一页速查

```bash
# 状态
cd ~/ubuntu/supabase/docker && docker compose ps
# 日志
docker compose logs -f kong      # auth / rest / realtime / storage / studio 同理
# 改配置后
docker compose up -d
# 重启 frpc（改动 frpc.toml 后；会断 SSH）
docker restart frpc
# 阿里云 frps 改端口后
ssh root@8.152.204.58 'cd /opt/frps && docker compose up -d'
# 密钥
cat ~/ubuntu/supabase/docker/.env ; cat ~/ubuntu/supabase/SECRETS.md
# 文档
cat ~/ubuntu/supabase/DEPLOYMENT.md
```
