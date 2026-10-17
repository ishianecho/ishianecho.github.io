---
title: "Linux 进程管理命令"
summary: "讲解 ps 的两种风格与 STAT 状态码、kill 家族与信号语义、lsof 查文件与端口占用、nice 与 renice 调整优先级，以及通过 /proc 深挖单个进程"
date: 2026-09-28
categories: [Infra, Linux]
tags: [linux, cli]
---

## 概述

- 参考文档：[Linux man-pages](https://man7.org/linux/man-pages/)
- 进程是运行中的程序实例，内核给每个进程一个 PID、一组状态与一张打开文件表，本篇的命令都在读或改这三样东西
- 命令地图：

  ```mermaid
  flowchart LR
    A{"要做什么"} -->|"看进程"| B["ps pgrep pstree"]
    A -->|"结束进程"| C["kill pkill killall"]
    A -->|"查占用"| D["lsof fuser"]
    A -->|"调优先级"| E["nice renice"]
    A -->|"深挖单个"| F["/proc/PID"]
  ```

## 快速开始

  ```console
  $ ps aux | rg nginx         # 找进程
  $ pgrep -a nginx            # 按名字找 PID 并显示完整命令行
  $ kill 1234                 # 发默认信号 SIGTERM
  $ lsof -i :443              # 谁在用 443 端口
  ```

## 查看进程

- `ps`有两套选项风格，**不要混搭**：
  - BSD 风：`ps aux`，`a`所有终端进程、`u`用户友好格式、`x`连无终端的进程，日常用这套
  - System V 风：`ps -ef`，信息量等价，老脚本里常见
- `ps aux`的列：`%CPU`与`%MEM`占用、`VSZ`虚拟内存(KB)、`RSS`常驻内存(KB)、`STAT`状态、`TIME`累计 CPU 时间
- `STAT`状态码：
  - `R`运行或就绪、`S`可中断睡眠、`Z`僵尸(已退出待父进程收尸)、`T`暂停
  - `D`不可中断睡眠，通常在等 IO，**这种状态连`kill -9`都无效**，只能等 IO 返回或重启
  - 后缀：`s`会话首进程、`l`多线程、`+`前台进程组、`<`高优先级、`N`低优先级
- 自定义输出与排序：`ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head`看吃 CPU 前几名
- `pgrep`与`pkill`共用一套过滤条件：
  - `-l`附带名字，`-a`附带完整命令行，`-u user`限定用户，`-t tty`限定终端
  - `-f`匹配整条命令行而不只是程序名
  - `-o`最老的、`-n`最新的、`-P 父PID`某进程的直接子进程
- `pidof nginx`精确按程序名取 PID，`pstree -p`看进程树确认父子关系

## 信号与结束进程

- `kill`的本质是发信号，`kill -l`列出全部，常用的就这几个：
  - `SIGTERM`(15)：默认信号，进程可以捕获并清理后退出，**先发它**
  - `SIGKILL`(9)：内核直接回收，进程无法捕获，数据可能损坏，是最后手段
  - `SIGINT`(2)：就是终端里的`Ctrl+C`
  - `SIGHUP`(1)：挂断信号，守护进程惯例用它当"重读配置"用
  - `SIGSTOP`与`SIGCONT`：暂停与继续，`Ctrl+Z`发的是`SIGTSTP`
- 标准姿势：`kill PID`，等几秒确认没死，再`kill -9 PID`
- `killall`按进程名精确匹配发送，`pkill`按正则模式发送，**`pkill -f`的匹配面覆盖整条命令行，误伤风险最大**：
  - 在 ssh 会话里`pkill -f ssh`会连自己的会话一起杀掉
- 指定信号可用数字`-9`也可用名字`-KILL`，脚本里用名字更可读

## `lsof`

- `lsof`列出打开的文件，Linux 下"文件"包括普通文件、目录、管道与 socket，所以端口占用也归它管
- 按端口与网络查：

  ```console
  $ lsof -i :22                  # 22 端口相关的所有连接
  $ lsof -i TCP -sTCP:LISTEN     # 所有 TCP 监听
  $ lsof -i -P -n                # 禁用端口与主机名解析，快很多
  ```

- 其它维度：`-p PID`某进程打开的全部文件、`-u user`某用户、`+D /path`目录下被打开的文件
- 经典场景：
  - `umount`报`target is busy`：`lsof +D 挂载点`揪出占用者
  - 磁盘满了但`du`找不到大文件：`lsof | rg deleted`，文件已删但进程还握着句柄，空间不释放，重启对应进程即可
  - 只取 PID 用于脚本：`kill -9 $(lsof -t -i:8080)`，`-t`只输出 PID
- `fuser`是同功能的另一把刀：`fuser -v 8080/tcp`看占用某端口的进程

## 优先级与后台

- nice 值范围`-20`(最高)到`19`(最低)，默认`0`：
  - 普通用户只能降低优先级(数值调大)，负值需要 root
  - 启动时指定：`nice -n 10 命令`，运行中调整：`renice -n 5 -p PID`
- 后台运行与脱离终端：
  - `命令 &`挂后台，`jobs`、`fg`、`bg`管理，退出登录时会收到`SIGHUP`
  - `nohup 命令 &`让进程免疫挂断信号，输出默认进`nohup.out`
  - `setsid`开新会话彻底脱离终端，`disown`把已在跑的作业移出挂断名单
- 长期任务推荐直接写成 systemd 服务或 timer，交给`systemctl`托管，见[systemd、systemctl 与 journalctl](/2026/09/12/systemd-systemctl-journalctl/)

## `/proc`

- `/proc/PID/`是内核暴露的进程档案，`ps`与`top`的数据源头就在这里：
  - `cmdline`：完整命令行，参数以`\0`分隔：`tr '\0' ' ' < /proc/PID/cmdline`
  - `environ`：环境变量，同样`\0`分隔
  - `status`：人读的状态汇总，含线程数与上下文切换次数
  - `fd/`：每个打开的句柄一个符号链接，`ls -l /proc/PID/fd`
  - `limits`：文件句柄数等资源上限
- 数字目录随进程生死增减，配合`watch`即可做极简监控

## 相关阅读

- [监控操作系统](/2025/01/11/linux-infos/)：top 与系统级监控视角
- [Go 调度器(GMP)](/2026/09/24/go-gmp/)：Go 的 M 在这里就是一个普通线程
- [操作系统的进程管理](/2025/04/03/os-process/)：状态与调度的理论侧
