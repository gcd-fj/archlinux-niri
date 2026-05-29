# Arch Linux 基础安装整理

本文整理自「archlinux 简明指南」的基础安装页面，目标是记录一套可复用的 Arch Linux 基础安装流程。本文侧重 UEFI 启动、Btrfs 文件系统、`@` 与 `@home` 子卷、GRUB 引导和基础网络配置。

原文链接：https://arch.icekylin.online/guide/rookie/basic-install.html

> 重要提醒：分区、格式化、挂载和引导安装都可能影响磁盘数据。操作前先确认目标磁盘和分区，涉及双系统时尤其不要误格式化 Windows 使用的 EFI 分区。

## 0. 进入安装环境

从 Arch Linux 安装介质启动，进入安装环境后开始执行命令。

常用辅助命令：

```sh
clear
rmmod pcspkr
```

如果要永久禁用蜂鸣器，可在安装完成后的系统中创建或编辑 `/etc/modprobe.d/blacklist.conf`：

```conf
blacklist pcspkr
```

## 1. 确认 UEFI 启动模式

检查是否存在 EFI 变量：

```sh
ls /sys/firmware/efi/efivars
```

如果能列出内容，说明当前是 UEFI 模式。若没有输出或路径不存在，需要回到 BIOS/UEFI 设置中确认启动方式。

## 2. 连接网络

Arch Linux 安装过程需要联网。

无线网络可使用 `iwctl`：

```sh
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect wifi-name
exit
```

其中 `wlan0` 要替换成实际无线网卡名称，`wifi-name` 要替换成实际 SSID。

如果无线网卡没有正常启用，可检查设备和软硬件开关：

```sh
lspci -k | grep Network
rfkill list
ip link set wlan0 up
rfkill unblock wifi
```

有线网络通常插入可用网线后会通过 DHCP 自动联网。

## 3. 测试网络

```sh
ping www.bilibili.com
```

能看到返回数据说明网络可用。`ping` 不会自动退出，需要按 `Ctrl+C` 停止。

## 4. 同步系统时间

```sh
timedatectl set-ntp true
timedatectl status
```

系统时间不准确可能导致证书校验、软件包下载等问题。

## 5. 配置 pacman 镜像源

编辑 `/etc/pacman.d/mirrorlist`：

```sh
vim /etc/pacman.d/mirrorlist
```

可将常用国内镜像放在文件顶部：

```conf
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://repo.huaweicloud.com/archlinux/$repo/os/$arch
Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch
```

此阶段不要添加 `archlinuxcn` 源，避免基础安装时引入额外变量。

## 6. 分区规划

示例方案适用于 UEFI + Btrfs：

| 挂载点/用途 | 建议大小 | 说明 |
| --- | --- | --- |
| `/boot` | 约 256 MiB 或使用已有 EFI 分区 | 双系统时通常复用 Windows 创建的 EFI 分区 |
| Swap | 内存的 60% 或与内存相同 | 为休眠预留空间时可适当增大 |
| `/` | 128 GiB 以上 | Btrfs 分区中的 `@` 子卷 |
| `/home` | 128 GiB 以上 | Btrfs 分区中的 `@home` 子卷 |

Btrfs 下 `/` 和 `/home` 可以位于同一个物理分区，通过不同子卷区分。

先查看磁盘和分区：

```sh
lsblk
fdisk -l
```

SATA/SCSI 磁盘示例：

```sh
cfdisk /dev/sda
```

NVMe 硬盘示例：

```sh
cfdisk /dev/nvme0n1
```

虚拟机 VirtIO 磁盘示例：

```sh
cfdisk /dev/vda
```

在 `cfdisk` 中通常需要创建：

- 第 1 个分区：EFI system，用于挂载到 `/boot`
- 第 2 个分区：Linux swap，用于 Swap
- 第 3 个分区：Linux filesystem，用于 Btrfs 系统分区

本文后续统一按下面的示例分区编号说明：

