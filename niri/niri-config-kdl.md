# niri KDL 配置文件整理

本文记录 niri 的 `config.kdl` 配置方式、KDL 基础语法、推荐拆分方案、格式化工具和日常维护流程。

参考资料：

- https://niri-wm.github.io/niri/Configuration:-Introduction
- https://github.com/niri-wm/niri/wiki/Configuration:-Include
- https://kdl.dev/

## 1. 配置文件位置

niri 默认读取：

```text
~/.config/niri/config.kdl
```

如果 `$XDG_CONFIG_HOME` 已设置，则优先使用：

```text
$XDG_CONFIG_HOME/niri/config.kdl
```

如果用户配置不存在，niri 会回退到：

```text
/etc/niri/config.kdl
```

第一次启动时，niri 通常会根据内置默认配置生成用户配置。建议从默认配置开始改，不要从空文件开始写。

也可以临时指定配置文件：

```sh
niri --config /path/to/config.kdl
```

或：

```sh
NIRI_CONFIG=/path/to/config.kdl niri
```

`--config` 的优先级高于 `$NIRI_CONFIG`。

## 2. 修改与校验

niri 支持配置热重载。保存 `config.kdl` 后，大多数设置会自动应用，包括快捷键、输出、窗口规则等。

修改前建议先备份：

```sh
cp -a ~/.config/niri ~/.config/niri.bak.$(date +%Y%m%d-%H%M%S)
```

校验配置：

```sh
niri validate
```

如果想测试某个临时配置文件：

```sh
niri validate --config /path/to/config.kdl
```

如果配置写错，niri 通常不会直接崩溃，而是保留上一个可用配置并提示错误。排错时查看用户日志：

```sh
journalctl --user -u niri -n 100
```

## 3. KDL 基础语法

niri 配置使用 KDL。它看起来像带层级的命令：

```kdl
output "eDP-1" {
    mode "2560x1600@165"
    scale 1.5
}
```

常见结构：

```kdl
节点名 参数 属性=值 {
    子节点 参数
}
```

注释使用 `//`：

```kdl
// 这是一行注释
```

使用 `/-` 可以注释掉整个节点：

```kdl
/-output "HDMI-A-1" {
    off
}
```

布尔开关在 niri 里经常表现为 flag。写出来表示开启，注释或删除表示关闭：

```kdl
input {
    focus-follows-mouse
}
```

部分 flag 也支持显式关闭：

```kdl
prefer-no-csd false
```

## 4. include 拆分规则

niri 25.11 起支持 `include`。`include` 只能写在顶层，不能写进 `layout {}`、`input {}` 等内部区块。

正确：

```kdl
include "includes/input.kdl"
include "includes/binds.kdl"
```

错误：

```kdl
layout {
    include "includes/layout-border.kdl"
}
```

include 路径可以是：

```kdl
include "includes/binds.kdl"
include "./includes/binds.kdl"
include "/home/user/.config/niri/includes/binds.kdl"
```

niri 26.04 起也支持 `~/`：

```kdl
include "~/dotfiles/niri/includes/binds.kdl"
```

include 是有顺序的。后面的配置会覆盖前面已经设置的同类选项：

```kdl
include "includes/layout.kdl"

layout {
    gaps 8
}
```

如果 `layout.kdl` 里也设置了 `gaps`，最后生效的是 `config.kdl` 里后写的 `gaps 8`。

可选 include：

```kdl
include optional=true "local.kdl"
```

`optional=true` 只表示文件不存在时不报错。如果文件存在但语法错误，仍然会导致配置校验失败。

## 5. 推荐拆分方案

推荐把 `config.kdl` 保持为入口文件，只负责 include 顺序和少量全局说明。

目录结构：

```text
~/.config/niri/
├── config.kdl
├── includes/
│   ├── environment.kdl
│   ├── input.kdl
│   ├── outputs.kdl
│   ├── layout.kdl
│   ├── binds.kdl
│   ├── autostart.kdl
│   ├── window-rules.kdl
│   ├── layer-rules.kdl
│   ├── animations.kdl
│   └── gestures.kdl
├── dms/
│   ├── colors.kdl
│   ├── layout.kdl
│   ├── alttab.kdl
│   └── binds.kdl
└── local.kdl
```

入口 `config.kdl` 示例：

```kdl
// ~/.config/niri/config.kdl

include "includes/environment.kdl"
include "includes/input.kdl"
include "includes/outputs.kdl"
include "includes/layout.kdl"
include "includes/animations.kdl"
include "includes/gestures.kdl"
include "includes/window-rules.kdl"
include "includes/layer-rules.kdl"
include "includes/autostart.kdl"
include "includes/binds.kdl"

// DMS 生成或维护的配置片段。使用 DMS 时再保留。
include optional=true "dms/colors.kdl"
include optional=true "dms/layout.kdl"
include optional=true "dms/alttab.kdl"
include optional=true "dms/binds.kdl"

// 本机私有配置，建议不提交到 Git。
include optional=true "local.kdl"
```

