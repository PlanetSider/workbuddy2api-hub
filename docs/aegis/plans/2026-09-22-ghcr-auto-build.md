# GHCR 自动构建与发布实施计划

## Aegis Visibility

本次变更新增 GitHub Actions 发布边界和 GHCR 权限；计划用于固定触发条件、标签语义、凭据范围与验证证据，避免镜像被错误发布或 Compose 继续指向旧镜像。

## Goal

将仓库推送到 GitHub 后自动构建 Docker 镜像并发布到 GitHub Container Registry：

- Pull Request 只构建验证，不推送镜像；
- `main` 分支推送发布 `latest` 与提交 SHA 标签；
- `v*` Git 标签推送发布版本标签，并由 metadata action 生成稳定的版本标签；
- 使用仓库内置 `GITHUB_TOKEN`，不新增长期 Docker 凭据。

## Architecture

- 现有 `Dockerfile` 是唯一镜像构建入口，保持不变。
- 新增 `.github/workflows/docker-image.yml` 作为 CI/CD owner：checkout、Buildx、GHCR 登录、metadata、build/push、镜像 provenance attestation。
- 镜像规范名为 `ghcr.io/${{ github.repository }}`，由 GitHub Actions 在运行时展开为 `ghcr.io/planetsider/workbuddy2api-hub`。
- 现有 `docker-compose.yml` 作为生产部署入口，只使用 GHCR 镜像并通过 `pull_policy: always` 拉取最新发布产物，不承担本地构建。

## Tech Stack

- GitHub Actions
- Docker Buildx / `docker/build-push-action`
- `docker/metadata-action`
- GitHub Container Registry
- 现有 Python 3.11 Alpine Dockerfile

## Baseline / Authority Refs

- `Dockerfile:2-24`：当前镜像基础、复制内容、端口和启动命令。
- `docker-compose.yml:1-17`：生产拉取镜像、更新策略、端口和镜像名。
- `README.md:96-125`：Docker 生产部署方式及持久化边界。
- GitHub 官方发布 Docker 镜像指南：<https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images>
- Docker metadata action 标签规则：<https://github.com/docker/metadata-action>

## Compatibility Boundary

- 容器内应用行为、端口 `8788`、挂载目录 `/app/accounts` 和 `/app/usage` 不变。
- `docker compose pull && docker compose up -d --no-build` 只从 GHCR 拉取并运行，不在部署机本地构建。
- 本次不保留本地开发 Compose 文件；本地源码构建不属于默认部署路径。
- 仅 CI 的 `push` 条件会改变远程发布行为；PR 不会写入 GHCR。
- GHCR 首次发布后的包可见性仍由仓库/组织设置决定，工作流不自动修改包设置。

## Requirement Ready Check

- Requirement source: 用户确认发布到 GHCR。
- Scope: 用户确认推荐触发策略（PR 验证；`main` 推送和 `v*` 标签发布）。
- Scenario: PR 校验、主分支持续发布、版本标签发布。
- Acceptance: workflow 可被 GitHub 解析；PR 的 `push` 为 false；主分支和版本标签生成预期 tags；使用 `GITHUB_TOKEN` 的 packages write 权限；生产 Compose 不含 `build`，只从本仓库 GHCR 镜像拉取。
- Decision: ready。

## Change Necessity

本次需要新增 GitHub Actions 工作流来完成远程构建发布，并修改生产 Compose 以移除本地构建入口；仅修改文档无法实现这两个部署行为。运行时 Python 和 Dockerfile 不需要改动。

## Ripple Signal Triage

- Signal: distribution/release surface。
- Canonical owner: `.github/workflows/docker-image.yml`。
- Downstream consumer: `docker-compose.yml`、README 中的 Docker 部署用户。
- Source of truth: workflow 的 `images` 与 metadata tag 规则；Compose 只负责拉取和运行发布镜像。
- Retirement: 移除 Compose 的本地 `build` owner，不保留第二套本地开发 Compose；不保留旧第三方镜像名作为默认值，避免发布产物与文档发生分叉。