| 磁盘类型 | EFI 分区 | Swap 分区 | Linux system 分区 |
| --- | --- | --- | --- |
| SATA/SCSI | `/dev/sda1` | `/dev/sda2` | `/dev/sda3` |
| NVMe | `/dev/nvme0n1p1` | `/dev/nvme0n1p2` | `/dev/nvme0n1p3` |
| VirtIO | `/dev/vda1` | `/dev/vda2` | `/dev/vda3` |

三种命名方式是同样的分区逻辑，只是设备名称不同：

- `/dev/sdaN` 常见于 SATA、SCSI、USB 磁盘
- `/dev/nvme0n1pN` 常见于 NVMe 硬盘
- `/dev/vdaN` 常见于 KVM/QEMU 等虚拟机的 VirtIO 磁盘

双系统场景下，EFI 分区可复用已有分区；全新安装时才按需创建并格式化。

写入分区表后再次检查：

```sh
lsblk
fdisk -l
```

## 7. 格式化与创建 Btrfs 子卷

以下命令按第 6 节的示例分区编写：

- EFI：`/dev/sda1`、`/dev/nvme0n1p1`、`/dev/vda1`
- Swap：`/dev/sda2`、`/dev/nvme0n1p2`、`/dev/vda2`
- Linux system：`/dev/sda3`、`/dev/nvme0n1p3`、`/dev/vda3`

如果实际分区编号不同，必须先根据 `lsblk` 输出确认后再调整命令。

### 7.1 EFI 分区

全新安装且确认目标 EFI 分区可格式化时：

SATA/SCSI：

```sh
mkfs.fat -F32 /dev/sda1
```

NVMe：

```sh
mkfs.fat -F32 /dev/nvme0n1p1
```

VirtIO：

```sh
mkfs.fat -F32 /dev/vda1
```

双系统场景下，如果 EFI 分区已由 Windows 使用，通常不要格式化。

### 7.2 Swap 分区

SATA/SCSI：

```sh
mkswap /dev/sda2
```

NVMe：

```sh
mkswap /dev/nvme0n1p2
```

VirtIO：

```sh
mkswap /dev/vda2
```

### 7.3 Btrfs 分区

SATA/SCSI：

```sh
mkfs.btrfs -L myArch /dev/sda3
```

NVMe：

```sh
mkfs.btrfs -L myArch /dev/nvme0n1p3
```

VirtIO：

```sh
mkfs.btrfs -L myArch /dev/vda3
```

临时挂载 Btrfs 分区：

SATA/SCSI：

```sh
mount -t btrfs -o compress=zstd /dev/sda3 /mnt
df -h
```

NVMe：

```sh
mount -t btrfs -o compress=zstd /dev/nvme0n1p3 /mnt
df -h
```

VirtIO：

```sh
mount -t btrfs -o compress=zstd /dev/vda3 /mnt
df -h
```

创建子卷：

```sh
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume list -p /mnt
umount /mnt
```

`@` 和 `@home` 是常见布局，也方便后续使用 Timeshift 等快照工具。

## 8. 挂载分区

挂载顺序要从根目录开始。

SATA/SCSI 磁盘示例：

```sh
mount -t btrfs -o subvol=/@,compress=zstd /dev/sda3 /mnt
mkdir /mnt/home
mount -t btrfs -o subvol=/@home,compress=zstd /dev/sda3 /mnt/home
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
```

NVMe 磁盘示例：

```sh
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvme0n1p3 /mnt
mkdir /mnt/home
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvme0n1p3 /mnt/home
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
swapon /dev/nvme0n1p2
```

VirtIO 磁盘示例：

```sh
mount -t btrfs -o subvol=/@,compress=zstd /dev/vda3 /mnt
mkdir /mnt/home
mount -t btrfs -o subvol=/@home,compress=zstd /dev/vda3 /mnt/home
mkdir -p /mnt/boot
mount /dev/vda1 /mnt/boot
swapon /dev/vda2
```

检查挂载和 Swap：

```sh
df -h
free -h
```

## 9. 安装基础系统

安装基础包、内核、固件和 Btrfs 工具：

```sh
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs
```

