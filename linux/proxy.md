# 代理与开发加速配置

本文记录常用开发网络配置，包括 Git 访问 GitHub 走本机 SOCKS5 代理，npm 使用阿里云镜像加速并设置全局存储位置，以及 Rust/rustup 使用阿里云镜像安装和下载依赖。

## GitHub 代理

下面示例使用 `127.0.0.1:1080`，实际使用时把 `1080` 改成自己的代理软件提供的 SOCKS5 端口。

### 只让 GitHub 走代理

```sh
git config --global http.https://github.com.proxy socks5://127.0.0.1:1080
git config --global https.https://github.com.proxy socks5://127.0.0.1:1080
```

这两条配置只对 `https://github.com` 生效，不会影响所有 Git 仓库。

### 查看配置

```sh
git config --global --get http.https://github.com.proxy
git config --global --get https.https://github.com.proxy
```

也可以查看所有全局 Git 配置：

```sh
git config --global --list
```

### 取消配置

```sh
git config --global --unset http.https://github.com.proxy
git config --global --unset https.https://github.com.proxy
```

### 说明

- `socks5://127.0.0.1:1080` 表示使用本机 SOCKS5 代理
- 如果代理软件端口不是 `1080`，需要改成实际端口
- 这个配置只影响 HTTPS 形式的 GitHub 地址，例如 `https://github.com/user/repo.git`
- 如果仓库远端是 SSH 地址，例如 `git@github.com:user/repo.git`，这组 Git HTTP 代理配置不会生效

## npm 阿里云加速

npm 官方源在国内访问不稳定时，可以改用阿里云 npm 镜像：

```sh
npm config set registry https://registry.npmmirror.com
```

查看当前 registry：

```sh
npm config get registry
```

恢复官方源：

```sh
npm config set registry https://registry.npmjs.org
```

## npm 全局存储位置

默认情况下，npm 全局包可能安装到系统目录，容易遇到权限问题。建议把全局包和缓存放到用户目录下。

创建目录：

```sh
mkdir -p ~/.npm-global ~/.npm-cache
```

设置全局安装目录和缓存目录：

```sh
npm config set prefix ~/.npm-global
npm config set cache ~/.npm-cache
```

把全局命令目录加入 `PATH`。如果使用 Bash：

```sh
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

如果使用 Zsh：

```sh
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

如果使用 Fish：

```fish
fish_add_path ~/.npm-global/bin
```

检查配置：

```sh
npm config get prefix
npm config get cache
npm config get registry
```

测试全局安装：

```sh
npm install -g pnpm
pnpm --version
```

## Rust 阿里云镜像

Rust 官方安装脚本会通过 rustup 下载工具链。国内网络访问 `static.rust-lang.org` 较慢或失败时，可以使用阿里云 rustup 镜像。

### 临时使用阿里云镜像安装

只对当前终端生效：

```sh
export RUSTUP_DIST_SERVER=https://mirrors.aliyun.com/rustup
export RUSTUP_UPDATE_ROOT=https://mirrors.aliyun.com/rustup/rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装完成后让当前终端加载 Cargo 环境：

```sh
source "$HOME/.cargo/env"
```

检查版本：

```sh
rustc --version
cargo --version
rustup --version
```

### 永久配置 rustup 镜像

如果使用 Bash：

```sh
echo 'export RUSTUP_DIST_SERVER=https://mirrors.aliyun.com/rustup' >> ~/.bashrc
echo 'export RUSTUP_UPDATE_ROOT=https://mirrors.aliyun.com/rustup/rustup' >> ~/.bashrc
source ~/.bashrc
```

如果使用 Zsh：

```sh
echo 'export RUSTUP_DIST_SERVER=https://mirrors.aliyun.com/rustup' >> ~/.zshrc
echo 'export RUSTUP_UPDATE_ROOT=https://mirrors.aliyun.com/rustup/rustup' >> ~/.zshrc
source ~/.zshrc
```

如果使用 Fish：

```fish
set -Ux RUSTUP_DIST_SERVER https://mirrors.aliyun.com/rustup
set -Ux RUSTUP_UPDATE_ROOT https://mirrors.aliyun.com/rustup/rustup
```

### 配置 Cargo 使用阿里云 crates.io 镜像

创建 Cargo 配置目录：

```sh
mkdir -p ~/.cargo
```

编辑 `~/.cargo/config.toml`：

```sh
vim ~/.cargo/config.toml
```

写入：

```toml
[source.crates-io]
replace-with = "aliyun"

[source.aliyun]
registry = "sparse+https://mirrors.aliyun.com/crates.io-index/"
```

Cargo 1.68 及之后支持 sparse registry，通常优先使用上面的 `sparse+` 配置。

测试依赖下载：

```sh
cargo search tokio
```

### 恢复官方源

取消 rustup 镜像环境变量：

```sh
unset RUSTUP_DIST_SERVER
unset RUSTUP_UPDATE_ROOT
```

如果已经写入 shell 配置文件，需要从 `~/.bashrc`、`~/.zshrc` 或 Fish universal variable 中删除对应配置。

恢复 Cargo 官方源时，删除或注释 `~/.cargo/config.toml` 中的这几段：

```toml
[source.crates-io]
replace-with = "aliyun"

[source.aliyun]
registry = "sparse+https://mirrors.aliyun.com/crates.io-index/"
```
