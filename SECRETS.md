# Supabase 自托管密钥备忘（勿提交 git / 勿外传）
> 完整版（含服务器登录、frp、全部 key、连接串）：`CREDENTIALS.md`

- 密钥实际存放于 docker/.env（600）
- Studio 控制台：http://8.152.204.58:54321/project/default
  - 用户名 supabase，密码 DASHBOARD_PASSWORD（见 docker/.env）
- Postgres 直连（frp）：host=8.152.204.58 port=54322 user=postgres
  - 密码 POSTGRES_PASSWORD（见 docker/.env）
- ANON_KEY / SERVICE_ROLE_KEY / JWT_SECRET 均在 docker/.env
- JWT 有效期 10 年（生成于本次部署日）
