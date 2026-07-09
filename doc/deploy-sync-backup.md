# 部署、上游同步与数据备份指南

本仓库配置了三个 GitHub Actions 工作流：

| 工作流 | 文件 | 触发方式 | 作用 |
| ------ | ---- | -------- | ---- |
| 🚀 部署 | `.github/workflows/deploy-cloudflare.yml` | 推送到 `main`（或手动） | 构建前端 + 部署 Worker `cloudmail` |
| 💾 备份 | `.github/workflows/backup-data.yml` | 每天 UTC 17:30（或手动） | 导出 D1 / KV / R2 数据，打包上传到 R2 桶 `cloud-mail-backup` |
| 🔄 同步 | `.github/workflows/sync-upstream.yml` | 每周日（或手动） | 拉取上游仓库更新并自动开 PR |

## 必需的 GitHub Secrets

进入仓库 Settings → Secrets and variables → Actions → New repository secret，添加：

| Secret 名称 | 必需 | 说明 |
| ----------- | :--: | ---- |
| `CLOUDFLARE_API_TOKEN` | ✅ | Cloudflare API 令牌（见下方权限要求） |
| `CLOUDFLARE_ACCOUNT_ID` | ✅ | Cloudflare 账户 ID（Workers 概览页右侧可复制） |
| `DOMAIN` / `ADMIN` / `JWT_SECRET` | ❌ | 线上 Worker 已配置，留空则保留现有值 |
| `INIT_URL` | ❌ | 数据库已初始化，不需要 |
| `D1_DATABASE_ID` / `KV_NAMESPACE_ID` / `R2_BUCKET_NAME` | ❌ | 资源 ID 已内置在 `mail-worker/wrangler-action.toml`，只有更换资源时才需要 |

> 与上游文档不同：本仓库已把 D1 / KV / R2 的资源 ID 直接写进 `wrangler-action.toml`，
> 所以**只需要设置 `CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_ACCOUNT_ID` 两个 Secret** 即可部署和备份。

### API 令牌权限

在 [Cloudflare Dashboard → API Tokens](https://dash.cloudflare.com/profile/api-tokens) 创建令牌，需要以下账户级权限：

- Workers Scripts：编辑
- Workers KV Storage：编辑
- D1：编辑
- Workers R2 Storage：编辑

## 当前绑定的资源

| 资源 | 名称 / ID |
| ---- | --------- |
| Worker | `cloudmail`（与线上现有 Worker 同名，部署会原地更新，保留域名、路由、邮件路由和已配置的变量/机密） |
| D1 数据库 | `email`（`2027b1fb-7dc1-4e8a-a452-540390db482f`） |
| KV 命名空间 | `email`（`8433e5042f274023bfd4495d7a3ca3dd`） |
| R2 数据桶 | `email`（邮件附件） |
| R2 备份桶 | `cloud-mail-backup`（备份文件存放处） |

## 备份说明

每天自动运行（也可在 Actions 页面手动触发 “💾 Backup Cloudflare data”），备份内容：

- **D1 数据库**：`wrangler d1 export` 导出完整 SQL（含表结构和数据）
- **KV 命名空间**：所有 key 及其值（`kv/keys.json` + `kv/values/` + `kv/index.tsv` 对照表）
- **R2 附件**：`email` 桶内的全部对象

打包为 `cloud-mail-backup-<时间戳>.tar.gz` 后：

1. 上传到 R2 桶 `cloud-mail-backup/<时间戳>/` 下（长期保存，可自行清理旧备份）
2. 同时作为 GitHub Actions Artifact 保留 14 天（方便直接从 Actions 页面下载）

> 另外 Cloudflare D1 自带 Time Travel 功能，可将数据库恢复到过去 30 天内任意时间点，
> 无需备份文件：`npx wrangler d1 time-travel restore email --timestamp=<unix时间戳>`

## 恢复数据

1. 从 R2 桶 `cloud-mail-backup` 或 Actions Artifact 下载备份包并解压
2. 恢复 D1（建议先恢复到新库验证）：
   ```bash
   npx wrangler d1 execute <数据库名> --remote --file=d1-email.sql
   ```
3. 恢复 KV：按 `kv/index.tsv` 对照表用 `wrangler kv key put` 写回
4. 恢复 R2 附件：`npx wrangler r2 object put email/<key> --file=r2/<key> --remote`

## 上游同步

同步工作流默认跟踪 [`eoao/cloud-mail`](https://github.com/eoao/cloud-mail)。
如需改为其他上游（例如 `LaziestRen/cloud-mail`），在 Settings → Secrets and variables →
Actions → Variables 添加变量 `UPSTREAM_REPO`，值填 `所有者/仓库名`。

发现上游有新提交时，会把上游代码推送到 `upstream-sync` 分支并自动开 PR；
检查无误后合并 PR，即会自动触发部署。若 PR 显示冲突，需在 GitHub 上解决冲突后再合并。

> 注意：需要在 Settings → Actions → General 里勾选
> “Allow GitHub Actions to create and approve pull requests”，否则自动开 PR 会失败。
