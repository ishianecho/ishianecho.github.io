---
title: "coreutils 入门"
summary: "介绍 coreutils 是什么，并讲解 ls、cat、cp、mv 等看一眼就会的基础命令的常用选项"
date: 2026-09-07
categories: [Infra, Linux]
tags: [linux, cli]
---

## 概述

- 参考文档：[GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/)
- `coreutils` 是 GNU 的核心工具集，包含文件操作、文本查看与 shell 环境相关的上百个基础命令，几乎所有 `Linux` 发行版都预装
- 历史上它由 `fileutils`、`textutils`、`shellutils` 等包合并而来，`ls --version` 输出里的 `GNU coreutils` 即归属证明
- 按功能分三类：

  ```mermaid
  flowchart LR
    A["coreutils"] --> B["文件操作"]
    A --> C["文本查看"]
    A --> D["身份与信息"]
    B --> B1["ls cp mv rm touch"]
    C --> C1["cat head tail"]
    D --> D1["pwd whoami hostname date"]
  ```

- 本文只讲"看一眼就会"的命令，`sort`、`xargs` 等需要专门学习的命令见[sed、awk 与进阶 coreutils](/2026/09/21/cli-sed-awk-coreutils/)

## 快速开始

- 每个命令都可用 `--help` 看简要用法，详细手册用 `man 命令`：

  ```console
  $ ls --help
  $ man cat
  ```

- 命令格式统一为`命令 选项 参数`，多个短选项可合并：`ls -lah` 等价于`ls -l -a -h`

## 身份与位置

- `pwd` 打印当前目录的绝对路径
- `whoami` 打印当前用户名，`hostname` 打印主机名
- `ls` 列出目录内容，最常用的组合是 `-lah`：
  - `-l` 长格式，逐行显示权限、链接数、属主、属组、大小与修改时间
  - `-a` 显示以`.`开头的隐藏文件
  - `-h` 以人类可读单位显示大小，如`1.5K`、`2.3M`
  - `-t` 按修改时间排序，配`-r`反转，找"最近改过的文件"
  - `-F` 给目录加`/`、可执行文件加`*`等类型后缀
  - `-1` 每行一个名字，给管道下游消费时干净
- `cd` 切换目录，注意它是 shell 内建命令而非 coreutils：
  - `cd -` 回到上一个目录
  - `cd ~` 或不带参数回到家目录

## 文件查看

- `cat` 顺序输出整个文件：
  - `-n` 显示行号
  - `-A` 显示所有不可见字符，`^I`是制表符、`$`是行尾，排查"看着一样其实不一样"的格式问题必备
  - 多个文件参数会依次拼接输出
- `head` 看开头，默认`10`行：
  - `-n 20`指定行数，`-c 100`按字节数截取
- `tail` 看结尾，选项同`head`：
  - `-f` 持续追踪文件增长，看日志的经典用法
  - `-n +2`从第`2`行输出到结尾，跳过表头常用
- **`cat` 会一次读整个文件**，大文件查看用`head`/`tail`或分页器`less`，`less`来自`util-linux`而非 coreutils

## 文件操作

- `touch` 更新文件时间戳，文件不存在时创建空文件
- `mkdir` 创建目录，`-p` 一次性创建多级路径且已存在不报错
- `rmdir` 只能删除空目录，实际更常用`rm -r`
- `cp` 复制文件或目录：
  - `-r` 递归复制目录，**复制目录必须加**
  - `-i` 覆盖前询问，`-f` 强制覆盖
  - `-a` 归档模式，等价于`-dR --preserve=all`，保留权限、时间戳与链接，备份首选
  - `-u` 只在源比目标新时复制，可做简易同步
- `mv` 移动或重命名：
  - `mv a b`是重命名，`mv a dir/`是移动进目录
  - 目标存在时默认直接覆盖，**批量移动前先确认目标路径**
  - `-n` 不覆盖已存在文件，`-u` 只移动更新的文件
- `rm` 删除文件：
  - `-r` 递归删除目录，`-f` 忽略不存在并不再询问
  - **没有回收站，删了就是删了**，脚本里慎用`rm -rf`拼出来的路径
- 权限与链接相关命令(`chmod`、`ln` 等)见[文件系统](/2024/03/12/linux-file/)

## 相关阅读

- [查找工具(find、fd 与 rg)](/2026/09/10/cli-find-fd-rg/)：定位到文件之后再做这些操作
- [sed、awk 与进阶 coreutils](/2026/09/21/cli-sed-awk-coreutils/)：文本流水线里的进阶命令
