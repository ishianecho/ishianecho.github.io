---
title: "GNOME 桌面"
summary: "介绍 GNOME Shell 与 Mutter 的组成、概览与工作区工作流、gsettings 与 dconf 配置体系、扩展生态及 Wayland 下的注意事项"
date: 2026-10-03
categories: [Infra, Linux]
tags: [linux, gnome, desktop]
---

## 概述

- 参考文档：
  - [GNOME Help](https://help.gnome.org/users/gnome-help/stable/)
  - [GNOME Developer Center](https://developer.gnome.org/)
- GNOME 是完整桌面环境，界面层是 GNOME Shell，应用基于 GTK4 与 libadwaita，显示管理器是 GDM
- **默认 Wayland 会话**，合成器是 Mutter，X11 会话仍可从登录界面选
- 设计哲学是"少选项、给默认"，深度定制靠 gsettings 与扩展，与[KDE Plasma 桌面](/2026/09/30/kde-plasma/)的思路相反
- 栈结构：

  ```mermaid
  flowchart LR
    A["GNOME Shell(顶栏/概览/通知)"] --> B["Mutter(合成器+窗口管理)"]
    B --> C["Wayland"]
    B --> D["XWayland(跑 X11 应用)"]
    E["核心应用(Files/Terminal/...)"] --> F["GTK4 + libadwaita"]
  ```

## 快速开始

- 安装：Arch 系`pacman -S gnome gnome-tweaks`，Debian 系`apt install gnome`，Fedora Workstation 默认就是 GNOME
- 启用显示管理器：`systemctl enable --now gdm`
- 上手路径：`Super`进概览 → 输入应用名搜索启动 → `Super+↑/↓`在工作区间搬窗口 → 顶栏右上角是快速设置

## 工作流

- 概览(Overview)：`Super`或左上热角触发，一屏看全窗口、工作区与应用搜索
- 工作区动态增减：窗口拖到屏幕边缘自动开新工作区，`Super+PgUp/PgDn`切换，`Super+Shift+PgUp/PgDn`把当前窗口搬过去
- 窗口管理：
  - `Super+↑`最大化，`Super+↓`还原，`Super+←/→`半屏平铺
  - `Alt+Tab`跨应用切换，`Alt+\``同应用的多窗口间切换
  - `Super+D`显示桌面
- 顶栏右上是快速设置(Quick Settings)：网络、蓝牙、音量、亮度、暗色模式与电源一排胶囊按钮
- 通知集中在顶栏与锁屏，勿扰模式在快速设置里
- 传统托盘图标默认不显示，需要扩展，见扩展一节

## 核心应用

- Files(Nautilus)文件管理器：
  - 多标签、`F2`重命名(含批量模板)、列表与网格视图切换
  - 远程与移动设备经 GVfs 挂载：地址栏输`sftp://host/path`即可
- Terminal：GNOME Terminal 或新版 Ptyxis，`Ctrl+Shift+T`新标签，配色方案在首选项管理
- Text Editor 与 Evince(PDF)、Loupe(图片)组成日常阅读三件套
- Software 应用商店：系统包与 Flatpak 都从这里装
- Settings(`gnome-control-center`)：分模块的设置中心，搜索框直达任意开关

## 配置体系

- 配置三层：`gsettings`命令行 → dconf 二进制存储 → `dconf-editor`图形浏览，**改的都是同一份数据**
- 常用操作：

  ```console
  $ gsettings get org.gnome.desktop.interface color-scheme                    # 读
  $ gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'      # 写
  $ gsettings list-schemas | rg keyboard                                      # 找 schema
  $ gsettings list-recursively org.gnome.desktop.peripherals                  # 全量看
  $ gsettings reset org.gnome.desktop.interface color-scheme                  # 恢复默认
  ```

- 值是 GVariant 语法：字符串带引号，空字符串数组写`@as []`，元组写`(1, 2)`
- `gnome-tweaks`补上官方设置不给的项：字体细节、启动应用、中键粘贴
- 改键位主要动两个 schema：`org.gnome.desktop.wm.keybindings`窗口管理类，`org.gnome.settings-daemon.plugins.media-keys`系统类
- 自定义启动脚本放`~/.config/autostart/`的`.desktop`文件，或用 systemd 用户单元

## 扩展

- 扩展是给 GNOME Shell 打补丁的 JS 插件，仓库在[extensions.gnome.org](https://extensions.gnome.org)，本地用`gnome-extensions`管理：

  ```console
  $ gnome-extensions list
  $ gnome-extensions enable dash-to-dock@micxgx.gmail.com
  $ gnome-extensions disable xxx
  ```

- 装机率最高的几个：
  - Dash to Dock：把左侧 Dash 变成常驻任务栏
  - AppIndicator and KStatusNotifierItem Support：找回传统托盘图标，输入法与通讯软件刚需
  - Just Perfection：界面元素微调，隐藏顶栏等
- 管理 GUI 用 Extension Manager(Flatpak)或浏览器插件配`gnome-browser-connector`
- **GNOME 大版本升级常炸扩展**：升级前把非关键扩展禁用，桌面卡死时切到 TTY 用`gnome-extensions disable`救急

## Wayland 注意

- 屏幕共享、远程桌面与分数缩放依赖`xdg-desktop-portal-gnome`与 PipeWire，缺包时共享整屏只会黑一块
- X11 应用走 XWayland，混排缩放发糊，原理见[Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)
- 全局热键只能由合成器与遵循 Wayland 协议的应用注册，老式 X11 热键工具失效
- 输入法另有一层坑，见[在 GNOME 中配置 fcitx5](/2026/10/06/gnome-fcitx5/)

## 相关阅读

- [KDE Plasma 桌面](/2026/09/30/kde-plasma/)：可定制性拉满的另一条路
- [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)：Wayland 与合成器的底层关系
