---
title: "Wine 注册表"
summary: "讲解 Wine 注册表与 prefix 内 .reg 文件的对应关系与值类型编码，以及用 regedit 与 reg.exe 完成 Windows regedit 同等修改的方法与常见案例"
date: 2026-09-15
categories: [Infra, Linux]
tags: [linux, wine]
---

## 概述

- 参考文档：
  - [Wine Wiki: Registry](https://gitlab.winehq.org/wine/wine/-/wikis/Registry)
  - [Wine Wiki: Useful Registry Keys](https://gitlab.winehq.org/wine/wine/-/wikis/Useful-Registry-Keys)
- Wine 用平面文本文件模拟 Windows 注册表，键路径组织成与 Windows 相同的树，由`wineserver`统一读写
- 每个 prefix(由`WINEPREFIX`指定的独立 Windows 环境)有自己的一套注册表，互不干扰
- 文件与根键的对应：

  ```mermaid
  flowchart LR
    A["WINEPREFIX 目录"] --> B["system.reg"]
    A --> C["user.reg"]
    A --> D["userdef.reg"]
    B -->|"对应"| E["HKEY_LOCAL_MACHINE"]
    C -->|"对应"| F["HKEY_CURRENT_USER"]
    D -->|"对应"| G["HKEY_USERS 下默认用户"]
  ```

- **Wine 运行期间不要手改`.reg`文件**，`wineserver`退出时会把内存态整体写回，手改会被覆盖，要改就用工具改或退出后改

## 文件格式

- 头部一行`WINE REGISTRY Version 2`，之后按键路径分段，键内逐行列出值：

  ```ini
  [Software\\Wine\\DllOverrides]          # 键路径，反斜杠写成两个
  "winemenubuilder"=""                    # REG_SZ 空字符串
  "mscoree"="native,builtin"              # REG_SZ 字符串
  "Enable"=dword:00000001                 # REG_DWORD
  "Payload"=hex:de,ad,be,ef               # REG_BINARY
  ```

- 值类型的编码：
  - `REG_SZ`：`"内容"`字符串
  - `REG_DWORD`：`dword:`后跟 8 位十六进制
  - `REG_BINARY`：`hex:`后跟逗号分隔的字节
  - `REG_EXPAND_SZ`：`hex(2):`，含环境变量待展开的字符串
  - `REG_MULTI_SZ`：`hex(7):`，多行字符串，行间以`00 00`分隔
- 语法与 Windows`.reg`导出文件同源，`regedit`导出的文件可直接导入，**但编码与换行要保持导出时的 UTF-16LE 与 CRLF**，手写文件建议交给工具导入

## 修改工具

- `wine regedit`：图形界面，操作与 Windows 的`regedit`完全一致，支持导出与导入`.reg`
- `wine reg.exe`：命令行，脚本友好，**配`WINEPREFIX`可精确操作某个 prefix**：

  ```console
  $ WINEPREFIX=~/games/gameX wine reg add "HKCU\\Software\\Wine\\DllOverrides" \
      /v mscoree /t REG_SZ /d native,builtin /f
  $ WINEPREFIX=~/games/gameX wine reg query "HKLM\\Software\\Wine" /s
  $ WINEPREFIX=~/games/gameX wine reg delete "HKCU\\Some\\Key" /v Name /f
  $ WINEPREFIX=~/games/gameX wine reg export "HKCU\\Software\\Wine" backup.reg
  $ WINEPREFIX=~/games/gameX wine reg import backup.reg
  ```

  - `/v`值名、`/t`类型、`/d`数据、`/f`跳过确认、`/s`递归查询
- 键名缩写与 Windows 一致：`HKLM`即`HKEY_LOCAL_MACHINE`，`HKCU`即`HKEY_CURRENT_USER`，`HKU`即`HKEY_USERS`
- 在 Windows 侧做过的修改，把同样的键路径与值在 Wine 里`reg add`一遍即可得到同等结果，**这是两边对齐的通用方法**

## 常见修改案例

- DLL 覆盖，`HKCU\Software\Wine\DllOverrides`：
  - 值名是要覆盖的 DLL 名，值为加载顺序，如`native,builtin`表示优先用 Windows 原生 DLL
  - 等价于`winecfg`中 Libraries 页的手动设置，但可脚本化地批量下发到多个 prefix
- 按程序伪装 Windows 版本，`HKCU\Software\Wine\AppDefaults\game.exe\Version`：
  - 字符串值`Windows`设为`win10`，只影响该程序，对应`winecfg`的 per-application 设置
- 中文界面乱码，`HKLM\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes`：
  - 把`MS Shell Dlg`与`MS Shell Dlg 2`指向`Microsoft YaHei`，解决字体缺失导致的方块字
- 窗口与虚拟桌面：
  - `HKCU\Software\Wine\X11 Driver`里`Decorated`=`N`可去掉窗口装饰
  - `HKCU\Software\Wine\Explorer\Desktops`定义桌面尺寸，再由`Explorer`的`Desktop`值启用，把窗口程序关进虚拟桌面
- 系统环境变量，`HKLM\System\CurrentControlSet\Control\Session Manager\Environment`：改`PATH`等，对应 Windows 的系统属性设置
- **改前先备份**：`reg export`整棵关心的子树，或直接`cp -a`整个 prefix 目录，注册表改坏是 Wine 排障的高频原因

## 相关阅读

- [在 Linux 上玩游戏](/2026/10/14/linux-gaming/)：Proton 如何为每个游戏管理独立 prefix
- [Linux 显示协议与图形栈](/2026/10/14/linux-display-stack/)：Wine 应用为何常运行在 XWayland 上