这个顺序的思路：

- 环境变量放最前面，方便输入法、Toolkit、代理等设置统一生效
- 输入设备、输出设备、布局等基础行为先加载
- 窗口规则和 layer 规则在快捷键前加载
- `binds.kdl` 靠后，方便覆盖前面或 DMS 中的快捷键
- `local.kdl` 最后加载，用于每台机器的差异配置

## 6. 各文件职责

`environment.kdl`：

```kdl
environment {
    XMODIFIERS "@im=fcitx"
}
```

`input.kdl`：

```kdl
input {
    keyboard {
        xkb {
            layout "us"
        }
        numlock
    }

    touchpad {
        tap
        natural-scroll
    }
}
```

`outputs.kdl`：

```kdl
output "eDP-1" {
    mode "2560x1600@165"
    scale 1.5
}
```

`layout.kdl`：

```kdl
layout {
    gaps 8

    border {
        on
        width 2
        active-color "#8aadf4"
        inactive-color "#494d64"
    }
}
```

注意：把 `border` 拆到 include 文件时，建议显式写 `on`。仅写空的 `border {}` 在 include 文件中不会等同于开启边框。

`autostart.kdl`：

```kdl
spawn-at-startup "fcitx5"
spawn-at-startup "xwayland-satellite"
```

如果已经通过 systemd 用户服务启动 DMS，不要再写：

```kdl
spawn-at-startup "dms" "run"
```

`window-rules.kdl`：

```kdl
window-rule {
    match app-id="firefox$"
    open-maximized true
}
```

`binds.kdl`：

```kdl
binds {
    Mod+Return { spawn "alacritty"; }
    Mod+Q { close-window; }
}
```

## 7. 拆分时的注意事项

多数配置区块会按 include 顺序合并，只覆盖写到的字段。适合拆分的内容：

- `layout`
- `overview`
- `environment`
- `binds`
- `window-rule`
- `layer-rule`
- `output`

不要把同一个普通 section 分散成多个重复块，除非 niri 文档明确支持合并。比较稳妥的做法是：

- `input` 只放在 `input.kdl`
- `layout` 只放在 `layout.kdl`
- `animations` 只放在 `animations.kdl`
- `binds` 只放在 `binds.kdl`

以下类型要特别注意：

- `binds`：相同快捷键以后面的定义为准
- `window-rule`：按 include 位置插入，顺序会影响匹配和覆盖
- `output`：每个输出名只能配置一次，不要在多个文件里重复写同一个 `output "eDP-1"`
- `struts`、`preset-column-widths`、`animations` 里的部分子区块、`input` 的设备子区块：有些不是深度合并，拆分时容易互相覆盖

## 8. 格式化工具

KDL 可以用 `kdlfmt` 格式化。推荐通过 Cargo 安装：

```sh
cargo install kdlfmt
```

格式化单个文件：

```sh
kdlfmt format ~/.config/niri/config.kdl
```

格式化整个 niri 配置目录：

```sh
kdlfmt format ~/.config/niri
```

如果使用 npm，也可以：

```sh
npm install -g kdlfmt
kdlfmt format ~/.config/niri
```

格式化前先校验：

```sh
niri validate
```

格式化后再次校验：

```sh
niri validate
```

如果配置里有大量手写注释，首次使用格式化工具前建议先备份或用 Git 管理，避免格式化结果不符合个人习惯。

## 9. 编辑器支持

建议给编辑器安装 KDL 语法支持：

- VS Code / VSCodium：安装 `KDL` 扩展
- Neovim：使用 Treesitter 的 KDL parser
- Helix：确认当前版本是否内置 KDL 语法

如果需要 LSP 诊断，可以安装 `kdl-lsp`：

```sh
cargo install kdl-lsp
```

但 niri 语义级校验仍以 `niri validate` 为准。KDL LSP 只能检查通用 KDL 语法，不一定知道 niri 的配置项是否存在、参数是否正确。

## 10. Git 管理建议

建议提交：

```text
config.kdl
includes/*.kdl
```

建议忽略：

```text
local.kdl
*.bak.*
```

如果直接把 `~/.config/niri` 做成 dotfiles 仓库，可以写 `.gitignore`：

```gitignore
local.kdl
*.bak.*
```

多机器配置建议：

```kdl
include optional=true "local.kdl"
```

每台机器单独维护自己的 `local.kdl`，例如：

```kdl
output "eDP-1" {
    scale 1.5
}
```

这样主配置可以共享，屏幕、键盘、启动项等机器差异放在本地文件。

## 11. 维护流程

日常修改建议按这个顺序：

```sh
cd ~/.config/niri
niri validate
```

编辑对应拆分文件后：

```sh
niri validate
```

确认无误后格式化：

```sh
kdlfmt format ~/.config/niri
niri validate
```

如果使用 Git 管理：

```sh
git diff
git add config.kdl includes
git commit -m "更新 niri 配置"
```
