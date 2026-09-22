# Project Agent Guide

## 项目定位

本项目是一个用 Go 编写的、自托管的 OpenAI 兼容网关，将 CodeBuddy 账号接入统一的 HTTP API，并提供内嵌 Web 管理面板。服务默认监听 `7863`，面板路径为 `/panel/`。

模块路径为 `github.com/linguo2625469/workbuddy2api-panel`。面板前端资源位于 `internal/panel`，通过 Go embed 进入服务二进制。

## 目录与职责

- `cmd/server`：主服务入口、配置加载和依赖装配。
- `cmd/login`、`cmd/signin`、`cmd/credit`：账号登录、签到和积分工具。
- `cmd/trial`：试用/探测相关命令入口。
- `internal/server`：HTTP 路由、鉴权、请求处理和请求日志。
- `internal/upstream`：上游 CodeBuddy 请求、模型目录、SSE、任务和账号行为接口。
- `internal/pool`：账号池、冷却、熔断、会话选择和状态持久化。
- `internal/panel`：Web 管理面板及其 API。
- `internal/scheduler`：签到、活跃、旅行、保活和任务调度。
- `internal/auth`、`internal/httpauth`、`internal/session`：凭证、HTTP 鉴权和会话相关逻辑。
- `scripts`：运维、探测和任务辅助脚本。
- `Dockerfile`：多阶段、无 CGO 的镜像构建；运行时默认使用 UID `10001` 的 `app` 用户。
- `docker-compose.yml`：生产部署入口，默认使用 GHCR 镜像。
- `.github/workflows/docker-image.yml`：GitHub Actions 镜像构建与发布流程。

## 本地开发与验证

项目需要 Go `1.22.5` 或更高版本。提交 Go 代码前，按改动范围执行：

```bash
go test ./...
go vet ./...
go build ./...
git diff --check
```

涉及镜像或 Compose 的改动，在已安装 Docker 的环境中额外执行：

```bash
docker compose config
docker build -t workbuddy2api-local .
```

如果需要运行本地构建镜像：

```bash
WB2API_IMAGE=workbuddy2api-local WB2API_PULL_POLICY=missing docker compose up -d
```

不要把本地生成的二进制、`config.json`、账号凭证或运行状态提交到仓库。

## Docker 与发布约束

GitHub Actions 在以下事件构建镜像：

- Push：构建并发布到 `ghcr.io/planetsider/workbuddy2api-panel-docker`。
- Pull Request：只构建验证，不发布镜像。
- 默认分支的镜像标签包含 `latest`；提交 SHA 标签使用 `sha-<commit>` 形式。
- 镜像目标架构为 `linux/amd64` 和 `linux/arm64`。

Compose 默认配置为：

```text
ghcr.io/planetsider/workbuddy2api-panel-docker:latest
```

部署前需要准备宿主机文件和目录：

```bash
cp config.example.json config.json
mkdir -p auths data
```

Compose 会持久化挂载 `auths/`、`data/` 和 `config.json`。面板配置页需要写入 `config.json`，不要把该挂载设置为只读。宿主机目录权限不匹配时，优先使用 `PUID`/`PGID` 显式指定容器用户；不要为了绕过权限问题默认改成 root。

使用固定版本时通过 `WB2API_IMAGE` 覆盖镜像，例如：

```bash
WB2API_IMAGE=ghcr.io/planetsider/workbuddy2api-panel-docker:sha-<commit> docker compose up -d
```

## 敏感数据边界

以下内容只能留在本地或受控的部署环境中，不得写入源码、日志、Issue 或提交：

- `auths/` 中的 access token、refresh token 和账号信息。
- `config.json` 中的 `api_key`、Upstash token 及其他私密配置。
- `.env`、私钥、证书和运行状态文件。

修改鉴权、凭证落盘、文件路径拼接或上游请求头时，必须同时考虑泄露、路径穿越、权限和日志脱敏风险。

## 修改规则

1. 先阅读相关模块和测试，沿用现有的配置、错误处理和日志模式。
2. 保持 `/v1/chat/completions`、`/v1/models`、`/status`、`/healthz` 及面板 API 的既有兼容性；配置字段变更要同步更新 `config.example.json` 和 `README.md`。
3. 行为变更优先补充或更新对应的 Go 测试；Docker/CI 变更至少检查 YAML、镜像引用、标签和卷挂载。
4. 不要在 Compose 中重新引入默认本地 `build`，除非部署需求明确改变；本项目当前的发布源是 GitHub Actions + GHCR。
5. 不要修改或删除用户已有的未提交改动，不要使用强制推送、硬重置或无范围清理。
6. 变更部署流程、环境变量、端口、数据目录或发布标签时，必须同步更新本文件和 `README.md`。

## 提交前检查

提交前确认：

- `git status` 中没有意外的凭证、配置或构建产物。
- `git diff --check` 通过。
- 相关测试已运行；无法运行时明确说明缺失的工具或环境。
- Compose 仍保留 `auths`、`data`、`config.json` 三类持久化挂载。
- CI 的 Pull Request 构建不会写入 GHCR，Push 构建才会发布镜像。
