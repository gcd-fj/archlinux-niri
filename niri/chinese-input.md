# niri 中文输入法配置

本文记录 Arch Linux + niri 下配置中文输入法的方法。niri 是 Wayland 合成器，推荐使用 `fcitx5`。如果只是日常中文输入，优先选择 `fcitx5-rime + rime-ice-git`；如果想简单省事，也可以先用 `fcitx5-chinese-addons`。

参考资料：

- https://github.com/SHORiN-KiWATA/Shorin-ArchLinux-Guide/wiki/中文输入法
- https://arch.icekylin.online/guide/rookie/desktop-env-and-app.html

## 1. 安装 fcitx5

安装基础框架：

```sh
sudo pacman -S --needed fcitx5-im fcitx5-configtool
```

`fcitx5-im` 会安装 fcitx5 的基础组件、GTK/Qt 输入法模块等常用包。`fcitx5-configtool` 用于图形化添加和调整输入法。

## 2. 选择中文输入方案

### 2.1 简单方案：中文输入合集

```sh
sudo pacman -S --needed fcitx5-chinese-addons
```

`fcitx5-chinese-addons` 包含拼音、双拼、五笔等常用方案，安装简单，适合先快速让中文输入可用。

### 2.2 推荐方案：Rime + 雾凇拼音

```sh
sudo pacman -S --needed fcitx5-rime
paru -S --needed rime-ice-git
```

说明：

- `fcitx5-rime`：Rime 中州韵输入法引擎
- `rime-ice-git`：雾凇拼音方案，通常从 AUR 或 archlinuxcn 安装

创建 Rime 用户配置目录：

```sh
mkdir -p ~/.local/share/fcitx5/rime
```

编辑默认方案配置：

```sh
vim ~/.local/share/fcitx5/rime/default.custom.yaml
```

写入：

```yaml
patch:
  __include: rime_ice_suggestion:/
```

重新部署 Rime 后生效：

```sh
fcitx5-remote -r
```

如果图形界面中没有立即生效，可以退出并重新登录 niri 会话。

## 3. niri 环境变量

niri 下通常只需要设置 `XMODIFIERS`：

```kdl
environment {
    XMODIFIERS "@im=fcitx"
}
```

把这段加入 `~/.config/niri/config.kdl`。修改后重新登录 niri，或者重启 niri 会话。

如果某些 GTK/Qt 应用无法输入中文，可再补充：

```kdl
environment {
    XMODIFIERS "@im=fcitx"
    GTK_IM_MODULE "fcitx"
    QT_IM_MODULE "fcitx"
}
```

不要一开始就把所有变量都写进全局环境。Wayland 原生应用通常优先走 text-input 协议，过多兼容变量有时反而会让特定应用表现异常。

## 4. niri 启动 fcitx5

在 `~/.config/niri/config.kdl` 中加入：

```kdl
spawn-at-startup "fcitx5"
```

如果使用 DMS，并且 DMS 或用户级服务已经启动了 fcitx5，不要重复启动。可先检查：

```sh
pgrep -a fcitx5
```

没有输出时，再考虑加入 `spawn-at-startup "fcitx5"`。

## 5. 添加输入法

打开配置工具：

```sh
fcitx5-configtool
```

常用设置：

- 关闭“仅显示当前语言”
- 添加 `Rime`
- 如果使用 `fcitx5-chinese-addons`，添加 `Pinyin`、`Shuangpin` 或 `Wubi`
- 在全局选项中设置切换快捷键

Rime 方案切换：

- 按 `F4` 打开 Rime 方案菜单
- 选择 `雾凇拼音` 或其他已安装方案

查看当前系统中已有的 Rime 方案：

```sh
ls /usr/share/rime-data/*.schema.yaml
```

文件名去掉 `.schema.yaml` 后就是 scheme 名称。

## 6. Electron / Chromium 应用

部分 Chromium 或 Electron 应用在 Wayland 下需要额外启动参数，否则中文输入候选框或输入焦点可能异常。

常用参数：

```sh
--enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime
```

如果从终端启动 VS Code，可以写 alias：

```sh
alias code='code --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime'
```

Fish 可写：

```fish
abbr code 'code --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime'
```

如果应用从 `.desktop` 启动，复制桌面文件到用户目录再改，不要直接修改 `/usr/share/applications` 中的文件：

```sh
mkdir -p ~/.local/share/applications
cp /usr/share/applications/code.desktop ~/.local/share/applications/
vim ~/.local/share/applications/code.desktop
```

把 `Exec=` 后面的命令追加 Wayland IME 参数。

## 7. Locale 与异常排查

查看当前 locale：

```sh
locale
```

不建议把 `LC_CTYPE` 固定成 `zh_CN.UTF-8` 后就不再检查。部分应用可能因为 `LC_CTYPE` 出现漏字、候选框异常或无法输入中文。可以按实际情况尝试：

```sh
LANG=en_US.UTF-8
LC_MESSAGES=zh_CN.UTF-8
```

如果某些应用必须中文 `LC_CTYPE` 才能输入中文，可只给该应用单独设置：

```sh
LC_CTYPE=zh_CN.UTF-8 app-command
```

常用检查命令：

```sh
pgrep -a fcitx5
fcitx5-diagnose
fcitx5-remote
```

如果输入法完全不可用，按顺序检查：

- `fcitx5` 进程是否存在
- `~/.config/niri/config.kdl` 是否设置了 `XMODIFIERS`
- 是否已经重新登录 niri 会话
- `fcitx5-configtool` 中是否添加了 Rime 或拼音输入法
- Electron/Chromium 应用是否需要 Wayland IME 参数

## 8. 卸载 fcitx5

按实际安装的包删除。Rime + 雾凇拼音示例：

```sh
sudo pacman -Rns fcitx5-im fcitx5-configtool fcitx5-rime
paru -Rns rime-ice-git
```

删除用户配置：

```sh
rm -rf ~/.config/fcitx5 ~/.local/share/fcitx5
```

最后从 `~/.config/niri/config.kdl` 中移除相关配置：

```kdl
environment {
    XMODIFIERS "@im=fcitx"
}

spawn-at-startup "fcitx5"
```
