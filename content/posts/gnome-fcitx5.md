---
title: "在 GNOME 中配置 fcitx5"
summary: "分析 fcitx5 在 GNOME Wayland 下失灵的三类根源(环境变量未进会话、IBus 抢占、应用未加载 IM 模块)，给出安装、environment.d 配置、Flatpak 覆盖与 fcitx5-diagnose 排查清单"
date: 2026-10-06
categories: [Infra, Linux]
tags: [linux, gnome, fcitx5]
---

## 概述

- 参考文档：
  - [fcitx5 Wiki](https://fcitx-im.org/wiki/Fcitx5)
  - [fcitx5 Wiki: Using Fcitx 5 on Wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland)
  - [Arch Wiki: Fcitx5](https://wiki.archlinux.org/title/Fcitx5)
- fcitx5 是输入法框架，拼音等具体输入法是它的插件(`fcitx5-chinese-addons`)
- **GNOME 下失灵通常不是 fcitx5 坏了**，而是三件事：环境变量没进图形会话、GNOME 自带的 IBus 抢占输入、应用没加载 fcitx 的 IM 模块
- 数据流：

  ```mermaid
  flowchart LR
    A["GTK 应用"] -->|"GTK_IM_MODULE=fcitx"| B["fcitx5 GTK 模块"]
    C["Qt 应用"] -->|"QT_IM_MODULE=fcitx"| D["fcitx5 Qt 模块"]
    B & D -->|"D-Bus"| E["fcitx5 守护进程"]
    E --> F["拼音等引擎"]
  ```

- Mutter 未实现 Wayland 的 input-method 协议，所以靠各 GUI 工具包自己的 IM 模块绕开协议直连 fcitx5，环境变量因此成了生命线

## 快速开始

- 安装(CachyOS 与 Arch 系)：

  ```console
  $ pacman -S fcitx5-im fcitx5-chinese-addons
  ```

  - `fcitx5-im`组含`fcitx5`本体、`fcitx5-gtk`(GTK 3 与 4 模块)、`fcitx5-qt`与配置工具`fcitx5-configtool`
- 把环境变量写进`~/.config/environment.d/im.conf`：

  ```ini
  GTK_IM_MODULE=fcitx
  QT_IM_MODULE=fcitx
  XMODIFIERS=@im=fcitx
  ```

  - **Wayland 会话读的是 environment.d，`~/.bashrc`与`~/.xprofile`对 GNOME Wayland 不生效**，这是第一大坑
- 注销重登，`fcitx5-configtool`里把"Pinyin"从可用列表加到当前输入法列表
- 在`设置 → 键盘`里把输入源只留一个英文键盘，`Ctrl+Space`切换中英，输入由 fcitx5 接管

## GNOME 的坑

- IBus 抢占：
  - GNOME 把 IBus 深度嵌进 Shell，**输入源列表里一旦添加了中文输入法就轮不到 fcitx5**
  - 排查：输入源只保留`English (US)`或对应键盘布局，中文输入全部交给 fcitx5
- 托盘没有键盘图标：GNOME 默认无托盘，装 AppIndicator 扩展找回，或接受只用`fcitx5-configtool`管理
- 切换键冲突：`Ctrl+Space`若与 GNOME 或应用冲突，在`fcitx5-configtool → 全局选项`里换
- `im-config`与`imsettings`是 Debian 系的工具，Arch 系不存在，照搬旧教程会找不到北

## 应用侧适配

- Chromium 与 Electron 应用(VS Code、Discord 等)在 Wayland 下默认不接 fcitx5：
  - 启动参数加`--enable-wayland-ime`启用 Wayland 输入法支持
  - 或让该应用走 X11(`--ozone-platform=x11`)，靠`XMODIFIERS`工作
- Flatpak 应用在沙箱里看不到宿主的 fcitx5，需要放行：

  ```console
  $ flatpak override --user --env=GTK_IM_MODULE=fcitx \
      --env=QT_IM_MODULE=fcitx --env=XMODIFIERS=@im=fcitx \
      --filesystem=xdg-run/fcitx5 --talk-name=org.fcitx.Fcitx5
  ```

- 终端模拟器大多随 GTK 正常工作；Wayland 原生的自绘应用(kitty 等)按各自文档单独配

## 排查清单

- 万能第一步：`fcitx5-diagnose`，逐项列出环境变量、IM 模块注册与守护进程状态，照着红字修
- 验证变量真的进了会话：在图形会话的终端里`echo $GTK_IM_MODULE`，空的就是没被 systemd 用户会话读到
- 手动前台拉起看报错：`fcitx5 -r -d`重启守护进程
- 自启动：fcitx5 自带`/etc/xdg/autostart/org.fcitx.Fcitx5.desktop`，Wayland 会话一般无需手配
- 定位手段：登录界面切到`GNOME on X11`会话，**X11 下好使而 Wayland 下不行，基本锁定 IM 模块没加载或应用不支持**，再按应用侧一节处理

## 相关阅读

- [GNOME 桌面](/2026/10/03/gnome-desktop/)：Shell、扩展与 gsettings 体系
- [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)：为什么 Wayland 的输入法与 X11 不一样
