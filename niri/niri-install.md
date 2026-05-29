# Arch Linux + niri 桌面安装整理

本文记录在 `linux/archlinux-basic-install.md` 完成之后，继续安装 niri 桌面环境时需要做的准备和配置。基础安装文档已经包含用户、sudo、NetworkManager、系统更新、`multilib`、`archlinuxcn`、AUR 助手等内容，这里不再重复。

参考资料：

- https://arch.icekylin.online/guide/rookie/graphic-driver.html
- https://github.com/SHORiN-KiWATA/Shorin-ArchLinux-Guide/wiki/安装桌面环境前的准备
- https://wiki.archlinux.org/title/Niri
- https://danklinux.com/docs/dankmaterialshell/installation/
- https://danklinux.com/docs/dankmaterialshell/compositors/

> 建议在安装显卡驱动、桌面组件或 DMS 前先更新系统。如果已经配置 Timeshift，也建议先创建一张快照。

## 0. 前置检查

确认已经完成基础安装文档中的内容：

- 已创建普通用户，并能使用 `sudo`
- 已完成 `pacman -Syu`
- 如需 Steam、Wine、32 位显卡库，已开启 `multilib`
- 如需从中文社区源安装 AUR 助手，已配置 `archlinuxcn`
- 已安装并可使用 `yay` 或 `paru`

确认当前显卡：

```sh
lspci -k | grep -A 3 -E "VGA|3D|Display"
```

如果是虚拟机，通常不需要安装实体机显卡驱动。

## 1. 基础桌面组件

安装常用目录、音频、蓝牙、电源模式、文件系统支持和字体：

```sh
sudo pacman -S --needed \
  xdg-user-dirs ntfs-3g \
  sof-firmware alsa-firmware alsa-ucm-conf \
  pipewire wireplumber pipewire-pulse pipewire-alsa pipewire-jack \
  bluez power-profiles-daemon \
  noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra \
  adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts \
  ttf-jetbrains-mono-nerd
```

生成用户目录：

```sh
xdg-user-dirs-update
```

启用系统服务：

```sh
sudo systemctl enable --now bluetooth
sudo systemctl enable --now power-profiles-daemon
```

PipeWire 相关用户服务一般会由桌面会话按需拉起。如果需要手动启用：

```sh
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```

## 2. 显卡驱动

Wayland/niri 下优先使用内核 KMS 与 Mesa。显卡驱动安装完成后，建议重启一次再继续安装桌面。

### 2.1 Intel 核显

```sh
sudo pacman -S --needed mesa lib32-mesa vulkan-intel lib32-vulkan-intel
```

通常不建议安装 `xf86-video-intel`，默认 modesetting 驱动更稳。较老的 Intel HD 4000 以下型号不支持 Vulkan，可不安装 `vulkan-intel`。

### 2.2 AMD 集显或独显

新 AMD 显卡通常使用开源 AMDGPU 驱动：

```sh
sudo pacman -S --needed mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon
```

较老的 ATI/TeraScale 显卡使用：

```sh
sudo pacman -S --needed mesa lib32-mesa xf86-video-ati
```

AMD 双显卡需要让程序使用独显时，可加 `DRI_PRIME=1`：

```sh
DRI_PRIME=1 glmark2
DRI_PRIME=1 steam
```

### 2.3 NVIDIA 独显

Turing/RTX 20、GTX 16 系列及更新显卡，当前 Arch 官方包优先使用 open kernel modules：

```sh
sudo pacman -S --needed nvidia-open nvidia-settings nvidia-utils lib32-nvidia-utils
```

如果使用非标准内核，例如 `linux-zen`、`linux-lts` 或自定义内核，安装 DKMS 版本，并确保对应内核 headers 已安装：

```sh
sudo pacman -S --needed nvidia-open-dkms nvidia-settings nvidia-utils lib32-nvidia-utils
```

Pascal/GTX 10 系列及更老显卡需要按实际型号选择 legacy 驱动或 nouveau。安装前先查 Arch Wiki 和 AUR 包状态，不要直接套用新显卡命令。

安装 NVIDIA 官方驱动后，检查 `/etc/mkinitcpio.conf` 的 `HOOKS`。如果里面有 `kms`，按当前 Arch/NVIDIA 建议评估是否移除，避免 nouveau 被打进 initramfs。修改后重新生成 initramfs：

```sh
sudo mkinitcpio -P
```

NVIDIA 双显卡需要用独显运行程序时，优先安装并使用 `nvidia-prime`：

```sh
sudo pacman -S --needed nvidia-prime
prime-run glmark2
prime-run steam
```

