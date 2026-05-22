# MoEmail 部署与换机恢复指南

本文档记录当前可用的 Cloudflare 部署配置，方便换电脑后继续维护或重新部署。

## 当前可用地址

- Pages 项目：`moemail-github`
- 访问地址：`https://moemail-github.pages.dev`
- GitHub 仓库：`https://github.com/zanderwang9999/moemail.git`
- 生产分支：`master`

## 当前 Cloudflare 资源

- Account ID：`1baed04f83383be27d297bd6e7d17308`
- D1 数据库绑定：`DB`
- D1 数据库名：`moemail`
- D1 数据库 ID：`eaa3c072-6961-4143-9734-647de59866b7`
- KV 绑定：`SITE_CONFIG`
- KV Namespace ID：`8d97e2de9d74422092597e459d235d21`
- Email Worker：`email-receiver-worker`
- Cleanup Worker：`cleanup-worker`
- 收件域名：`openwang.eu.cc`

## 关键运行配置

`wrangler.toml` 已提交，包含：

- `name = "moemail-github"`
- `compatibility_flags = ["nodejs_compat"]`
- `pages_build_output_dir = ".vercel/output/static"`
- D1 绑定 `DB`
- KV 绑定 `SITE_CONFIG`

Cloudflare Pages 构建设置：

- 框架预设：`Next.js`
- 构建命令：`pnpm install && pnpm build:pages`
- 构建输出目录：`.vercel/output/static`
- 生产分支：`master`

## 必须配置的 Pages 环境变量

在 Cloudflare Pages 项目 `moemail-github` 的生产环境中配置：

```text
AUTH_SECRET=<随机强密钥>
AUTH_TRUST_HOST=true
AUTH_URL=https://moemail-github.pages.dev
NEXTAUTH_URL=https://moemail-github.pages.dev
CLOUDFLARE_ACCOUNT_ID=1baed04f83383be27d297bd6e7d17308
```

生成 `AUTH_SECRET` 的 PowerShell 命令：

```powershell
$bytes = New-Object byte[] 32; $rng = [Security.Cryptography.RandomNumberGenerator]::Create(); $rng.GetBytes($bytes); [Convert]::ToBase64String($bytes)
```

也可以使用 Wrangler 写入 Pages secret：

```powershell
wrangler pages secret put AUTH_SECRET --project-name moemail-github
wrangler pages secret put AUTH_TRUST_HOST --project-name moemail-github
wrangler pages secret put AUTH_URL --project-name moemail-github
wrangler pages secret put NEXTAUTH_URL --project-name moemail-github
wrangler pages secret put CLOUDFLARE_ACCOUNT_ID --project-name moemail-github
```

## 必须配置的 KV 值

当前站点使用 `openwang.eu.cc` 收信，必须在 `SITE_CONFIG` 中设置：

```powershell
wrangler kv key put EMAIL_DOMAINS "openwang.eu.cc" --namespace-id 8d97e2de9d74422092597e459d235d21
```

验证：

```powershell
wrangler kv key get EMAIL_DOMAINS --namespace-id 8d97e2de9d74422092597e459d235d21
```

预期输出：

```text
openwang.eu.cc
```

## Email Routing 配置

Cloudflare 域名：`openwang.eu.cc`

路径：

```text
openwang.eu.cc → 电子邮件 → 电子邮件路由 → 路由规则
```

必须存在：

```text
Catch-all 地址 → 发送到 Worker → email-receiver-worker → 启用
```

MX 记录应指向 Cloudflare：

```text
route1.mx.cloudflare.net
route2.mx.cloudflare.net
route3.mx.cloudflare.net
```

SPF 记录：

```text
v=spf1 include:_spf.mx.cloudflare.net ~all
```

## 换电脑恢复步骤

1. 安装 Node.js 20、Git、pnpm。
2. 克隆仓库：

```powershell
git clone https://github.com/zanderwang9999/moemail.git
cd moemail
```

3. 安装依赖：

```powershell
pnpm install
```

4. 登录 Cloudflare：

```powershell
pnpm exec wrangler login
```

5. 如需重新部署 Email Worker：

```powershell
pnpm run deploy:email
```

6. 如需重新部署 Cleanup Worker：

```powershell
pnpm run deploy:cleanup
```

7. Pages 推荐通过 GitHub 自动构建。推送到 `master` 即会触发部署：

```powershell
git push
```

## 验证命令

检查 Auth：

```powershell
curl.exe -i https://moemail-github.pages.dev/api/auth/providers
curl.exe -i https://moemail-github.pages.dev/api/auth/csrf
curl.exe -i https://moemail-github.pages.dev/api/auth/session
```

检查 D1 邮箱和消息：

```powershell
wrangler d1 execute moemail --remote --command "SELECT address FROM email ORDER BY created_at DESC LIMIT 10;"
wrangler d1 execute moemail --remote --command "SELECT count(*) AS messages FROM message;"
```

监听收信 Worker：

```powershell
wrangler tail email-receiver-worker --config wrangler.email.json --format pretty --sampling-rate 0.99
```

## 当前验证结果

已验证：

- 用户注册正常
- 用户登录正常
- Session 正常
- D1/KV 绑定正常
- `nodejs_compat` 生效
- `openwang.eu.cc` Catch-all 到 `email-receiver-worker` 正常
- Windsurf 验证码邮件可入库

## 注意事项

- 不要使用 `@moemail.app` 收信；该域名不在当前 Cloudflare 账号中。
- 目前应使用 `@openwang.eu.cc`。
- 不要提交真实 `AUTH_SECRET`、OAuth Secret、Cloudflare API Token。
- Windows 本地 `next-on-pages` 曾出现 API 路由错乱；生产部署应优先使用 GitHub + Cloudflare Pages 的 Linux 构建环境。
