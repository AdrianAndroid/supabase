# Supabase 自托管部署说明（接力文档）

> 供其他 Agent / 运维接手使用。本文档不含密钥；密钥位置见「密钥管理」。
> 部署时间：2026-11-12　版本：supabase tag v1.26.08

## 拓扑

```
公网 → 阿里云 8.152.204.58(安全组+frps) → frp 隧道 → 本机 192.168.2.2
     → Docker Compose(supabase) 容器
```

| 组件 | 位置 |
|---|---|
| Supabase 仓库 | `~/ubuntu/supabase`（浅克隆，tag v1.26.08） |
| Compose | `~/ubuntu/supabase/docker`（`docker-compose.yml` 有本地修改） |
| 配置/密钥 | `docker/.env`（600） |
| frp 客户端 | 容器 `frpc`（host 网络，配置 `~/ubuntu/frpc/frpc.toml`） |
| frp 服务端 | 阿里云 `/opt/frps/frps.toml` + `/opt/frps/docker-compose.yml`，容器 `frps` |

## 端口映射

容器 → 宿主(仅 127.0.0.1) → frpc → 阿里云 frps（安全组放行 50000-59999）→ 公网：

| 用途 | 容器 | 宿主端口 | 公网地址 |
|---|---|---|---|
| Kong API 网关（REST/Auth/Storage/Realtime/Functions） | 8000 | 54321 | `http://8.152.204.58:54321` |
| Studio 控制台（经 Kong 主路径） | — | — | `http://8.152.204.58:54321/project/default` |
| Studio 直连（备用） | 3000 | 54323 | `http://8.152.204.58:54323` |
| Postgres（supavisor 连接池） | 5432 | 54322 | `8.152.204.58:54322` |
| Kong HTTPS | 8443 | 54324 | 未转发 |

- 阿里云 frps 若新增对外端口：编辑 `/opt/frps/docker-compose.yml` 的 `ports` 后
  `cd /opt/frps && docker compose up -d`（会自动重建 frps 容器，隧道自动重连）。
  注意 frps `allowPorts` 限定 50000–59999。

## 客户端接入

- REST：`http://8.152.204.58:54321/rest/v1/`（附 `apikey: <ANON_KEY>`）
- Auth：`/auth/v1`、Storage：`/storage/v1`、Realtime：`/realtime/v1`、Functions：`/functions/v1`
- Studio：`http://8.152.204.58:54321/project/default`，用户名 `supabase`，
  密码见 `.env` 的 `DASHBOARD_PASSWORD`
- Postgres：`postgresql://postgres.<POOLER_TENANT_ID>:<POSTGRES_PASSWORD>@8.152.204.58:54322/postgres`
  （supavisor 要求用户名带租户后缀，`POOLER_TENANT_ID` 当前为 `your-tenant-id`；
  如需改租户名，改 `.env` 后 `docker compose up -d` 重建 pooler）

## 密钥管理

> 全部凭据汇总（服务器/阿里云/frp/Supabase 全 key/连接串）：`CREDENTIALS.md`（600）

- 全部密钥仅在 `docker/.env`（600）与 `../SECRETS.md`（600）。
- `JWT_SECRET` / `ANON_KEY` / `SERVICE_ROLE_KEY` 本次新生成（HS256，10 年有效期）。
- `REALTIME_DB_ENC_KEY` **必须恰好 16 字符**（AES-128），否则 realtime 启动报
  `ErlangError: Bad key size` 并反复重启。
- 改 `.env` 后 `docker compose up -d`（会重建受影响容器）。

## 本地修改记录（相对上游 v1.26.08）

- `docker-compose.yml`：
  - 所有 ports 加 `127.0.0.1:` 绑定：Kong 8000/8443、supavisor 5432/6543
  - supavisor 宿主端口改为 `127.0.0.1:54322:5432`
  - **不要改 `POSTGRES_PORT` 变量本身**（内部服务都用它连 db:5432）
  - studio 增加 `ports: 127.0.0.1:54323:3000`
- `.env`：KONG_HTTP_PORT=54321、KONG_HTTPS_PORT=54324、
  SUPABASE_PUBLIC_URL/API_EXTERNAL_URL=http://8.152.204.58:54321…，
  其余密钥全部替换为强随机值。
- frpc.toml：新增 supabase-kong / supabase-db / supabase-studio 三个 `[[proxies]]`。
- 阿里云 `/opt/frps/docker-compose.yml`：新增 54321/54322/54323 三个 ports。

## 开机自启保障（已验证）