如果遇到 GPG 证书相关错误，可先更新安装环境中的 keyring：

```sh
pacman -S archlinux-keyring
```

安装基础工具：

```sh
pacstrap /mnt networkmanager vim sudo zsh zsh-completions
```

如果习惯使用 Bash，可把 `zsh zsh-completions` 换成 `bash-completion`。

## 10. 生成 fstab

```sh
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab
```

确认 `/`、`/home`、`/boot` 和 Swap 都已写入，且 UUID、文件系统类型、挂载选项符合预期。

## 11. 进入新系统

```sh
arch-chroot /mnt
```

进入后，`/mnt` 中的新系统会成为当前命令行环境的 `/`。

## 12. 设置主机名、hosts 与时区

编辑主机名：

```sh
vim /etc/hostname
```

示例内容：

```text
arch
```

编辑 hosts：

```sh
vim /etc/hosts
```

示例内容：

```text
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1        localhost
::1              localhost
127.0.0.1        arch.localdomain arch
```

设置时区：

```sh
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

## 13. 同步硬件时间

```sh
hwclock --systohc
```

## 14. 设置 Locale

编辑 `/etc/locale.gen`：

```sh
vim /etc/locale.gen
```

取消以下两行的注释：

```text
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

生成 locale：

```sh
locale-gen
```

设置系统语言：

```sh
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

不建议在这里设置中文 locale，否则 TTY 可能出现乱码。

## 15. 设置 root 密码

```sh
passwd root
```

输入密码时终端不会显示字符，这是正常行为。

## 16. 安装 CPU 微码

Intel CPU：

```sh
pacman -S intel-ucode
```

AMD CPU：

```sh
pacman -S amd-ucode
```

只安装与当前 CPU 平台匹配的微码包即可。

## 17. 安装 GRUB 引导

安装 GRUB 相关包：

```sh
pacman -S grub efibootmgr os-prober
```

安装到 EFI 分区：

```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH
```

编辑 GRUB 默认配置：

```sh
vim /etc/default/grub
```

建议调整：

```conf
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=5 nowatchdog"
GRUB_DISABLE_OS_PROBER=false
```

说明：

- 移除 `quiet`，便于启动时观察日志
- 将 `loglevel` 调整为 `5`，方便排错
- 加入 `nowatchdog`，可改善部分机器关机或重启速度
- 双系统需要 `GRUB_DISABLE_OS_PROBER=false` 才能让 GRUB 检测其他系统

如果 `nowatchdog` 对 Intel 看门狗无效，可考虑改用：

```conf
modprobe.blacklist=iTCO_wdt
```

==生成 GRUB 配置==：

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

如果主板无法显示自定义 UEFI 启动项，可在确认需要时使用 fallback 路径安装：

```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH --removable
```

正常机器不需要执行 `--removable` 版本。

## 18. 完成安装并重启

退出 chroot，卸载分区并重启：

```sh
exit
umount -R /mnt
reboot
```

实体机重启前拔掉安装 U 盘，避免再次进入安装环境。虚拟机可直接重启。

## 19. 首次进入系统后的网络配置

使用 root 登录后，启用 NetworkManager：

```sh
systemctl enable --now NetworkManager
ping www.bilibili.com
```

无线网络可用 `nmcli`：

```sh
nmcli dev wifi list
nmcli dev wifi connect "Wi-Fi名（SSID）" password "网络密码"
```

也可以使用交互式工具：

```sh
nmtui
```

## 20. 更新系统

如果基础安装完成后间隔了一段时间，或者刚进入新系统准备继续配置，先更新整个系统：

```sh
pacman -Syu
```

如果更新了内核、systemd、显卡相关包或关键基础包，建议重启后再继续后续配置。

## 21. 配置 root 默认编辑器

Arch Linux 的部分终端工具默认可能调用 `vi`。如果习惯使用 `vim`，可以给 root 设置默认编辑器。

编辑 root 的登录配置：

```sh
vim ~/.bash_profile
```

加入：

```sh
export EDITOR='vim'
```

保存后重新登录 root，或手动执行：

```sh
source ~/.bash_profile
```

也可以在需要指定编辑器的命令前临时写环境变量，例如后面的 `visudo`。

## 22. 创建普通用户并配置 sudo

日常使用不建议长期登录 root。创建普通用户，并加入 `wheel` 组以便后续通过 `sudo` 临时提权。

以下示例用户名为 `myusername`，实际使用时替换成自己的用户名：

```sh
useradd -m -G wheel -s /bin/bash myusername
passwd myusername
```

参数说明：

- `-m`：创建用户时同时创建家目录
- `-G wheel`：把用户加入 `wheel` 附加组
- `-s /bin/bash`：指定默认 shell 为 Bash

如果前面安装系统时选择了 Zsh，也可以改为：

```sh
useradd -m -G wheel -s /bin/zsh myusername
passwd myusername
```

使用 `visudo` 编辑 sudoers：

```sh
EDITOR=vim visudo
```

找到下面这一行：

```sudoers
# %wheel ALL=(ALL:ALL) ALL
```

取消注释，改为：

```sudoers
%wheel ALL=(ALL:ALL) ALL
```

保存退出后，`wheel` 组用户即可使用 `sudo`。建议切换到新用户验证：

```sh
su - myusername
sudo pacman -Syu
```

## 23. 开启 multilib 与 archlinuxcn 源

`multilib` 用于启用 32 位库支持，一些游戏、Wine 或闭源软件会用到。`archlinuxcn` 是 Arch Linux 中文社区仓库，可按需添加。

编辑 pacman 配置：

```sh
sudo vim /etc/pacman.conf
```

找到 `[multilib]` 段，取消这两行的注释：

```conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

