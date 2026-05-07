# DeepSeek TUI 发布手册

> **English**：[RELEASE_RUNBOOK.md](RELEASE_RUNBOOK.md)

本手册是发布 Rust crate、GitHub Release 资源与 `deepseek-tui` npm 封装的**权威说明**。

当前打包说明：

- `deepseek-tui` 是今日面向用户的**运行时与 TUI 包**。
- `deepseek-tui-core` 是提取/parity 工作的支持性 workspace crate，**不是**发货运行时的替代品。

## 规范发布目标

- 终端用户 crate：
  - `deepseek-tui`
  - `deepseek-tui-cli`
- 本 workspace 一并发布的支持 crate：
  - `deepseek-secrets`、`deepseek-config`、`deepseek-protocol`、`deepseek-state`、`deepseek-agent`、`deepseek-execpolicy`、`deepseek-hooks`、`deepseek-mcp`、`deepseek-tools`、`deepseek-core`、`deepseek-app-server`、`deepseek-tui-core`
- crates.io 上的 `deepseek-cli` 为**无关** crate，不属于本发布流。

## 版本协调

- Rust crate 继承 [Cargo.toml](../Cargo.toml) 的 workspace 共享版本。
- 内部 path 依赖版本应与 workspace 版本一致；workspace 版本 bump 后陈旧 pin 会成为发布阻断项。
- npm 封装版本在 [npm/deepseek-tui/package.json](../npm/deepseek-tui/package.json)。
- `deepseekBinaryVersion` 控制 npm 封装下载的 GitHub Release 二进制版本。
- 允许**仅 npm 打包**发布：提升 npm 包版本、`deepseekBinaryVersion` 仍钉在上一轮 Rust 二进制、重新跑 `npm pack` 冒烟后再 `npm publish`。

## 预检

打 tag 前在仓库根执行：

```bash
./scripts/release/check-versions.sh   # workspace、npm、lockfile 间版本漂移
cargo fmt --all -- --check
cargo check --workspace --all-targets --locked
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo test --workspace --all-features --locked
cargo publish --dry-run --locked --allow-dirty -p deepseek-tui
./scripts/release/publish-crates.sh dry-run
```

`check-versions.sh` 亦在 CI 每次 push/PR 运行（`.github/workflows/ci.yml` 的 `versions` job），在发布前而非发布当下捕获 `Cargo.toml`、各 crate manifest、`npm/deepseek-tui/package.json` 与 `Cargo.lock` 不一致。

`publish-crates.sh dry-run` 对无未发布 workspace 依赖的 crate 做完整 `cargo publish --dry-run`，并对有依赖的 workspace crate 做打包预检——避免 crates.io 尚未包含新 workspace 版本时的假阴性，同时仍在发布前校验包内容。

npm 封装验证：构建两个发货二进制并跑跨平台冒烟脚本。会打包 npm、安装到干净临时项目、经 HTTP 提供本地 release 资源，并检查调度器→TUI 路径（`deepseek doctor --help`）与直连 TUI（`deepseek-tui --help`）。

```bash
cargo build --release --locked -p deepseek-tui-cli -p deepseek-tui
node scripts/release/npm-wrapper-smoke.js
```

设 `DEEPSEEK_TUI_KEEP_SMOKE_DIR=1` 可保留临时 pack/安装目录供检查。

若也要本地跑 `npm run release:check`，先用完整资源矩阵 fixture 再生本地资源目录再起服务：

```bash
DEEPSEEK_TUI_PREPARE_ALL_ASSETS=1 node scripts/release/prepare-local-release-assets.js
cd npm/deepseek-tui
DEEPSEEK_TUI_VERSION=X.Y.Z DEEPSEEK_TUI_RELEASE_BASE_URL=http://127.0.0.1:8123/ npm run release:check
```

`DEEPSEEK_TUI_VERSION` 设为你要验证的 npm 包版本。

CI 工作流在 Linux、macOS、Windows 上跑相同的 tarball 安装 + 委托入口冒烟。

发布后验证两注册表均可见：

```bash
./scripts/release/check-published.sh X.Y.Z
```

**勿**在运行该命令确认 `deepseek-tui@X.Y.Z` 已在 npm、且每个 `deepseek-*` crate 在 crates.io 均为 `X.Y.Z` 之前将 Rust 发布标为完成。罕见的仅 npm 封装发布可带 `--allow-npm-binary-mismatch`，并在 release note 明确**未**发新 Rust 二进制。

