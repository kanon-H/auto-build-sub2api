
# auto-build-sub2api 自动构建项目搭建指南

## 1. 项目概述
本项目通过 Fork 上游仓库 + GitHub Actions 实现 `sub2api` Docker 镜像的自动构建与推送。当上游仓库更新时，自动同步并触发构建，无需手动干预。

## 2. 前置准备（人工完成）

在开始之前，请确认以下信息并替换文档中的占位符：

| 占位符 | 说明 | 示例值 |
| :--- | :--- | :--- |
| `{UPSTREAM_REPO}` | 上游原始仓库地址 | `https://github.com/原作者/sub2api.git` |
| `{UPSTREAM_BRANCH}` | 上游主分支名称 | `main` 或 `master` |
| `{DOCKERHUB_USERNAME}` | 你的 Docker Hub 用户名 | `myuser` |
| `{DOCKERHUB_IMAGE}` | 镜像名称 | `auto-build-sub2api` |
| `{GITHUB_REPO}` | 你 Fork 后的仓库地址 | `https://github.com/你的用户名/auto-build-sub2api.git` |

### 必须完成的准备工作
1.  在 GitHub 上 Fork 上游仓库 `{UPSTREAM_REPO}`，并重命名为 `auto-build-sub2api`
2.  在 Docker Hub 创建同名镜像仓库 `{DOCKERHUB_IMAGE}`
3.  在 Fork 后的 GitHub 仓库 → Settings → Secrets and variables → Actions 中添加：
    -   `DOCKERHUB_USERNAME`：你的 Docker Hub 用户名
    -   `DOCKERHUB_TOKEN`：Docker Hub Access Token（非密码，在 Docker Hub Account Settings → Security 中生成）
4.  确认上游仓库的 License 允许第三方构建与分发

## 3. 需要生成的文件清单

核心自动化由以下两个 workflow 文件实现。第三方 Action 使用所列主版本对应的完整提交 SHA 固定；另提供 `.github/dependabot.yml` 每周检查 Action 更新。

### 3.1 `.github/workflows/sync-upstream.yml`
**功能**：定时检查上游更新，自动合并到本仓库默认分支（当前为 `master`）
**触发条件**：每 6 小时自动执行 + 支持手动触发
**关键要求**：
-   使用 `actions/checkout@v4`，`fetch-depth: 0`
-   添加 upstream remote 并 fetch
-   优先使用 `git merge --ff-only` 快进合并；本地 CI/文档提交导致分叉时，回退到 `git merge --no-edit`，保留双方历史
-   仅在有新提交时 push，避免无效触发
-   合并失败时标记为 workflow failed，不强制覆盖

### 3.2 `.github/workflows/docker-build.yml`
**功能**：作为可复用工作流，由同步工作流显式调用，构建并推送多架构 Docker 镜像
**触发条件**：`sync-upstream` 成功 push 后调用；首次发布或失败重试可手动勾选 `force_build`

> 默认 `GITHUB_TOKEN` 的 push 不会触发另一个 push 工作流，因此使用 `workflow_call` 串联，无需额外 PAT。普通 push 不发布镜像，上游无更新时默认跳过构建。

**关键要求**：
-   使用 `docker/setup-qemu-action@v3` + `docker/setup-buildx-action@v3` 支持多架构
-   构建平台：`linux/amd64,linux/arm64`
-   镜像标签策略：
    -   `latest`
    -   `{YYYYMMDD}-{short-sha}` 格式的版本标签
-   使用 BuildKit 缓存加速构建
-   构建完成后输出镜像 digest 信息

## 4. README.md 合规声明模板

请在 README 顶部添加以下内容：

```markdown
> ⚠️ **Unofficial Automated Build**
> This is an unofficial automated Docker image build of [sub2api]({UPSTREAM_REPO}).
> All credits go to the original authors. This repository only provides CI/CD automation.
> Upstream source: {UPSTREAM_REPO}
```

## 5. 验证清单

代码生成后，请按以下步骤验证：

- [ ] 两个 workflow 文件语法正确，无 YAML 缩进错误
- [ ] `sync-upstream.yml` 中 upstream URL 和分支名正确
- [ ] `docker-build.yml` 中镜像名称与 Docker Hub 一致
- [ ] 所有敏感信息均通过 `${{ secrets.* }}` 引用，无硬编码
- [ ] 手动触发一次 `sync-upstream`，确认能正常拉取上游代码
- [ ] 确认 `docker-build` 在上游无更新时不会重复构建
- [ ] 构建成功后在 Docker Hub 验证镜像可正常 pull

## 6. 当前实现的启用与维护

### 6.1 启用步骤

1. 将 `.github/workflows/sync-upstream.yml`、`.github/workflows/docker-build.yml`、`.github/dependabot.yml`、`README.md` 和本指南提交到默认分支（当前 `master`）。
2. 在 Docker Hub 创建 `sub2api` 仓库；在 GitHub 仓库 **Settings → Secrets and variables → Actions → Secrets** 配置：
   - `DOCKERHUB_USERNAME`：填写你的实际 Docker Hub 用户名；工作流只通过 `${{ secrets.DOCKERHUB_USERNAME }}` 引用。
   - `DOCKERHUB_TOKEN`：具有该镜像仓库写入权限的 Docker Hub Access Token；工作流只通过 `${{ secrets.DOCKERHUB_TOKEN }}` 引用。