如果需要添加 `archlinuxcn`，在文件末尾加入一个镜像源即可。下面列出几个常用镜像，实际保留其中一个或多个都可以：

```conf
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch
Server = https://repo.huaweicloud.com/archlinuxcn/$arch
```

刷新软件包数据库并更新：

```sh
sudo pacman -Syyu
```

安装 archlinuxcn keyring：

```sh
sudo pacman -S archlinuxcn-keyring
```

如果安装 `archlinuxcn-keyring` 时遇到 `farseerfc` key 信任不足的问题，可以先本地信任该 key：

```sh
sudo pacman-key --lsign-key "farseerfc@archlinux.org"
sudo pacman -S archlinuxcn-keyring
```

需要安装 AUR 助手时，可在 archlinuxcn 源可用后安装 `yay`：

```sh
sudo pacman -S yay
```

到这里为止，系统已经完成进入桌面环境前的基础用户、sudo、软件源和更新配置。后续可以继续选择安装 KDE Plasma、GNOME、niri 或其他桌面/窗口管理器。

## 24. 可选：安装 fastfetch

```sh
pacman -S fastfetch
fastfetch
```

## 25. 关机命令

```sh
shutdown -h now
```

或：

```sh
poweroff
```

## 安装检查清单

- [ ] 已确认从 UEFI 模式启动
- [ ] 网络可用，`ping` 测试正常
- [ ] 系统时间已同步
- [ ] mirrorlist 已调整
- [ ] 已确认目标磁盘和分区
- [ ] EFI、Swap、Btrfs 分区处理正确
- [ ] 已创建 `@` 和 `@home` Btrfs 子卷
- [ ] `/`、`/home`、`/boot`、Swap 已正确挂载
- [ ] `pacstrap` 基础系统和必要工具安装完成
- [ ] `/etc/fstab` 已生成并检查
- [ ] 主机名、hosts、时区、硬件时间已设置
- [ ] locale 已生成，`LANG=en_US.UTF-8`
- [ ] root 密码已设置
- [ ] CPU 微码已安装
- [ ] GRUB 已安装并生成配置
- [ ] 重启后 NetworkManager 已启用
- [ ] 系统已通过 `pacman -Syu` 更新
- [ ] 已创建普通用户并加入 `wheel` 组
- [ ] `wheel` 组 sudo 权限已启用
- [ ] 已按需开启 `multilib` 和 `archlinuxcn`