不建议把下面这些 PRIME 环境变量写入全局环境，否则可能导致混成器异常或黑屏：

```sh
__NV_PRIME_RENDER_OFFLOAD=1
__GLX_VENDOR_LIBRARY_NAME=nvidia
__VK_LAYER_NV_optimus=NVIDIA_only
```

### 2.4 测试与排错

安装测试工具：

```sh
sudo pacman -S --needed mesa-utils glmark2
```

测试 OpenGL：

```sh
glxgears
glmark2
```

查看渲染器：

```sh
glxinfo -B
```

如果安装驱动后黑屏，可用 `Ctrl+Alt+F1` 到 `Ctrl+Alt+F6` 切换 TTY，登录后回滚配置或恢复快照。

## 3. niri + DMS 桌面方案

DMS（DankMaterialShell）不是单纯的 quickshell 主题，而是一套面向 Wayland 合成器的完整桌面 shell。它会替代传统 niri 手动拼装方案里的 `waybar`、`mako`、`fuzzel`、`swaylock`、`swayidle`、polkit 前端等组件，并提供启动器、通知中心、控制中心、锁屏、空闲管理、壁纸/主题、截图、剪贴板、亮度、音量等能力。

因此如果目标就是 niri + DMS，不需要先按传统方式把 niri 生态组件全部手动装一遍。推荐顺序是：

1. 先完成显卡驱动和基础桌面组件
2. 再用 DMS 安装器或 Arch 官方包安装 niri + DMS
3. 最后补齐自己需要的浏览器、文件管理、输入法、Flatpak、Steam 等应用

### 3.1 推荐：DMS 安装器

官方安装入口：

```sh
curl -fsSL https://install.danklinux.com | sh
```

这个脚本本身不是直接安装全部内容，而是：

- 检测系统架构
- 从 GitHub 获取 DankMaterialShell 最新 release
- 下载 `dankinstall` 二进制安装器
- 校验 sha256
- 运行 `dankinstall`

`dankinstall` 是交互式安装器，会按发行版处理依赖和配置。在 Arch 上，DMS 已经进入官方 `extra` 仓库；niri 方案会安装 DMS、niri 以及相关依赖，并生成/调整对应配置。

执行前建议：

- 不要用 root 用户运行
- 已完成 `sudo pacman -Syu`
- 已安装并重启到正确显卡驱动
- 已配置 Timeshift 或其他快照
- 如果已有 `~/.config/niri/config.kdl`，先备份

```sh
if [ -d ~/.config/niri ]; then
  cp -a ~/.config/niri ~/.config/niri.bak.$(date +%Y%m%d-%H%M%S)
fi
```

安装完成后重新登录，在显示管理器中选择 niri；如果没有显示管理器，可在 TTY 中启动：

```sh
niri-session
```

DMS 通常通过用户级 systemd 服务随 niri 会话启动。可检查：

```sh
systemctl --user status dms
dms doctor
```

常用排错命令：

```sh
journalctl --user -u dms -n 80
dms restart
dms kill
```

### 3.2 可选：Arch 官方包手动安装

如果不想运行远程安装脚本，可以直接使用 Arch 官方仓库包。niri 官方快速开始给出的 Arch 组合是：

```sh
sudo pacman -S --needed \
  niri xwayland-satellite \
  xdg-desktop-portal-gnome xdg-desktop-portal-gtk \
  alacritty dms-shell-niri matugen cava qt6-multimedia-ffmpeg
```

让 DMS 作为 niri 会话的一部分启动：

```sh
systemctl --user add-wants niri.service dms
```

然后重新登录 niri 会话，或在 TTY 启动：

```sh
niri-session
```

包说明：

- `niri`：滚动式平铺 Wayland 合成器
- `xwayland-satellite`：让 X11 应用在 niri 下运行
- `xdg-desktop-portal-gtk`、`xdg-desktop-portal-gnome`：文件选择、屏幕共享、Flatpak 等 portal 能力
- `dms-shell-niri`：Arch 官方的 DMS niri 集成包，依赖 `dms-shell` 和 `niri`
- `matugen`：基于壁纸生成 Material 配色
- `cava`：音频可视化
- `qt6-multimedia-ffmpeg`：DMS 声音反馈相关能力

如果进入 niri 后同时出现 Waybar 和 DMS bar，说明默认 niri 配置里还保留了 `spawn-at-startup "waybar"`。编辑 `~/.config/niri/config.kdl`，删除或注释这行，再重启 niri。

### 3.3 niri 与 DMS 配置关系

