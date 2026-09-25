# 项目代理说明

本文档是 `workbuddy2api-hub` 的项目级开发与运维约定。后续代理在修改代码、CI 或部署配置前应先阅读本文件，并以仓库现有实现为准。

## 项目定位

WorkBuddy2API-Hub 是一个基于 Python 标准库实现的 WorkBuddy 多账号反向代理网关，提供 OpenAI 兼容的 Chat Completions 与 Responses API，以及 Web 监控看板、账号管理、用量统计和后台调度能力。

运行时不依赖第三方 Python 包。默认监听 `8788` 端口，账号凭证和运行数据必须保存在持久化目录中。

## 主要文件

- `wb_proxy.py`：HTTP 服务入口、鉴权、Chat/Responses API 和看板接口。
- `wb_accounts.py`：账号池、凭证、上游请求和账号级代理处理。
- `wb_catalog.py`：国际版/国内版模型目录与能力声明。
- `wb_identity.py`：出站客户端身份和请求头定义。
- `wb_fingerprint.py`：账号设备指纹派生逻辑。
- `wb_scheduler.py`：后台定时保活、签到和任务调度。
- `wb_settings.py`：设置、API Key 和持久化配置处理。
- `wb_tasks.py`：国内版成长任务、签到和猫猫旅行逻辑。
- `dashboard.html`：Web 看板前端。
- `Dockerfile`：唯一的容器镜像构建入口。
- `docker-compose.yml`：唯一的生产部署 Compose 文件，只拉取 GHCR 镜像，不在部署机本地构建。
- `.github/workflows/docker-image.yml`：GitHub Actions 镜像构建、发布和 provenance attestation 工作流。

## 数据与安全边界

- 不要读取、打印、提交或上传真实账号凭证、API Key、Token 或 Cookie。
- `accounts/*.json` 是机密运行数据，`usage/*.jsonl` 和 `usage/*.json` 是运行时数据，均不得提交。
- 修改账号或设置逻辑时，保持现有持久化格式和原子写入行为；除非需求明确要求，不要迁移或删除已有数据。
- 不要把调试凭证、真实上游响应或个人环境路径写入测试、日志、文档或 Git 历史。

## Docker 与发布约定

生产镜像地址：

```text
ghcr.io/planetsider/workbuddy2api-hub
```

生产部署只能使用 GitHub Actions 已构建的镜像：

```bash
docker login ghcr.io
docker compose pull
docker compose up -d --no-build
```

不要为本地开发再添加第二个 Compose 文件，也不要把 `build: .` 加回生产 `docker-compose.yml`。源码级本地构建不是默认部署路径。

GitHub Actions 的发布行为：

- Pull Request：只构建验证，不推送镜像。
- 推送到 `main`：发布 `latest` 和 `sha-<短提交号>`。
- 推送 `v*` 标签：发布 SemVer 版本标签和提交 SHA 标签。
- 发布认证使用 GitHub 自动提供的 `GITHUB_TOKEN`，不新增 Docker Hub 凭据。
- 镜像发布失败时，先检查仓库 Actions 的 `packages: write` 权限和 GHCR 包可见性。

容器运行时契约保持稳定：端口为 `8788`，账号目录挂载到 `/app/accounts`，用量目录挂载到 `/app/usage`。除非需求明确要求，不要改变这些路径或启动参数。

## 修改原则

- 先阅读相关模块和测试，再做最小范围修改；遵循现有函数、数据结构和命名方式。
- 不要为了修复单个问题引入重复 owner、旁路 fallback 或第二套持久化格式。
- Python 运行时继续保持标准库实现；新增依赖必须有明确理由，并同步 Docker 构建和文档。
- API 行为、模型目录、鉴权、账号隔离和代理绑定属于兼容性敏感区域，修改后必须运行相关回归测试。
- CI/CD 改动只应放在 `.github/workflows/`，不要把发布逻辑塞进运行时 Python 代码。
- 文档中的镜像地址、启动命令和 CI 标签规则必须与工作流和 Compose 保持一致。

## 验证方式

测试脚本是独立可执行脚本，不使用 pytest。修改运行时逻辑后，至少运行完整回归集：

```powershell
$tests = Get-ChildItem -File -Filter '_test_*.py'
foreach ($test in $tests) { python $test.FullName }
node _test_matrix_filters.js
```

修改 CI、Compose 或文档后，至少检查：

```bash
git diff --check
git status --short
```

还应确认：

- PR 不会执行镜像推送；
- `main` 和 `v*` 标签的发布条件仍正确；
- 生产 Compose 不含 `build`，且镜像地址为本项目 GHCR 地址；
- 没有新增未被 `.gitignore` 覆盖的运行时机密文件。

本地没有 Docker 时，不要用其他命令伪造镜像构建通过；应明确记录未执行项，并依赖 GitHub Actions runner 或实际部署机验证 Docker 构建和拉取。

## Git 操作

- 默认在当前 `main` 分支工作，不要擅自创建或切换 worktree。
- 提交前检查差异和工作区状态，只提交与当前任务相关的文件。
- 禁止强制推送、硬重置或删除用户已有改动。
- 涉及远程推送、发布、删除或权限变更时，必须以用户明确授权为准。