## Rust Crate 发布

crates.io 发布为**手工**——无自动化 `crates-publish` GitHub 工作流。操作者在已 `cargo login` 的工作站运行 `scripts/release/` 辅助脚本。

1. 在 [Cargo.toml](../Cargo.toml) 更新 workspace 版本。
2. 本地运行 `./scripts/release/check-versions.sh` 与 `./scripts/release/publish-crates.sh dry-run`，均需通过。
3. 打 `vX.Y.Z` tag（通常将版本 bump 推 `main` 后由 `auto-tag.yml` 创建 tag——npm 小节有 `RELEASE_TAG_PAT` 要求）。
4. 按顺序用 `./scripts/release/publish-crates.sh publish` 发布：`deepseek-secrets` → `deepseek-config` → `deepseek-protocol` → `deepseek-state` → `deepseek-agent` → `deepseek-execpolicy` → `deepseek-hooks` → `deepseek-mcp` → `deepseek-tools` → `deepseek-core` → `deepseek-app-server` → `deepseek-tui-core` → `deepseek-tui-cli` → `deepseek-tui`。
5. 每发布一个 crate 需等待 crates.io 可见后再发依赖它的包。

发布辅助脚本可幂等重跑：已发布版本会跳过。

## GitHub Release 资源

`.github/workflows/release.yml` 构建以下二进制：

- `deepseek-linux-x64`、`deepseek-macos-x64`、`deepseek-macos-arm64`、`deepseek-windows-x64.exe`
- `deepseek-tui-linux-x64`、`deepseek-tui-macos-x64`、`deepseek-tui-macos-arm64`、`deepseek-tui-windows-x64.exe`

发布任务还上传 `deepseek-artifacts-sha256.txt`。npm 安装器与发布验证脚本均依赖该校验清单。

## npm 封装发布

**npm publish 步骤为手工。** `release.yml` 不再跑 `npm publish`，因 npm 账户每次发布需 2FA OTP，且尚未配置可绕过 2FA 的 automation token。GitHub Release 流程仍全自动化；仅 npm 封装需开发者在已 `npm login` 且带验证器应用的工作站操作。

### 步骤

1. [npm/deepseek-tui/package.json](../npm/deepseek-tui/package.json) 中 npm 包版本与 workspace `Cargo.toml` 对齐。CI 版本漂移检查会在 tag 前捕获不一致。
2. `deepseekBinaryVersion` 设为应提供二进制的 GitHub release tag。
3. 将版本 bump 推 `main`。`auto-tag.yml` 创建匹配 `vX.Y.Z`，`release.yml` 构建二进制矩阵并起草 GitHub Release。
4. **等待 GitHub Release 完成**，含全部八个签名二进制与 `deepseek-artifacts-sha256.txt`。npm `prepublishOnly`（`scripts/verify-release-assets.js`）要求资源齐全。
5. 在开发机手工发布 npm 封装：

```bash
cd npm/deepseek-tui
npm publish --access public
# （按提示输入验证器 OTP）
```

### 为何不自动化？

- 旧 `release.yml` 的 `publish-npm` job 使用 `secrets.NPM_TOKEN`，但 npm 默认 2FA 策略要求发布 token 要么为启用「Bypass 2FA for token authentication」的 automation token，要么账户级关闭 2FA——目前均未配置。
- 独立 `publish-npm.yml` 与 `crates-publish.yml` 已移除。未来若迁移到 npm Trusted Publishing（OIDC），可再引入专用工作流。

### 若日后修复 token

重新启用自动化发布：配置带「Bypass 2FA for token authentication」的 npm automation token（或通过 OIDC 设置 Trusted Publishing），在仓库保存对应 secret，并将 `publish-npm` job 加回 `release.yml`（或专用工作流），同时更新本节「手工」表述。

## 恢复与回滚

- Crate 部分发布：重跑 `./scripts/release/publish-crates.sh publish`，已发布版本会跳过。
- GitHub 资源缺失或校验清单不完整：修复 `.github/workflows/release.yml`，在 `npm publish` 前 retag 或上传修正资源。
- 仅 npm 封装问题：只 bump npm 版本、`deepseekBinaryVersion` 钉在上一已知良好 Rust 版，重新打包发布。
- 错误 npm 发布**不可覆盖**：发布带修正元数据或安装逻辑的新 npm 版本。
