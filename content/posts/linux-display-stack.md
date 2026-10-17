---
title: "Linux 显示协议与图形栈"
summary: "从 X11 到 Wayland：显示协议、合成器、wlroots 与 XWayland 各自扮演什么角色，游戏与安卓容器为何离不开它们"
date: 2026-10-14T00:00:00.000Z
categories: [Infra, Linux]
tags: [linux, wayland, graphics]
---
## 概述

- 参考文档：[Arch Wiki: Wayland](https://wiki.archlinux.org/title/Wayland)
- `Linux` 桌面把"画面如何到屏幕"拆成了两层：**显示协议**定义应用与显示服务端如何对话，**图形栈**负责渲染与上屏：

  ```mermaid
  flowchart TD
      A["应用(浏览器/游戏/Android 容器...)"] -->|"Wayland 协议 / X11 协议"| B["合成器(显示服务端：mutter / KWin / Hyprland / sway ...)"]
      B -->|"EGL / Vulkan"| C["Mesa / NVIDIA 驱动"]
      C -->|"DRM/KMS"| D["屏幕"]
  ```

- `X11` 是服役近四十年的老协议，`Wayland` 是现代替代者；如今主流发行版默认会话都是 `Wayland`，但两者仍长期共存

## `X11` 与 `Wayland`

- **`X11`**(`X Window System`)：
  - 客户端/服务端架构，`X Server` 集中管理输入与输出，应用通过套接字与之通信
  - 历史包袱：协议本身不支持的安全模型(任意应用可截获其他应用的输入与画面)、撕裂、HiDPI 与多显示器支持不力，都是 `Wayland` 出现的直接原因
- **`Wayland`**：
  - 把"显示服务端"与"合成器"合为一体：每个 `Wayland` 合成器就是显示服务器，应用把各自渲染好的缓冲区提交给合成器，由合成器合成上屏
  - 安全性更好(应用只能看到自己的内容)、帧调度由合成器统一控制(撕裂可消除)、多显示器不同刷新率/缩放是协议一等公民
  - 代价是大量原本 `X Server` 全局提供的功能(截屏、屏幕录制、输入法、颜色管理等)需要逐个以协议扩展的形式补齐，这也是迁移期阵痛的来源

## 合成器(`Compositor`)

- **合成器**是 `Wayland` 协议的服务端程序，负责把各应用的画面缓冲合成到屏幕上，同时处理窗口管理、输入分发、多显示器布局
- 生态分布：
  - 桌面环境自带：`GNOME` 的 `mutter`、`KDE` 的 `KWin`
  - 平铺式窗口管理器：`Hyprland`(动画与装饰)、`sway`(`i3` 的 `Wayland` 传承)
  - 特殊用途：`cage`(kiosk 单应用全屏)、`gamescope`(`Valve` 的嵌套游戏合成器)
- 玩游戏最关心的两个能力：
  - **`VRR`/可变刷新率**(`FreeSync`/`G-Sync`)：`mutter`、`KWin` 均已支持，撕裂与卡顿感显著降低
  - **直通扫描(`direct scan-out`)**：全屏独占时跳过合成直接上屏，降低延迟，但合成器介入的帧(如叠加层)会退回合成路径
- `HDR`、多显示器不同刷新率等长期痛点在 `Wayland` 合成器上正逐步补齐，体验逐年变好
- **嵌套合成器**：合成器本身也可以作为另一个合成器的客户端运行(会话里套会话)，`gamescope` 与 `cage` 都以此方式嵌在桌面里单独服务某个游戏或容器

## `wlroots`

- **`wlroots`** 是一个实现 `Wayland` 合成器通用逻辑的**库**，可以理解为"造合成器的轮子库"：
  - 封装了 `KMS`/`DRM`(显示输出)、`EGL`/`Vulkan` 渲染、输入设备、`XWayland` 兼容层等脏活，让 `sway`、`Hyprland`、`cage` 这类独立合成器不必从零实现协议细节
  - 主流桌面(`GNOME`/`KDE`)的合成器**不**基于 `wlroots`，各自维护协议实现；`wlroots` 主要服务于轻量/平铺/特殊用途合成器生态

## `XWayland`

- **`XWayland`** 是运行在 `Wayland` 合成器内部的 `X11` 兼容服务：`X11` 应用照常连"X Server"，而它把绘制结果作为普通 `Wayland` 客户端提交合成
- 它是过渡期不可或缺的胶水层：大量游戏(尤其 `Wine` 下仅支持 `X11` 全屏的老游戏)与遗留应用靠它活在 `Wayland` 会话里
- 代价：`XWayland` 窗口走"应用渲染 → `X` 协议 → 合成"两跳，无法用到 `Wayland` 的部分原生能力(如无撕裂的帧调度)，`wlroots` 系合成器的 `XWayland` 支持质量直接影响这类应用的兼容性

## 相关阅读

- [在 Linux 上玩游戏](/2026/10/14/linux-gaming/)：`gamescope` 如何用嵌套合成器给游戏做缩放与帧率控制
- [在 Linux 上玩安卓游戏](/2026/10/14/android-gaming-linux/)：`Waydroid` 如何作为 `Wayland` 客户端直通显示、`cage` 如何给它当 kiosk 画布
