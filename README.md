# sing-box-rules

这是一个用于同步 `sing-box` `.srs` 规则文件的自动化仓库。

仓库当前通过 GitHub Actions 定时下载上游规则文件，并将 `config/` 目录中的本地 JSON 规则集编译为 `.srs` 文件，再统一推送到指定的 Gitea 仓库目录 `rules/`，适合用于维护路由规则、广告拦截规则等二进制规则集。

## 功能概览

- 定时同步多个上游 `.srs` 规则文件
- 支持将 `config/*.json` 使用 `sing-box` 编译为 `.srs`
- 支持 GitHub Actions 手动触发
- 单个本地 JSON 编译失败时只输出 warning，不中断整体同步
- 仅在检测到文件变更时提交并推送
- 自动推送到指定的 Gitea 仓库分支

## 当前同步的规则文件

工作流会下载以下文件：

| 本地文件名 | 来源 |
| --- | --- |
| `adblock_reject.srs` | `https://raw.githubusercontent.com/REIJI007/AdBlock_Rule_For_Sing-box/main/adblock_reject.srs` |
| `geosite-category-ads-all.srs` | `https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-category-ads-all.srs` |
| `adblocksingbox.srs` | `https://raw.githubusercontent.com/217heidai/adblockfilters/main/rules/adblocksingbox.srs` |

同时，工作流会扫描仓库中的 `config/*.json` 文件，并将每个文件编译为同名 `.srs` 文件，例如：

| 本地 JSON | 编译产物 |
| --- | --- |
| `config/custom-rule.json` | `rules/custom-rule.srs` |

所有下载得到和本地编译得到的 `.srs` 文件都会先放到 GitHub Actions 运行环境中的 `rules/` 目录，然后复制到目标 Gitea 仓库的：

```text
rules/
```

## 工作流说明

工作流文件位于：

`./.github/workflows/rule-sync.yml`

### 触发方式

- 定时执行：每周四北京时间 20:00
- 手动执行：通过 GitHub Actions 的 `workflow_dispatch`

说明：GitHub Actions 的 `cron` 使用 UTC 时区，当前配置 `0 12 * * 4` 对应北京时间每周四 20:00。

### 执行流程

1. 创建临时目录 `rules/`
2. 使用 `wget` 下载上游 `.srs` 文件
3. 下载并安装固定版本的 `sing-box`
4. 将 `config/*.json` 编译为对应的 `.srs` 文件
5. 如果某个本地 JSON 编译失败，则输出 warning 和具体错误日志，并继续处理剩余文件
6. 使用 `GITEA_URL`、`GITEA_REPO` 和 `GITEA_TOKEN` 克隆目标 Gitea 仓库
7. 将所有 `.srs` 文件复制到 `gitea-repo/rules`
8. 若检测到变更，则自动提交并推送到目标分支

## GitHub Secrets 配置

在 GitHub 仓库的 `Settings > Secrets and variables > Actions` 中配置以下 Secrets：

| Secret 名称 | 说明 |
| --- | --- |
| `GITEA_URL` | Gitea 服务地址，例如 `https://gitea.example.com` |
| `GITEA_REPO` | 目标仓库路径，例如 `username/sing-box-rules` |
| `GITEA_TOKEN` | 具备目标仓库推送权限的 Gitea Personal Access Token |
| `GITEA_BRANCH` | 目标分支名，可选；未设置时默认使用 `main` |

## 使用方式

### 1. Fork 或创建本仓库

将当前仓库推送到 GitHub，并确保 Actions 已启用。

### 2. 配置 Secrets

按上文要求设置 Gitea 相关凭据。

### 3. 手动触发验证

进入 GitHub 仓库的 `Actions` 页面，运行 `Sync SRS Rule Files to Gitea` 工作流，确认：

- 上游规则文件可以正常下载
- 本地 `config/*.json` 可以正常编译为 `.srs`
- Gitea 仓库可以正常克隆或初始化
- 编译失败时会显示 warning 和具体错误日志
- 规则文件被复制到目标目录 `rules/`
- 有变更时可以成功提交并推送

## 仓库定位

这个仓库本身更像是一个“同步入口”而不是规则文件存储仓库：

- GitHub 仓库负责调度和执行同步任务
- Gitea 仓库负责保存最终同步产物
- 目标规则目录固定为 `rules/`

## 注意事项

- 当前同步逻辑依赖第三方上游仓库可访问性
- 若上游文件路径变更，工作流会下载失败，需要同步更新 `rule-sync.yml`
- 若本地 JSON 规则格式错误，工作流会输出 warning 和具体错误日志，但不会阻塞其他 `.srs` 文件同步
- 若目标 Gitea 仓库为空，工作流会自动初始化仓库并创建目标分支
- 只有检测到内容变更时才会创建新的提交

## 后续可扩展方向

- 增加更多 `.srs` 或其他规则文件来源
- 在同步前增加校验步骤，例如文件存在性和哈希校验
- 增加失败通知，例如 Telegram、邮件或 Webhook
- 支持同步到多个目标仓库或多个目录