## TDD Route

- Mode: off
- Decision: skipped
- Authority: 用户未要求 TDD；变更是 CI 配置与镜像引用，不新增运行时行为。
- Test posture: 采用静态 YAML 检查、现有无网络测试、Docker 构建验证；CI 本身在 GitHub runner 上进行最终验证。

## Implementation Tasks

1. 新增 `.github/workflows/docker-image.yml`。
   - 触发：`pull_request` 到 `main`、`push` 到 `main`、`push` 的 `v*` 标签、手动 `workflow_dispatch`。
   - 权限：`contents: read`、`packages: write`、`attestations: write`、`id-token: write`。
   - PR：构建但 `push: false`，不登录 GHCR。
   - 非 PR：使用 `${{ github.actor }}` 与 `${{ secrets.GITHUB_TOKEN }}` 登录 `ghcr.io`。
   - metadata：分支/标签/SHA 标签；`main` 生成 `latest`，版本标签生成 semver 版本标签，所有发布保留 `sha-<short>`。
   - build-push：使用现有 Dockerfile；成功后生成 GHCR provenance attestation。
   - 对 GitHub Actions 依赖采用固定 major 版本（至少 checkout、setup-buildx、login、metadata、build-push、attest），便于后续升级审查。

2. 修改 `docker-compose.yml` 的生产部署契约。
   - 删除 `build: .`，只保留 `ghcr.io/planetsider/workbuddy2api-hub:latest`。
   - 增加 `pull_policy: always`，确保 `latest` 在部署时从 GHCR 更新。
   - 保留端口、环境变量和持久化挂载；不新增本地开发 Compose 文件。

3. 更新 README Docker 段落。
   - 以 `docker compose pull` 和 `docker compose up -d --no-build` 为默认部署流程。
   - 保留 GHCR 直接运行方式和标签说明。
   - 提示私有 GHCR 包需要先登录。
   - 不修改业务配置、安全提示和运行参数。

## Verification

- 解析 `.github/workflows/docker-image.yml` 的 YAML 结构；确认 workflow 事件、权限、条件、镜像名和 tags 配置存在且无重复冲突。
- 静态检查生产 Compose 不含 `build`、含 GHCR `image` 和 `pull_policy: always`，且仓库只有默认生产 Compose 文件。
- 在本机执行仓库已有 Python 测试脚本和 `node _test_matrix_filters.js`，确认 CI 配置未触碰运行时逻辑。
- 若本机 Docker 可用，执行 `docker compose config` 和远程镜像拉取；当前环境未安装 Docker，因此将该项记录为环境限制，并依赖部署机/GitHub runner 验证。
- 检查 `git diff --check`、工作区状态和最终 diff。

## Risks / Open Items

- GitHub 仓库设置若禁止 GitHub Actions 写入 packages，发布 job 会失败；需在仓库 Settings → Actions → General 将 workflow 权限允许为读写，或至少允许 `GITHUB_TOKEN` 写入 packages。
- GHCR 包首次创建后的公开/私有状态不由 workflow 自动决定。
- 当前仓库 README 的历史链接仍有旧仓库归属信息，本次只更新 Docker 镜像发布相关内容，不扩展到项目品牌/链接重构。

## Plan Pressure Test

- Owner fit: workflow 负责 CI/CD，Dockerfile 继续负责镜像内容，生产 Compose 只负责拉取和运行，边界清晰。
- Contract fit: 镜像名和 tags 是显式发布契约，README 与 Compose 对齐。
- Verification fit: 静态检查 + 现有测试 + GitHub runner Docker 构建覆盖配置和构建风险。
- Task executability: 单一工作流、单一 Compose 引用和文档同步，任务可直接执行。
- Result: within scope。

## Execution Route

采用 inline 执行：变更集中在 CI 配置、Compose 和 README，任务之间存在明确顺序且协调收益不足。

## User confirmation required

No。GHCR 与触发策略已经由用户明确确认；仓库 Settings 权限属于首次发布前的操作提示，不需要在本地修改。
