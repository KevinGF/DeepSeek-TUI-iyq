# 安装 DeepSeek TUI

> **English**：[INSTALL.md](INSTALL.md)

本文涵盖所有支持的安装路径与最常见的「装不上」故障，包括 **Linux ARM64** 与其它较少见平台。

若只要最短路径，见根目录 [README](../README.md#quickstart) 或 [简体中文 README](../README.zh-CN.md#快速开始)。

---

## 1. 支持的平台

自 v0.8.8 起，`deepseek-tui` 为以下平台/架构组合提供预编译二进制：

| 平台     | 架构 | npm install | `cargo install` | GitHub release 资源 |
| ------------ | ------------ | :---------: | :-------------: | ----------------------------------------------------- |
| Linux        | x64 (x86_64) |     ✅      |       ✅        | `deepseek-linux-x64`, `deepseek-tui-linux-x64`        |
| Linux        | arm64        |     ✅      |       ✅        | `deepseek-linux-arm64`, `deepseek-tui-linux-arm64`    |
| macOS        | x64          |     ✅      |       ✅        | `deepseek-macos-x64`, `deepseek-tui-macos-x64`        |
| macOS        | arm64 (M 系列) | ✅      |       ✅        | `deepseek-macos-arm64`, `deepseek-tui-macos-arm64`    |
| Windows      | x64          |     ✅      |       ✅        | `deepseek-windows-x64.exe`, `deepseek-tui-windows-x64.exe` |
| 其它 Linux（musl、riscv64 等） | — |   ❌¹    |       ✅²       | 源码构建                                     |
| FreeBSD / OpenBSD              | — |   ❌      |       ✅²       | 源码构建                                     |

¹ npm 包会以明确错误退出并指向本文。  
² 需工具链能编译较新的 Rust workspace；见下文 [从源码构建](#5-从源码构建)。

> **Linux ARM64 说明（v0.8.7 及更早）。** v0.8.7 及更早**不**发布 Linux ARM64 预编译；鸿蒙轻薄本、Asahi Linux、树莓派、AWS Graviton 等会看到 `npm i -g deepseek-tui` 报 `Unsupported architecture: arm64`。v0.8.8 起发布 `deepseek-linux-arm64` 与 `deepseek-tui-linux-arm64`，在基于 glibc 的 ARM64 Linux 上可直接 `npm i -g deepseek-tui`。若卡在 v0.8.7，见 [从源码构建](#5-从源码构建)——`cargo install` 可用。

---

## 2. 通过 npm 安装（推荐）

```bash
npm install -g deepseek-tui
deepseek
```

`postinstall` 从对应 GitHub release 下载匹配的二进制对，校验 SHA-256 清单，并将 `deepseek` 与 `deepseek-tui` 放到 `PATH`。

常用环境变量：

| 变量                            | 用途                                                                                |
| ----------------------------------- | -------------------------------------------------------------------------------------- |
| `DEEPSEEK_TUI_VERSION`              | 固定 wrapper 下载的版本（默认 `deepseekBinaryVersion`）          |
| `DEEPSEEK_TUI_GITHUB_REPO`          | 将下载指向 fork（`owner/repo`）                                          |
| `DEEPSEEK_TUI_RELEASE_BASE_URL`     | 覆盖下载根（如内网镜像或 release 代理）            |
| `DEEPSEEK_TUI_FORCE_DOWNLOAD=1`     | 即使缓存标记匹配也重新下载                                     |
| `DEEPSEEK_TUI_DISABLE_INSTALL=1`    | 完全跳过 `postinstall` 下载（CI 冒烟、自带二进制）                 |
| `DEEPSEEK_TUI_OPTIONAL_INSTALL=1`   | 下载/解压失败不使 `npm install` 失败——适用于 CI 矩阵            |

> **中国大陆 npm 下载慢？** 若 `npm install` 本身慢（不仅是 postinstall 二进制），可使用 npm 镜像：
> ```bash
> npm config set registry https://registry.npmmirror.com
> npm install -g deepseek-tui
> ```
> 若更倾向 Cargo，见 [第 3 节](#3-通过-cargo-安装任意-tier-1-rust-目标)。

---

## 3. 通过 Cargo 安装（任意 Tier-1 Rust 目标）

若 GitHub release 慢、被墙，或架构不受支持，可直接从 crates.io 安装。需要**两个** crate——dispatcher 运行时会委托给 TUI。

```bash
# 需要 Rust 1.88+（https://rustup.rs）
cargo install deepseek-tui-cli --locked   # 提供 `deepseek`
cargo install deepseek-tui     --locked   # 提供 `deepseek-tui`
deepseek --version
```

### 中国大陆 / 镜像友好安装

在大陆安装时，为 **rustup**（工具链）与 **Cargo**（注册表）配置镜像，减少 TLS 超时与下载失败。

**步骤 1：通过 rustup 镜像安装 Rust**

```bash
# PowerShell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
(New-Object Net.WebClient).DownloadFile('https://win.rustup.rs/x86_64', 'rustup-init.exe')

# git-bash / msys2
export RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.tsinghua.edu.cn/rustup/rustup
./rustup-init.exe -y --default-toolchain stable

# Linux / macOS
export RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.tsinghua.edu.cn/rustup/rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
```

`RUSTUP_DIST_SERVER` 与 `RUSTUP_UPDATE_ROOT` 必须在运行 rustup-init **之前**设置；否则工具链下载仍可能遇到与安装器相同的 TLS 握手问题。

**步骤 2：配置 Cargo 注册表镜像**

```toml
# ~/.cargo/config.toml
[source.crates-io]
replace-with = "tuna"

[source.tuna]
registry = "sparse+https://mirrors.tuna.tsinghua.edu.cn/crates.io-index/"
```

`rsproxy`、腾讯 COS、阿里云 OSS 等镜像同理；选你网络最快的。

---

## 4. 从 GitHub Releases 手动下载

从 [Releases](https://github.com/Hmbown/DeepSeek-TUI/releases) 取匹配平台的二进制对，放到 `PATH` 上同一目录（如 `~/.local/bin`）：

```bash
# Linux ARM64 示例
mkdir -p ~/.local/bin
curl -L -o ~/.local/bin/deepseek      \
    https://github.com/Hmbown/DeepSeek-TUI/releases/latest/download/deepseek-linux-arm64
curl -L -o ~/.local/bin/deepseek-tui  \
    https://github.com/Hmbown/DeepSeek-TUI/releases/latest/download/deepseek-tui-linux-arm64
chmod +x ~/.local/bin/deepseek ~/.local/bin/deepseek-tui
deepseek --version
```

按每版本 SHA-256 清单校验：

```bash
curl -L -o /tmp/deepseek-artifacts-sha256.txt \
    https://github.com/Hmbown/DeepSeek-TUI/releases/latest/download/deepseek-artifacts-sha256.txt
( cd ~/.local/bin && sha256sum -c /tmp/deepseek-artifacts-sha256.txt --ignore-missing )
```

macOS 用 `shasum -a 256 -c` 代替 `sha256sum`。

---

## 5. 从源码构建

适用于未官方提供平台的兜底方案——含 musl、riscv64、龙芯、FreeBSD，以及 2024 年前的 ARM64 发行版。

### 前置条件

- **Rust** 1.88+ —— 用 [rustup](https://rustup.rs) 安装。
- **Linux 构建依赖**（Debian/Ubuntu/openEuler/麒麟）：
  ```bash
  sudo apt-get install -y build-essential pkg-config libdbus-1-dev
  # openEuler / RHEL 系：
  # sudo dnf install -y gcc make pkgconf-pkg-config dbus-devel
  ```
- **不**需要可用的 `cmake`。

### 构建并安装

```bash
git clone https://github.com/Hmbown/DeepSeek-TUI.git
cd DeepSeek-TUI

cargo install --path crates/cli --locked   # 提供 `deepseek`
cargo install --path crates/tui --locked   # 提供 `deepseek-tui`

deepseek --version
```

两个二进制默认在 `~/.cargo/bin/`；确保该目录在 `PATH` 中。

### 从 x64 交叉编译到 ARM64 Linux

若在 x64 Linux 上为 ARM64 目标构建（如鸿蒙 / openEuler ARM64 轻薄本），可用 [`cross`](https://github.com/cross-rs/cross)（Docker 封装官方 Rust 交叉目标）：

```bash
# 一次性
rustup target add aarch64-unknown-linux-gnu
cargo install cross --locked

# 每次构建
cross build --release --target aarch64-unknown-linux-gnu -p deepseek-tui-cli
cross build --release --target aarch64-unknown-linux-gnu -p deepseek-tui
```

产物在  
`target/aarch64-unknown-linux-gnu/release/deepseek` 与  
`target/aarch64-unknown-linux-gnu/release/deepseek-tui`。复制匹配的一对到 ARM64 主机（如 `scp`）并 `chmod +x`。

若无 Docker，直接装交叉链接器让 Cargo 构建：

```bash
sudo apt-get install -y gcc-aarch64-linux-gnu
rustup target add aarch64-unknown-linux-gnu

cat >> ~/.cargo/config.toml <<'EOF'
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
EOF

cargo build --release --target aarch64-unknown-linux-gnu -p deepseek-tui-cli
cargo build --release --target aarch64-unknown-linux-gnu -p deepseek-tui
```

musl 目标 `aarch64-unknown-linux-musl` 同理。

### Windows 源码构建

Windows 构建需要 [Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022) 中的 **MSVC C 工具链**（免费可选组件安装器，非完整 IDE）。

**前置（Windows）**

1. 安装 VS 2022 Build Tools —— 选 **「使用 C++ 的桌面开发」** 工作负载。
2. 安装 [Rust](https://rustup.rs) 1.88+（若在大陆下载，见上文 [中国大陆镜像](#中国大陆--镜像友好安装)）。
3. 安装 [Git for Windows](https://git-scm.com/download/win)（提供 `git` 与 `git-bash`）。

**推荐终端**：Windows Terminal、`git-bash` 或 PowerShell。`cmd.exe` 可用但缓冲小、PATH 行为受限。

**配置 MSVC 环境**

Build Tools 把 `cl.exe` 装到版本化目录，但**不会**全局加入 `PATH`。须手动设环境或使用 Developer Command Prompt。所需变量示例：

```powershell
# 按你安装调整版本号
$msvc = "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.44.35207"
$sdk   = "C:\Program Files (x86)\Windows Kits\10"
$sdkv  = "10.0.26100.0"

$env:INCLUDE  = "$msvc\include;$msvc\atlmfc\include;$sdk\Include\$sdkv\ucrt;$sdk\Include\$sdkv\um;$sdk\Include\$sdkv\shared"
$env:LIB      = "$msvc\lib\x64;$msvc\atlmfc\lib\x64;$sdk\Lib\$sdkv\ucrt\x64;$sdk\Lib\$sdkv\um\x64"
$env:LIBPATH  = "$msvc\lib\x64;$msvc\atlmfc\lib\x64"
$env:CC       = "$msvc\bin\Hostx64\x64\cl.exe"
$env:CXX      = "$msvc\bin\Hostx64\x64\cl.exe"
$env:PATH     = "$msvc\bin\Hostx64\x64;$env:PATH"
```

或从开始菜单打开 **「VS 2022 开发人员命令提示」**，它会运行 `vcvars64.bat` 自动配置。再在该会话把 `cargo` 加入 `PATH` 后在项目根运行 `cargo build`。

**Cargo 注册表镜像** —— Windows 下配置文件在 `%USERPROFILE%\.cargo\config.toml`。见上文 [步骤 2](#中国大陆--镜像友好安装)。

**构建**

```bash
git clone https://github.com/Hmbown/DeepSeek-TUI.git
cd DeepSeek-TUI
set CARGO_HTTP_CHECK_REVOKE=false   # 部分中国 ISP 后可能需要
cargo build --release
```

产物在 `target\release\deepseek.exe` 与 `target\release\deepseek-tui.exe`。

> **除非要改源码，Windows 上更推荐 `npm install -g`。** npm 包拉预编译二进制，完全避开 C 工具链依赖——见 [第 2 节](#2-通过-npm-安装推荐)。

---

## 6. 故障排除

### `Unsupported architecture: arm64 on platform linux`

release 早于 v0.8.8，未发布 Linux ARM64 二进制。升级（`npm i -g deepseek-tui@latest`）或按 [第 3 节](#3-通过-cargo-安装任意-tier-1-rust-目标) 使用 `cargo install`。

### 运行时 `MISSING_COMPANION_BINARY`

dispatcher（`deepseek`）需要 TUI 运行时（`deepseek-tui`）在同一 `PATH`。若只 `cargo install` 了一个 crate，请两个都装：

```bash
cargo install deepseek-tui-cli --locked
cargo install deepseek-tui     --locked
```

### `deepseek update` 报 `no asset found for platform deepseek-linux-aarch64`

v0.8.7 的 [#503](https://github.com/Hmbown/DeepSeek-TUI/issues/503)——自更新器用了 Rust 的 `aarch64`/`x86_64` 名，而 release 资源用 `arm64`/`x64`。v0.8.8 前变通：

```bash
npm i -g deepseek-tui@latest
# 或
cargo install deepseek-tui-cli --locked
```

### 中国大陆 npm 下载慢或超时

设 `DEEPSEEK_TUI_RELEASE_BASE_URL` 指向镜像 release 目录（rsproxy、TUNA、腾讯 COS、阿里云 OSS），或跳过 npm 用 [第 3 节](#3-通过-cargo-安装任意-tier-1-rust-目标) 的 Cargo 镜像。

### Debian/Ubuntu：构建时 `error: linker 'cc' not found`

安装 C 工具链：

```bash
sudo apt-get install -y build-essential pkg-config libdbus-1-dev
```

### Wrapper 装好但找不到 `deepseek`

`npm i -g` 安装到 `$(npm prefix -g)/bin`；确认该目录在 shell 的 `PATH` 中。用 nvm 时：`nvm use --lts && hash -r`。

### Windows：`rustup-init` 报 `TLS handshake eof` 或 `CRYPT_E_REVOCATION_OFFLINE`

到 `static.rust-lang.org` 的 TLS 在 GFW 或部分中国 ISP 后失败。在运行安装器**之前**设置 rustup 镜像：

```bash
# git-bash / msys2
export RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.tuna.tsinghua.edu.cn/rustup/rustup
./rustup-init.exe -y --default-toolchain stable
```

若 Rust 装好后 Cargo 仍报 `CRYPT_E_REVOCATION_OFFLINE`，`cargo build` 期间设 `CARGO_HTTP_CHECK_REVOKE=false`。

### Windows：`cargo build` 时找不到 MSVC（`cl.exe`）

VS Build Tools 不会把 `cl.exe` 加进全局 `PATH`。要么：

1. 从开始菜单打开 **「VS 2022 开发人员命令提示」**，在该窗口把 `%USERPROFILE%\.cargo\bin` 加进 `PATH` 再 `cargo build`；或  
2. 手动设 MSVC 环境变量——见上文 [Windows 源码构建](#windows-源码构建) 的 PowerShell 片段。

验证：`cl.exe /?` 应打印帮助。

### Windows：Cargo 执行 build script 时 `拒绝访问 (os error 5)`

第三方杀软（火绒、360、卡巴斯基等）可能阻止 Cargo 执行刚编译的 build-script 二进制（如 `libsqlite3-sys`、`aws-lc-sys`、`instability`）。错误与路径无关——改 `target-dir` 无效。

**现象**：`could not execute process ... build-script-build (never executed)`

**变通**（选一）：

1. **把项目 `target/` 加进杀软排除列表。**
2. **暂时关闭杀软** 再 `cargo build`。
3. **改用 `npm install -g deepseek-tui`** —— 预编译，完全跳过 Cargo 构建（[第 2 节](#2-通过-npm-安装推荐)）。
4. **用 `cargo install deepseek-tui-cli --locked`** 从 crates.io 装——二进制路径不同，部分杀软行为不同。

要验证 build-script 本身是否可执行（非损坏），在 `target/debug/build/<crate>/build-script-build` 下找到并手动运行：

```bash
target/debug/build/libsqlite3-sys-*/build-script-build
# 若能运行但 panic "NotPresent"（无 C 编译器），说明二进制本身没问题——
# 是杀软在拦 Cargo 的进程创建路径。
```

---

## 7. 验证安装

```bash
deepseek --version
deepseek doctor       # 检查 API key、提供商、运行时与 PATH 完整性
deepseek doctor --json
```

`doctor` 发现问题时非零退出并打印结构化修复提示。如需帮助可把 JSON 贴到 GitHub issue。
