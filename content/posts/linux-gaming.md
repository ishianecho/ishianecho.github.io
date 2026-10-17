---
title: "在 Linux 上玩游戏"
summary: "梳理 Linux 游戏生态的全家桶：Steam、Wine、Proton 及其分支、umu、gamescope 与 Lutris"
date: 2026-10-14T00:00:00.000Z
categories: [Games]
tags: [linux, gaming]
---
## Overview

- 参考文档：[Arch Wiki — Gaming](https://wiki.archlinux.org/title/Gaming)
- 在 `Linux` 上跑原生为 `Windows` 打造的游戏，本质是一条逐层转译的链路：

  ```mermaid
  flowchart TD
      G["游戏(Windows API)"] -->|"Wine / Proton：系统调用与 API 转译"| T["转译后的调用"]
      T -->|"DXVK / VKD3D-Proton：D3D → Vulkan 图形转译"| V["Vulkan 调用"]
      V -->|"Mesa / NVIDIA 驱动"| M["内核图形栈"]
      M -->|"Wayland / X11 合成器"| S["屏幕"]
  ```

- 各组件分工不同：`Steam`/`Lutris` 是**发行与启动**层，`Wine`/`Proton` 是**兼容**层，`gamescope` 是**呈现**层，最底下的显示协议与合成器层(合成器、`wlroots`、`XWayland`)单独整理在 [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)，一层层叠起来才有了今天开箱即用的体验

## `Valve` 与 `Steam`

- **`Valve`** 是 `Steam` 平台的母公司，也是整个 `Linux` 游戏生态最大的推动者：
  - `2018` 年推出 **`Proton`**，让 `Steam` 上的 `Windows` 游戏"一键在 `Linux` 下运行"
  - `2022` 年发布 **`Steam Deck`** 掌机，搭载定制 `Linux` 发行版 **`SteamOS`**，直接把 `Proton` 生态推为主流
  - 雇佣了大量开发者贡献 `Mesa`、`Kernel`、`wine` 上游，`Linux` 图形栈近几年的飞跃与其商业利益直接相关
- **`Steam`** 客户端本身就是 `Linux` 原生应用，`Linux` 版与 `Windows` 版功能一致
  - 在设置 → 兼容性中开启"对所有其他标题使用 `Steam Play`"后，游戏默认通过 `Proton` 运行，无需手动配置
  - 游戏页面标注的"`Steam Deck` 已验证"可以作为 `Linux` 兼容性的粗略参考：`已验证` > `可玩` > `不支持`
  - 反作弊是最大的坑：部分内核级反作弊(`EAC`/`BattlEye`)需要开发商显式开启 `Proton` 支持，未开启的游戏无法运行，购买前先查 `ProtonDB` 的社区评分

## `Wine`

- **`Wine`**(`Wine Is Not an Emulator`)是整个兼容层的基石，历史比 `Proton` 早二十年：
  - 它通过重新实现 `Windows` 的系统调用与 `API`(`ntdll`、`user32`、`gdi32`...)，把 `Windows` 程序的请求转译成 `Linux` 上等价的操作，**不是模拟器**，几乎无 `CPU` 性能损耗
  - 覆盖面极广：`PE` 加载器、注册表(`wineprefix` 中的 `.reg` 文件)、`DirectX`、音频、字体、`OLE`/`COM`……一个 `Windows` 程序赖以生存的环境几乎都有对应实现
- **`wineprefix`** 是理解 `Wine` 的关键概念：
  - 一个 `prefix` 就是一个虚拟的 `C:` 盘(`drive_c` 目录)加一套注册表，默认位于 `~/.wine`，通过 `WINEPREFIX=xxx` 环境变量可以为每个游戏维护独立的隔离环境
  - 隔离的好处：`A` 游戏需要的运行库补丁、注册表魔改不会污染 `B` 游戏，重装系统也只需备份各游戏的 `prefix`
- `Wine` 的版本命名曾经历 `1.x` → `2.x` → ... → `9.x` 的稳定/开发双轨制，现在以年份命名(如 `Wine 10.x`)，每年发一个大版本

## `Proton` 及其分支

- **`Proton`** 是 `Valve` 基于 `Wine` 的下游发行版，可以理解为"`Wine` + 一堆游戏专用补丁 + 转译库"的整合包：
  - 上游 `Wine` 追求通用正确性，`Proton` 则激进地针对游戏打补丁，很多修复先在 `Proton` 落地、验证稳定后才合回上游
  - 自带 `DXVK`、`VKD3D-Proton` 等转译库(见下节)，并深度集成 `Steam` 的运行时(`SteamLinuxRuntime`)，解决各发行版库版本不一致的问题
- **`Proton-GE`**(GloriousEggroll 版)是最流行的社区分支：
  - 在官方 `Proton` 基础上合入大量尚未进官方版的补丁、更新的转译库与媒体编解码器，常能救活官方版跑不起来或表现不佳的游戏
  - 更新极快，某个游戏发售首日出现问题时，社区往往第一个在 `GE` 里给出绕过方案
- **`proton-tkg`**：另一条定制路线，提供大量可配置的编译选项(`futex`、`fsync`、调度器调优等)，面向愿意自己编译折腾的高级用户
- **`Wine-GE`** 与 **`umu-launcher`**：
  - `Wine-GE` 曾是为非 `Steam` 启动器(如 `Lutris`、`EA app`)准备的 `GE` 版 `Wine`，**现已停止维护**
  - 官方替代方案是 **`umu-launcher`**：它实现了一套与 `Steam` 相同的运行时环境，让任何启动器(甚至一条命令行)都能以"官方 `Proton` 或 `Proton-GE`"来运行非 `Steam` 游戏——相当于把 `Proton` 从 `Steam` 里解耦出来给全系统用
  - `Lutris` 已原生集成 `umu`，非 `Steam` 游戏因此也能享受 `Proton` 分支的持续更新

## 常见转译库

- 这些库是 `Proton` 生态的"轮子"，`Steam` 与 `Lutris` 会在安装运行时自动带上，通常无需手动安装，认识它们有助于看懂报错与社区讨论：
  - **`DXVK`**：把 `D3D9`/`D3D10`/`D3D11` 转译为 `Vulkan`，是老游戏与新游戏能在 `Linux` 上满速运行的头号功臣
  - **`VKD3D-Proton`**：把 `D3D12` 转译为 `Vulkan`，`Valve` 维护的 `vkd3d` 分支，`DX12` 游戏靠它运行
  - **`DXVK-NVAPI`**：转译 `NVIDIA` 的 `NVAPI` 接口，让 `DLSS` 等特性在 `Proton` 下可用
  - **`Wine-Mono` / `Wine-Gecko`**：`Wine` 内置的 `.NET` 与浏览器引擎替代品，部分游戏的启动器依赖它们
  - **`FAudio` / `WinePulse`**：`XAudio2` 等音频接口到 `PipeWire`/`PulseAudio` 的适配

## `gamescope`

- **`gamescope`** 是 `Valve` 开发的**嵌套式合成器**(`SteamOS`/`Steam Deck` 的会话层，合成器与 `Wayland`/`wlroots` 的背景见 [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/))，可以嵌在任意桌面里单独给某个游戏用：
  - 核心能力一：**分辨率与缩放管理**——游戏内部以任意分辨率渲染，`gamescope` 负责用高性能滤波器(`FSR`、`NIS` 等)缩放到屏幕，掌机与小屏场景的救星
  - 核心能力二：**像素完美的全屏控制**——模拟"全屏独占"，无论窗口管理器做什么，游戏看到的都是独占全屏，规避了各桌面合成器行为不一致的问题
  - 其他实用功能：帧率限制、色相/锐化滤镜、嵌套窗口运行、`HDR` 映射
  - 典型用法是作为启动包装器：`gamescope -W 2560 -H 1440 -w 1920 -h 1080 -F fsr -- %command%`，在 `Steam` 里替换原始启动命令即可

## `Lutris`

- **`Lutris`** 是 `Linux` 上的**开源游戏启动器/管理器**，定位是"`Steam` 之外一切游戏"的统一入口：
  - 聚合管理各种来源：`GOG`/`Epic`/`Battle.net` 等商店与启动器、手动安装的本地游戏、模拟器、原生 `Linux` 游戏，全部收纳进一个库
  - 核心机制是**安装脚本社区共建**：每个游戏在社区维护一份安装脚本，声明需要的 `Wine`/`Proton` 版本、要下载的依赖、注册表改动等，用户点一下"Install"即可复刻他人的成功配置
  - 运行器(`runner`)体系：`Wine`、`Proton`(经由 `umu`)、原生、各类模拟器都是可插拔的 `runner`，每个游戏可单独指定版本与参数
  - 与 `Steam` 的分工：`Steam` 管平台内游戏，`Lutris` 管"不在 `Steam` 里的一切"，两者互补而非竞争

## 选择建议

- 只玩 `Steam` 游戏：装好 `Steam` 开 `Proton` 即可，遇到问题换 `Proton-GE` 试试
- `Steam` 外的 `Windows` 游戏：`Lutris` + `umu`(Proton 运行时)，找社区安装脚本
- 性能/兼容性调优：`gamescope` 控制分辨率缩放与帧率，`MangoHud` 观察实时帧率与负载
- 买游戏前查 [ProtonDB](https://www.protondb.com) 的社区报告，避免踩反作弊的雷
