---
title: "KDE Plasma 桌面"
summary: "介绍 Plasma 6 的组成与 Wayland 默认栈，讲解安装、KRunner 与工作流、Dolphin 与 Konsole 等核心应用、KWin 窗口规则与主题定制，以及 kwriteconfig6 与配置文件体系"
date: 2026-09-30
categories: [Infra, Linux]
tags: [linux, kde, desktop]
---

## 概述

- 参考文档：
  - [KDE UserBase](https://userbase.kde.org)
  - [KDE Plasma Desktop](https://kde.org/plasma-desktop)
- Plasma 是 KDE 项目的桌面环境，应用全家桶叫 KDE Gear，底层是 Qt 与 KDE Frameworks
- **Plasma 6 起 Wayland 会话是默认**，X11 会话保留为选项，合成器与窗口管理器合一于 KWin
- 栈结构：

  ```mermaid
  flowchart LR
    A["Plasma Shell(面板/小组件)"] --> B["KWin(合成器+窗口管理)"]
    B --> C["Wayland"]
    B --> D["XWayland(跑 X11 应用)"]
    E["KDE Gear(Dolphin/Konsole/...)"] --> F["Qt6 + KDE Frameworks"]
  ```

- 设计取向：选项丰富、高度可定制，与[GNOME 桌面](/2026/10/03/gnome-desktop/)的"少即是多"正好是两个极端

## 快速开始

- 安装：Arch 系`pacman -S plasma sddm`，Debian 系`apt install kde-plasma-desktop sddm`，Fedora 直接用 KDE Spin，openSUSE 安装盘可选 KDE
- 启用显示管理器：`systemctl enable --now sddm`
- 上手路径：登录桌面 → `Alt+Space`呼出 KRunner → 输入应用名启动 → 右键桌面进入面板与小组件模式

## 工作流

- Kickoff 是左下角启动器，单击`Meta`键也能唤起，支持首字母搜索与分类浏览
- KRunner(`Alt+Space`)是全局启动器，不止启动应用：
  - 直接算算式与单位换算，杀进程，切换窗口，跳转到任意设置页
  - 插件可在`系统设置 → KRunner`里逐个开关
- 虚拟桌面与活动(Activities)：
  - 虚拟桌面是几套平行工作区，快捷键自定义
  - 活动是带独立壁纸、小组件与窗口集的"项目空间"，适合按项目隔离环境
- 窗口管理：
  - `Alt+Tab`切换窗口，`Alt+F3`打开窗口操作菜单
  - `Meta+方向键`快速平铺到半屏与四分屏
  - 单击`Meta`进入 Overview 总览全部窗口与桌面
- 面板与小组件：右键面板进入编辑模式，可拖动、悬浮、智能隐藏，小组件直接从部件选择器拖入

## 核心应用

- Dolphin 文件管理器：
  - `F3`开双栏、`F4`呼出底部内嵌终端、`Ctrl+F`搜索、多标签页
  - 多选后`F2`批量重命名，支持序号与捕获组模板
  - 远程位置(sftp、smb)通过 KIO 协议直接像本地目录一样访问
  - 服务菜单：`~/.local/share/kio/servicemenus/`下放`.desktop`文件给右键加动作
- Konsole 终端：配置方案(Profile)管理外观与启动命令，支持分屏，`yakuake`提供`F12`下拉式终端
- Spectacle 截图：`PrtSc`全屏，`Meta+Shift+PrtSc`矩形区域，可配置自动保存与复制
- Discover 应用商店：统一装系统包与 Flatpak，后端是 PackageKit
- 系统设置(System Settings)：几乎所有配置的入口，模块支持搜索直达

## KWin 与定制

- 窗口规则：按窗口类或标题匹配，固定大小、位置、置顶、跳过任务栏，入口在`系统设置 → 窗口管理 → 窗口规则`，`xprop`查窗口类(X11 下)
- 桌面特效：`系统设置 → 桌面特效`里开关动画、模糊、窗口物理感等
- 主题分层：全局主题一键换全套外观，其下再细分配色、图标、窗口装饰与 Plasma 样式，资源在[KDE Store](https://store.kde.org)
- 多显示器与缩放：`显示与监视器`按屏设置，Wayland 下分数缩放质量明显好于 X11

## 配置文件

- Plasma 的设置几乎全部落盘在`~/.config/`：
  - `plasma-org.kde.plasma.desktop-appletsrc`：面板与小组件布局
  - `kwinrc`：KWin 行为与特效，`plasmarc`：桌面主题
  - `baloofilerc`：文件索引 Baloo 的开关与范围
- 命令行读写配置：

  ```console
  $ kwriteconfig6 --file kwinrc --group Compositing --key Enabled false   # 关合成
  $ kreadconfig6 --file plasmarc --group Plasma --key theme               # 读主题
  $ qdbus org.kde.KWin /KWin reconfigure                                  # 免注销重载 KWin
  ```

- Baloo 文件索引：`balooctl status`看状态，`balooctl suspend`暂停，失控时`balooctl purge`清空重来
- **桌面抽风先怀疑配置**：把对应 rc 文件改名后注销重登，恢复了就逐项加回定位坏项

## Wayland 注意

- X11 应用经 XWayland 运行，混合缩放会发糊，原理见[Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)
- 屏幕共享与远程桌面走`xdg-desktop-portal`，OBS 等 Flatpak 应用要装对应后端
- 老式 X11 全局热键工具部分失效，热键交给 KWin 与应用自身注册

## 相关阅读

- [GNOME 桌面](/2026/10/03/gnome-desktop/)：另一极的设计哲学
- [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)：Wayland、合成器与 XWayland 的关系