DMS 官方建议在 `~/.config/niri/config.kdl` 末尾包含它生成的配置片段：

```kdl
include "dms/colors.kdl"
include "dms/layout.kdl"
include "dms/alttab.kdl"
include "dms/binds.kdl"
```

这些文件用于让 DMS 管理配色、布局、Alt-Tab 和快捷键。安装器通常会处理这部分；手动安装时如果文件不存在，不要直接 include，先确认：

```sh
ls ~/.config/niri/dms
```

如果要手动让 niri 启动 DMS，也可以在 niri 配置中加入：

```kdl
spawn-at-startup "dms" "run"
```

但如果已经用了 `systemctl --user add-wants niri.service dms`，不要再重复写 `spawn-at-startup "dms" "run"`，避免启动两个 DMS 实例。

## 4. 不使用 DMS 时的传统 niri 组件

这一节只适合想要自己拼装 niri 桌面的情况。如果使用 DMS，通常不需要安装这些替代组件。

```sh
sudo pacman -S --needed \
  niri xwayland-satellite \
  xdg-desktop-portal xdg-desktop-portal-gtk xdg-desktop-portal-gnome \
  fuzzel mako waybar alacritty swaybg swayidle swaylock \
  polkit-gnome brightnessctl playerctl grim slurp wl-clipboard \
  udiskie gvfs gvfs-mtp
```

对应关系：

- `waybar`：状态栏，DMS 已提供 bar
- `fuzzel`：启动器，DMS 已提供 spotlight/launcher
- `mako`：通知，DMS 已提供通知中心
- `swaylock`、`swayidle`：锁屏和空闲管理，DMS 已提供会话管理
- `polkit-gnome`：认证弹窗，DMS 可处理 shell 侧集成
- `grim`、`slurp`、`wl-clipboard`：截图和剪贴板，DMS 也提供对应 CLI/界面能力，但这些工具仍可按个人习惯保留

niri 包提供桌面入口，可在显示管理器中选择 niri。没有显示管理器时从 TTY 启动：

```sh
niri-session
```

## 5. 常用软件

按需安装日常软件：

```sh
sudo pacman -S --needed fish firefox chromium obsidian flatpak ark gwenview steam
paru -S --needed flclash
```

推荐使用 Nautilus 作为文件管理器：

```sh
sudo pacman -S --needed nautilus gvfs gvfs-mtp gvfs-afc file-roller sushi
```

说明：

- `fish`：交互式 shell，确认需要后再切换默认 shell
- `firefox`、`chromium`：浏览器
- `obsidian`：笔记
- `flatpak`：适合安装 OBS、EasyEffects 等依赖复杂的软件
- `ark`、`gwenview`：压缩包和图片查看
- `steam`：需要 `multilib` 和对应 32 位显卡库
- `flclash`：AUR/第三方包，按实际网络环境选择安装方式
- `nautilus`：GNOME 文件管理器，和 GTK/Portal 环境兼容性较好
- `gvfs`、`gvfs-mtp`、`gvfs-afc`：提供挂载、安卓 MTP 和 iPhone 访问支持
- `file-roller`、`sushi`：提供压缩包处理和空格预览

Flatpak 可选更换国内镜像：

```sh
sudo flatpak remote-modify flathub --url=https://mirror.sjtu.edu.cn/flathub
```

或：

```sh
sudo flatpak remote-modify flathub --url=https://mirrors.ustc.edu.cn/flathub
```

## 6. 休眠到硬盘

只有使用 Swap 分区且确实需要休眠时再配置。先查看当前 initramfs hooks：

```sh
grep ^HOOKS /etc/mkinitcpio.conf
```

如果使用 `systemd` hooks，通常不需要额外添加 `resume`。如果使用 `udev` hooks，需要在 `udev` 后面加入 `resume`，然后重新生成 initramfs：

```sh
sudo vim /etc/mkinitcpio.conf
sudo mkinitcpio -P
```

测试休眠：

```sh
systemctl hibernate
```

## 7. 安装后检查

```sh
systemctl --user status pipewire wireplumber
systemctl status bluetooth power-profiles-daemon
niri --version
dms --version
dms doctor
glxinfo -B
```

检查清单：

- [ ] 显卡驱动与 32 位库已按硬件类型安装
- [ ] 重启后能进入 niri 会话
- [ ] Firefox/Chromium 文件选择器正常
- [ ] 截图、剪贴板、亮度、音量可用
- [ ] 蓝牙、U 盘、移动硬盘按需可用
- [ ] Steam 或 Wine 能识别正确显卡
- [ ] DMS 安装后没有覆盖重要个人配置
