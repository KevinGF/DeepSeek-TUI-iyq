# Docker

> **English**：[DOCKER.md](DOCKER.md)

DeepSeek TUI 在 [GitHub Container Registry](https://github.com/Hmbown/DeepSeek-TUI/pkgs/container/deepseek-tui) 提供官方多架构镜像（amd64 + arm64）。

## 快速开始

```bash
docker run --rm -it \
  -e DEEPSEEK_API_KEY="$DEEPSEEK_API_KEY" \
  -v ~/.deepseek:/home/deepseek/.deepseek \
  ghcr.io/hmbown/deepseek-tui:latest
```

镜像**仅**发布到 GitHub Container Registry（GHCR）。当前未配置 Docker Hub——若需要可向发布工作流添加带 Hub 凭据的 `docker/login-action`。

## 环境变量

| 变量 | 必需 | 说明 |
|-----------------------|----------|--------------------------------------------------|
| `DEEPSEEK_API_KEY`    | 是 | DeepSeek API 密钥 |
| `DEEPSEEK_BASE_URL`   | 否 | 自定义 API 根 URL（如 `https://api.deepseek.com`） |
| `DEEPSEEK_NO_COLOR`   | 否 | 设为 `1` 关闭终端彩色输出 |

## 卷

将 `~/.deepseek` 挂载以在容器重启间持久化会话、配置、技能、记忆与离线队列：

```bash
-v ~/.deepseek:/home/deepseek/.deepseek
```

无此挂载则每次启动都是全新环境。

## 非交互 / 管道用法

stdin 非 TTY 时，`deepseek` 进入调度器一次性模式（`deepseek -c "…"`）。可向 stdin 管道输入提示：

```bash
echo "Explain the Cargo.toml in structured English." | \
  docker run --rm -i -e DEEPSEEK_API_KEY ghcr.io/hmbown/deepseek-tui:latest
```

## 本地构建

```bash
# 单平台（本机架构）
docker build -t deepseek-tui .

# 多平台（需支持模拟的 builder）
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t deepseek-tui .
```

## Devcontainer

仓库含 [`.devcontainer/devcontainer.json`](../.devcontainer/devcontainer.json)，适用于 VS Code / GitHub Codespaces。预装 Rust 工具链、rust-analyzer 与 `deepseek` 二进制。在 devcontainer 中打开仓库即可获得可用开发环境。

## 标签

| Tag | 含义 |
|------------|--------------------------|
| `latest` | 最新稳定版 |
| `v0` | 最新 v0.x |
| `0.8.9` | 指定发行版本 |

推送 release 标签时会自动构建并推送镜像（见 [release.yml](../.github/workflows/release.yml)）。
