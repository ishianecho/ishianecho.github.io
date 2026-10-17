---
title: "Linux 发行版与包管理器"
summary: "讲解 /etc/os-release 的字段与读取方式，梳理 Debian、Red Hat、Arch、SUSE 等发行版家族，并对比 pacman、apt、dnf、zypper 等包管理器的常用用法"
date: 2026-09-08
categories: [Infra, Linux]
tags: [linux, cli]
---

## 概述

- 参考文档：[os-release Specification](https://www.freedesktop.org/software/systemd/man/latest/os-release.html)
- 发行版 = `Linux` 内核 + 一套用户空间 + 包仓库 + 默认配置的组合，同一个内核配上不同的包管理器就是不同的发行版
- `/etc/os-release` 是 freedesktop.org 定义的发行版标识文件，`systemd`、安装脚本与无数运维脚本都读它
- 家族与包管理器的对应：

  ```mermaid
  flowchart TD
    L["Linux 发行版"] --> A["Debian 系"]
    L --> B["Red Hat 系"]
    L --> C["Arch 系"]
    L --> D["SUSE 系"]
    L --> E["独立发行版"]
    A --> A1["apt / deb"]
    B --> B1["dnf / rpm"]
    C --> C1["pacman / zst"]
    D --> D1["zypper / rpm"]
    E --> E1["emerge nix apk xbps"]
  ```

## `os-release`

- 文件位于`/etc/os-release`，`/usr/lib/os-release`是发行版自带的兜底，`/etc`下的优先级更高
- 内容是合法的 shell 变量赋值，可以直接`source`后在脚本里引用：

  ```console
  $ . /etc/os-release && echo "$ID"
  ```

- 常用字段：
  - `ID`：小写发行版标识，如`arch`、`ubuntu`、`opensuse-tumbleweed`，脚本分支判断用它
  - `ID_LIKE`：父家族标识，`cachyos`的值是`arch`，判断软件兼容性时看它
  - `NAME`与`PRETTY_NAME`：发行版名称与带版本修饰的完整名称
  - `VERSION_ID`与`VERSION_CODENAME`：数字版本与代号，滚动发行版可能没有
  - `VARIANT`与`VARIANT_ID`：同一发行版的变体，如`Server`与`Desktop`
  - `HOME_URL`、`SUPPORT_URL`、`BUG_REPORT_URL`、`LOGO`：链接与图标名
- 查看方式：`cat /etc/os-release`，或`hostnamectl`输出里的`Operating System`行

## 发行版家族

- Debian 系，包格式`deb`：
  - `Debian`：稳定至上，仓库分`stable`、`testing`、`unstable(sid)`三档
  - `Ubuntu`及其风味版(`Kubuntu`、`Xubuntu`、`Lubuntu`)、`Linux Mint`、`Pop!_OS`、`elementary OS`
- Red Hat 系，包格式`rpm`：
  - `Fedora`：新技术的试验田，约半年一版
  - `RHEL`与重建版`Rocky Linux`、`AlmaLinux`，`CentOS Stream`位于`Fedora`与`RHEL`之间
- Arch 系，包格式`pkg.tar.zst`，滚动更新：
  - `Arch Linux`：`pacman`加`AUR`，wiki 质量极高
  - `CachyOS`：`ID_LIKE=arch`，提供`x86-64-v3/v4`优化仓库与定制内核，性能导向
  - `EndeavourOS`、`Manjaro`：降低安装与维护门槛
  - `Artix`：去掉`systemd`改用`runit`或`OpenRC`
  - `SteamOS 3`：基于`Arch`，系统分区只读
- SUSE 系，包格式`rpm`：
  - `openSUSE Tumbleweed`：滚动更新且经`openQA`自动化测试
  - `openSUSE Leap`：与`SLE`(SUSE Linux Enterprise)同源的稳定版
- 独立发行版：
  - `Gentoo`：源码包，按`USE`标志本地编译
  - `NixOS`：声明式配置，包互相隔离，整个系统可回滚
  - `Alpine`：`musl libc`加`busybox`，体积极小，容器常客
  - `Void Linux`：`runit`而非`systemd`，包管理用`xbps`
  - `Slackware`：仍在维护的最老发行版之一

## 包管理器用法

- `pacman`(Arch 系)：

  ```console
  $ pacman -S pkg            # 安装
  $ pacman -Syu              # 同步仓库并升级全系统，滚动发行版日常
  $ pacman -Ss 关键词         # 搜索仓库
  $ pacman -Si pkg           # 查看仓库包信息
  $ pacman -Rns pkg          # 卸载并清理依赖与配置
  $ pacman -Q                # 列出已安装的包
  $ pacman -Ql pkg           # 列出包内所有文件路径
  $ pacman -Qo /文件路径      # 查询文件属于哪个包
  $ pacman -Sc               # 清理包缓存
  ```

  - AUR 是用户仓库，用`yay`或`paru`等 wrapper 以同样语法安装
- `apt`(Debian 系)：

  ```console
  $ apt update               # 刷新索引，升级前必做
  $ apt upgrade              # 升级已装包
  $ apt full-upgrade         # 允许增删依赖的升级
  $ apt install pkg
  $ apt remove pkg           # 保留配置地卸载
  $ apt purge pkg            # 连配置一起删
  $ apt search 关键词
  $ apt show pkg
  $ apt autoremove           # 清理不再需要的依赖
  ```

  - `apt`是面向管理员的友好前端，底层是`apt-get`与`apt-cache`，脚本中官方推荐用后者更稳定的接口
- `dnf`(Fedora 与 RHEL 系)：

  ```console
  $ dnf install pkg
  $ dnf upgrade              # 全系统升级
  $ dnf remove pkg
  $ dnf search 关键词
  $ dnf info pkg
  $ dnf provides '*/libfoo.so'  # 查询文件属于哪个包，排依赖错误必备
  $ dnf history              # 事务历史
  $ dnf history rollback 12  # 回滚一次事务，rpm 系独有优势
  ```

  - Fedora 新版默认使用`dnf5`，命令语法兼容
- `zypper`(openSUSE)：

  ```console
  $ zypper refresh           # 刷新仓库
  $ zypper install pkg
  $ zypper remove pkg
  $ zypper search 关键词
  $ zypper info pkg
  $ zypper dup               # dist-upgrade，Tumbleweed 日常升级用它
  ```

- 其它家族速览：
  - `Gentoo`：`emerge --sync`同步仓库，`emerge -av 包名`装包并预览`USE`，`emerge --depclean`清理
  - `NixOS`：`nix profile install nixpkgs#包名`临时装包，正式配置写进`configuration.nix`后`nixos-rebuild switch`，`nix-env --rollback`整体回滚
  - `Alpine`：`apk add`、`apk del`、`apk search`、`apk upgrade`
  - `Void`：`xbps-install -Su`升级，`xbps-remove`卸载
- 跨发行版的通用层：
  - `Flatpak`：沙箱化桌面应用，主仓库为[Flathub](https://flathub.org)
  - `Snap`：`Ubuntu`力推，自带自动更新
  - 二者与发行版包管理器并行，不共享系统库，磁盘占用换隔离性

## 相关阅读

- [Linux 内核](/2025/08/20/linux-kernel/)：发行版打包的那个内核
- [Linux 文件系统](/2026/09/26/linux-filesystems/)：包文件落到磁盘后的组织方式
