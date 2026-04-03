# Rule Sync 本地 JSON 编译设计

## 概要

扩展 `.github/workflows/rule-sync.yml`，使工作流在运行时将 `config/*.json` 中的每个文件通过 `sing-box` 编译为同名 `.srs` 文件，并与下载得到的上游 `.srs` 文件一起暂存，最终仅将 `.srs` 产物推送到目标 Gitea 仓库的 `rules/` 目录。

单个 JSON 文件的编译失败不应导致整个工作流失败。工作流应对失败文件输出 warning，继续编译剩余文件，并仍然推送所有成功生成的 `.srs` 文件以及下载成功的上游 `.srs` 文件。

## 当前状态

- 当前工作流会将 3 个上游 `.srs` 文件下载到临时目录 `rules/`。
- 当前工作流会克隆或创建目标 Gitea 仓库，并将 `rules/*.srs` 复制到 `gitea-repo/rules/`。
- 当前仓库在 `config/` 目录下维护本地源码规则集，目前已有 `config/ai-proxy-list.json`。
- `README.md` 的部分描述与实际工作流不完全一致，尤其是 cron 说明和目标路径说明。

## 目标

- 在工作流运行时，将每个 `config/*.json` 编译为 `rules/<basename>.srs`。
- 保持最终推送契约不变，仍然只向 `gitea-repo/rules/` 推送 `.srs` 文件。
- 当某个 JSON 编译失败时，继续处理其他 JSON 文件。
- 在工作流日志中清晰暴露编译失败，并以 warning 形式提示。
- 更新文档，使维护者能明确理解“上游下载 + 本地 JSON 编译”的整体同步行为。

## 非目标

- 不将源 JSON 文件推送到 Gitea。
- 不引入单独的构建脚本或本地测试框架。
- 不修改现有 Gitea 仓库初始化与创建逻辑。
- 不修改目标同步目录 `rules/`。

## 拟议的工作流改动

### 1. 安装 `sing-box`

在下载上游规则文件之后新增一个独立步骤，在 GitHub Actions runner 中安装固定版本的 `sing-box` Linux 二进制。

设计约束：

- 使用固定版本而不是 `latest`，保证运行结果可复现。
- 从官方 GitHub Releases 的 Linux AMD64 产物下载。
- 将可执行文件放到后续步骤可直接调用的路径中。

### 2. 编译本地 JSON 规则集

在 `sing-box` 安装完成后新增一个步骤，用于：

- 检查 `config/` 目录是否存在。
- 遍历 `config/*.json`。
- 对每个 JSON 文件计算输出目标 `rules/<basename>.srs`。
- 执行 `sing-box rule-set compile --output "<target>" "<source>"`。

行为要求：

- 如果没有任何 JSON 文件，输出简短日志并成功继续。
- 如果编译成功，则保留生成的 `.srs` 文件到 `rules/`。
- 如果单个文件编译失败，不立即退出当前步骤。
- 对失败文件输出 GitHub Actions warning，并继续处理下一个文件。
- 步骤结束时，如果存在失败项，再额外输出一次汇总 warning。

### 3. 保持现有推送模型

保持 Gitea 同步步骤的整体结构不变：

- 继续克隆或创建目标仓库。
- 继续将 `rules/*.srs` 复制到 `gitea-repo/rules/`。
- 继续仅在目标仓库内容发生变化时才提交并推送。

这样可以保持目标仓库目录结构兼容，也避免将源文件混入产物仓库。

## 失败处理策略

### 下载失败

对于上游 `.srs` 文件下载，保持当前失败语义不变。如果某个上游下载命令失败，整个工作流仍按现有行为直接失败。

### 本地编译失败

本地编译失败属于软失败：

- 编译失败的 JSON 文件不会产出或更新对应 `.srs` 文件。
- 工作流会为该文件输出 warning。
- 其他 JSON 文件继续编译。
- 工作流仍继续进入 push 步骤。

这样可以保证单个本地规则源损坏时，不会阻塞其他有效本地规则或上游下载规则的交付。

## 日志要求

工作流日志需要清楚展示以下状态：

- 当前安装的 `sing-box` 版本。
- 当前正在编译哪个 JSON 文件。
- 哪些文件编译成功以及生成的目标文件名。
- 哪些文件编译失败并被跳过。
- 编译失败的文件需要在日志中显示具体错误信息。
- 是否未发现本地 JSON 文件。

对于编译失败，应使用 GitHub Actions warning annotation，以便在运行结果摘要中清晰可见。

## 文档更新

更新 `README.md` 以反映真实工作流行为：

- 说明工作流现在同时处理“下载的 `.srs` 文件”和“由 `config/*.json` 编译得到的本地规则集”。
- 明确只有 `.srs` 产物会被推送到 Gitea。
- 说明本地 JSON 编译失败只会产生 warning，不会中断整体同步。
- 将目标路径说明修正为 `rules/`。
- 将 cron 说明修正为与 `.github/workflows/rule-sync.yml` 中的实际表达式一致。

## 验证计划

由于当前仓库没有自动化测试套件，本次验证以工作流行为为主：

1. 检查渲染后的 workflow YAML，确认没有语法回归。
2. 检查 shell 循环，确认它能正确处理 0 个、1 个和多个 JSON 文件。
3. 确认编译命令会将 `config/foo.json` 映射为 `rules/foo.srs`。
4. 确认单个编译失败只产生 warning，不会中断整个 job。
5. 确认 push 步骤仍然只复制 `rules/*.srs`。
6. 合并后手动触发 GitHub Actions 工作流，验证真实 runner 中的 `sing-box` 下载路径和执行行为。

## 实现说明

- 在 shell 中保持变量展开使用 `"${VAR}"` 形式。
- 在 workflow 步骤里优先使用简单、可维护的 shell 写法。
- 避免在日志中打印带认证信息的远程地址或 token。
- 将改动范围控制在 `.github/workflows/rule-sync.yml` 及对应的 README 更新之内。