- `docker.service` / `containerd.service`：enabled ✓
- 全部 11 个 supabase 容器 + `frpc` + 阿里云 `frps`：`restart: unless-stopped`/`always` ✓
- 一般无需人工干预；`~/ubuntu/start_all_services.sh` 只覆盖部分历史服务，
  supabase 靠 restart 策略自恢复。

## 常用运维命令

```bash
cd ~/ubuntu/supabase/docker
docker compose ps                    # 全部 healthy 即正常
docker compose logs -f kong          # 单服务日志
docker compose up -d                 # 应用 .env/compose 变更（重建差异容器）
docker compose restart <svc>         # 重启单服务
docker compose down                  # 全停（数据保留在 ./volumes）
docker restart frpc                  # 改 frpc.toml 后生效（会瞬断经 frp 的 SSH）
```

- 升级：见上游 `docker/update.sh`、`upgrades.json`；升级后需重做上述
  docker-compose.yml 的 127.0.0.1 端口绑定与 studio 端口修改。
- 健康探测（宿主）：`curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:54321/rest/v1/ -H "apikey: <ANON_KEY>"` → 200（根路径 OpenAPI 为 admin only，anon 打数据路由才通）。

## 已知注意事项

- 公网纯 http 无 TLS，敏感生产场景建议前置 Nginx+证书或 Cloudflare。
- `/rest/v1/`（OpenAPI 根）上游设为 admin only：anon 访问返回 403
  `You cannot consume this service` 属预期，anon 正常访问数据路由 `/rest/v1/<table>`。
- edge-functions 首次启动需从 jsr.io / esm.sh 拉依赖，可能数分钟 unhealthy，
  日志出现 `main function started` 后转 healthy。
- 镜像拉取慢/断流时可显式指定镜像源：
  `docker pull docker.1ms.run/<image>:<tag> && docker tag docker.1ms.run/<image>:<tag> <image>:<tag>`
- `OPENAI_API_KEY` 为占位值；Edge Functions 调 OpenAI 需自行替换。

## 域名入口（2026-11-12 追加）

公网域名 **`supabase.zhaojian.xin`** → 阿里云 nginx 容器（`/opt/nginx/conf.d/supabase.zhaojian.xin.conf`）
→ `http://172.17.0.1:54321`（frps 发布的 Kong）→ 隧道 → 本机 Kong。

- 域名解析：`supabase.zhaojian.xin A 8.152.204.58`（同一泛域名下的既有约定，80 端口 http）
- 对外地址：
  - API：`http://supabase.zhaojian.xin`
  - Studio：`http://supabase.zhaojian.xin/project/default`（用户名 supabase / DASHBOARD_PASSWORD）
  - Realtime WebSocket：`ws://supabase.zhaojian.xin/realtime/v1/websocket`
  - 旧的 `http://8.152.204.58:54321` 仍可用（未关闭）
- `.env` 中 `SUPABASE_PUBLIC_URL` / `API_EXTERNAL_URL` / `SITE_URL` 已改为域名。
- 证书：目前无 TLS（与其他 `*.zhaojian.xin` 子域一致，仅 80）。要 https 需在阿里云 1Panel
  申请证书后改造该 conf 为 443 + 301。
- 注意：nginx 容器访问宿主端口必须用 docker0 网关 `172.17.0.1`（容器内 127.0.0.1 是容器自身），
  与 `jellyfin.zhaojian.xin.conf`（58096）的既有写法一致。

## Git 远端（2026-11-13 追加）

`~/ubuntu/supabase` 的远端已改：

| remote | URL | 用途 |
|---|---|---|
| `origin` | `git@github.com:AdrianAndroid/supabase.git` | 自己的仓库，日常提交/推送 |
| `upstream` | `https://github.com/supabase/supabase.git` | 上游，升级时拉取 |

- 当前分支：**`learn`**（跟踪 `origin/learn`），由 v1.26.08 的提交 + 本机改动构成，
  commit `1a52cffc chore(selfhost): docker compose port bindings + deployment/ops documentation (learn)`
- 仓库是浅克隆（`--depth 1`），升级到上游新版本建议重新 clone 再重做端口修改，
  不必强行 unshallow（体积 584MB）。
- ⚠️ `learn` 分支里已包含 `CREDENTIALS.md` / `SECRETS.md`（测试环境凭据），
  该 GitHub 仓库为 public，等于凭据公开。若要收紧：`git rm --cached` 这两个文件后
  加入 `.gitignore` 再提交（历史里仍可查到，需 rewrite 或轮换密钥）。
