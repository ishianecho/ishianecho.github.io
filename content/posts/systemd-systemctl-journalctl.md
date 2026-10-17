---
title: "systemd、systemctl 与 journalctl"
summary: "介绍 systemd 的组成与 unit 类型，讲解 systemctl 管理服务、编写 service 与 timer 单元以及 journalctl 过滤与清理日志的常用用法"
date: 2026-09-12
categories: [Infra, Linux]
tags: [linux, systemd]
---

## 概述

- 参考文档：[systemd Official Documentation](https://systemd.io/)
- `systemd`是大多数发行版的`PID 1`初始化系统，负责按依赖并行启动 unit、用 cgroup 跟踪进程、把日志收进二进制存储
- 核心组件：
  - `systemd`：管理器本体
  - `systemctl`：控制 unit 的命令行前端
  - `journald`：结构化日志守护进程
  - `logind`、`networkd`、`resolved`等可选子系统
- 架构：

  ```mermaid
  flowchart LR
    A["systemctl"] -->|"D-Bus"| B["systemd (PID 1)"]
    B --> C["service"]
    B --> D["timer"]
    B --> E["mount / socket / target"]
    C & D & E -->|"stdout stderr"| F["journald"]
    F --> G["journalctl"]
  ```

## 快速开始

  ```console
  $ systemctl status sshd         # 看服务状态与最近日志
  $ systemctl restart sshd        # 重启服务
  $ systemctl enable --now sshd   # 设开机自启并立刻启动
  $ journalctl -u sshd -e         # 跳到该服务日志末尾
  ```

## Unit

- unit 是 systemd 的管理对象，类型由文件扩展名区分：
  - `.service`守护进程或一次性命令，最常见
  - `.timer`定时触发另一个 unit，现代替代`cron`的方案
  - `.socket`按需激活：第一个连接到来才启动对应 service
  - `.target`一组 unit 的汇聚点，如`multi-user.target`、`graphical.target`
  - `.mount`挂载点、`.path`文件变化触发、`.slice`资源分组
- 查找顺序：`/etc/systemd/system` > `/run/systemd/system` > `/usr/lib/systemd/system`，**同名覆盖，自己的修改放`/etc`**
- 修改 unit 文件后必须`systemctl daemon-reload`让 PID 1 重读，**忘了这一步改了等于没改**

## `systemctl`

- 状态与列表：

  ```console
  $ systemctl status UNIT                    # 状态加最近日志
  $ systemctl list-units --type=service --state=running
  $ systemctl list-unit-files --state=enabled
  $ systemctl is-active UNIT                 # 输出 active/inactive，退出码可判断
  $ systemctl cat UNIT                       # 看实际生效的 unit 文件
  $ systemctl show UNIT -p ExecStart         # 查单个属性
  $ systemctl list-dependencies UNIT
  ```

- 生命周期：

  ```console
  $ systemctl start|stop|restart UNIT
  $ systemctl reload UNIT                    # 不中断地重读配置
  $ systemctl reload-or-restart UNIT         # 有 reload 用之，否则重启
  $ systemctl enable|disable UNIT            # 开机自启开关，不改变当前状态
  $ systemctl mask|unmask UNIT               # 彻底禁用，连手动 start 都不行
  ```

- 电源与目标：`reboot`、`poweroff`、`suspend`、`hibernate`，`isolate multi-user.target`立即切到无图形界面，`set-default graphical.target`改默认启动目标
- 用户级：`systemctl --user`管理当前用户的服务，unit 放`~/.config/systemd/user/`，随登录会话生命周期

## 编写 service

- 最小可用的`/etc/systemd/system/myapp.service`：

  ```ini
  [Unit]
  Description=My App                      # 描述
  Wants=network-online.target             # 弱依赖，尽量拉起
  After=network-online.target             # 排序：网络就绪后再启动

  [Service]
  Type=simple                             # ExecStart 前台进程即主进程
  User=myapp                              # 降权运行
  WorkingDirectory=/srv/myapp
  ExecStart=/usr/bin/myapp --config /etc/myapp.toml
  Restart=on-failure                      # 异常退出自动拉起
  RestartSec=5                            # 拉起间隔

  [Install]
  WantedBy=multi-user.target              # enable 时挂到哪个 target
  ```

- `Type`选择：
  - `simple`：前台进程，默认值
  - `forking`：进程会 fork 到后台，主进程退出后才算就绪
  - `oneshot`：跑完即退出，配`RemainAfterExit=yes`显示 active
  - `notify`：进程自己调`sd_notify`报告就绪，systemd 自家服务多用
- 日志：service 默认把 stdout 与 stderr 收进 journal，**不需要再重定向到文件**

## Timer

- `myapp.timer`与同名`myapp.service`配对：

  ```ini
  [Unit]
  Description=Run myapp daily

  [Timer]
  OnCalendar=*-*-* 04:00:00              # 每天 04:00
  Persistent=true                        # 关机错过的开机后补跑
  RandomizedDelaySec=10m                 # 随机抖动，避免整点拥塞

  [Install]
  WantedBy=timers.target
  ```

- `OnCalendar`语法用`systemd-analyze calendar "..."`离线验证，相对定时用`OnBootSec=`与`OnUnitActiveSec=`
- 相比`cron`的优势：日志进 journal、依赖管理、错过补跑、失败自动重试，写计划任务的思路见[用 Go 写 cronjob](/2026/04/10/cronjob-go/)

## `journalctl`

- 过滤维度：

  ```console
  $ journalctl -u myapp                   # 只看某 unit
  $ journalctl -u myapp -f                # 追加模式，类似 tail -f
  $ journalctl -e                         # 跳到末尾
  $ journalctl -b                         # 本次开机以来
  $ journalctl -b -1                      # 上一次开机
  $ journalctl -k                         # 仅内核消息
  $ journalctl --since "2026-09-01" --until "2026-09-12 12:00"
  $ journalctl -p err                     # err 及更严重的优先级
  $ journalctl -g "timeout"               # 按正则过滤消息正文
  $ journalctl -n 50 --no-pager           # 最近 50 行且不分页
  ```

- 输出格式：`-o short-precise`微秒时间戳、`-o cat`仅消息正文、`-o json-pretty`结构化
- 磁盘治理：

  ```console
  $ journalctl --disk-usage
  $ journalctl --vacuum-time=7d
  $ journalctl --vacuum-size=500M
  ```

- 默认日志重启即失，**持久化要创建`/var/log/journal`目录**，长期留存的再配`SystemMaxUse=`限制体积

## 相关阅读

- [Linux 发行版与包管理器](/2026/09/08/linux-distro-package-manager/)：哪些发行版不用 systemd
- [Linux 进程管理命令](/2026/09/28/linux-process-cli/)：服务背后就是普通进程