3. 在 **Actions** 页面启用 Fork 的工作流。确认组织/仓库策略允许作业申请 `contents: write`；同步 job 已显式申请，构建 job 仅使用 `contents: read`。
4. 打开 **Actions → Sync upstream → Run workflow**，选择默认分支，首次运行勾选 `force_build`。即使上游没有新提交，也会发布首个镜像。
5. 成功后查看运行摘要，获取镜像版本标签、源码提交和 digest，然后验证：

   ```bash
   # 将 YOUR_DOCKERHUB_USERNAME 替换为 Secrets 中配置的用户名
   docker buildx imagetools inspect docker.io/YOUR_DOCKERHUB_USERNAME/sub2api:latest
   docker pull docker.io/YOUR_DOCKERHUB_USERNAME/sub2api:latest
   ```

   Manifest 应包含 `linux/amd64` 和 `linux/arm64`；SBOM/来源证明还可能显示额外的 attestation 条目。

### 6.2 已实现的优化

- 每 6 小时检查（UTC `00:23 / 06:23 / 12:23 / 18:23`），错峰执行；调度可能被 GitHub 延迟，不保证准点。
- 无上游变更时不 push、不构建；仅 `force_build` 可显式例外。
- 同步使用 Git 合并，不使用 `reset --hard`、目录覆盖或强制推送。上游没有的本地 Action 文件会自然保留；推送前额外检查 `.github/workflows/`、`.github/dependabot.yml` 和 `AUTO-BUILD.md` 与同步前完全一致，任何新增、修改、删除都中止本次同步，等待人工审核。
- 同步及构建串行运行，避免较旧镜像覆盖较新 `latest`；运行中任务不自动取消。
- 构建 checkout 同步后准确 SHA，而非事件开始时的旧 SHA；版本标签使用 UTC 日期及 12 位短 SHA。
- BuildKit 使用 GitHub Actions 缓存；保留上游 Dockerfile 原生交叉编译优化，无需修改业务代码。
- 为 GitHub 托管 runner 使用官方 Go 模块代理与校验服务，传递完整提交与构建时间，并发布 SBOM 和最小来源证明。
- Docker 用户名和令牌均来自 Secrets，不写入镜像构建参数、标签或仓库文件。
- README 非官方构建声明和镜像源码/revision 标签方便追溯。
- 第三方 Action 固定到完整提交 SHA，由 Dependabot 每周提交更新 PR；checkout 不持久化 Git 凭证，同步仅在 push 时临时提供令牌。

### 6.3 故障恢复与限制

- **合并冲突**：本次同步失败且不 push；手动拉取上游解决冲突后推送，再勾选 `force_build` 发布。禁止用强制推送掩盖冲突。
- **构建/推送失败**：可重跑失败作业，或重新运行同步并勾选 `force_build`。上游已同步后，下一次普通定时执行不会自动重建同一提交。
- **分支保护禁止机器人直推**：需调整分支规则，或改用 PR 同步方案；当前实现不会绕过保护规则。
- **上游修改自动化配置**：若合并结果改变 `.github/workflows/`、`.github/dependabot.yml` 或 `AUTO-BUILD.md`，保护检查会主动失败，不会推送。需人工审查，保留本仓库所需 Actions 后合并；不要为通过检查而移除保护。上游其他工作流变更还可能受 `GITHUB_TOKEN` 的 workflows 推送权限限制，不要盲目放宽令牌权限。
- **上游属于信任边界**：同步后的 Dockerfile/源码会进入构建环境。只同步可信上游；需要更严格审核时应改为 PR 审批后发布。
- **基础镜像和工具链**：沿用上游 Dockerfile；上游指定的镜像标签不存在、依赖不可用或源码编译失败时，需要修复后重试，本地静态检查不能保证实际构建成功。
- **缓存**：GitHub 缓存有容量和淘汰限制，且 `type=gha` 不保证持久化 Dockerfile 内全部 `RUN --mount=type=cache` 内容，缓存失效不会影响正确性。
- **版本标签**：日期+SHA 便于追溯，但同日同提交手动重建会复用标签；部署需要严格不可变引用时，应使用摘要中的 `image@sha256:...`。
- **许可证**：仓库包含 LGPL-3.0 文本；非官方声明不能替代许可证义务。分发前应核对适用条款，保留许可证、版权和第三方声明，并提供与镜像 revision 对应的源码及构建资料。
- **长期维护**：GitHub 可能暂停公开仓库长期无活动的定时任务，应定期检查 Actions；Dependabot 更新 PR 仍需审核、验证后合并。

## 7. 注意事项

-   首次同步可能需要较长时间，取决于上游仓库大小
-   如果上游有破坏性变更导致构建失败，需手动介入排查
-   Docker Hub 免费账户有并发构建限制，多架构构建可能排队
-   建议设置上游同步频率不低于每 4 小时，避免触发 GitHub API 限流
-   定期更新 workflow 中使用的 Action 版本以保持安全性


上游原始仓库地址：https://github.com/asupc/sub2api
上游主分支名称：master
我的dockerhub用户名：kanonhua
镜像名称：sub2api
fork后的github地址：https://github.com/kanon-H/auto-build-sub2api.git

我任务dockerhub名称和dockerhub密钥，需要在action中使用密钥占位符来实现，不要直接写到代码中
