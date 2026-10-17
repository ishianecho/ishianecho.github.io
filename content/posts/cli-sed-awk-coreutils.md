---
title: "sed、awk 与进阶 coreutils"
summary: "讲解 sed 的替换与地址、awk 的字段模型与关联数组、wc 的计数陷阱，以及 sort、uniq、cut、tr、xargs 等需要专门学习才能上手的命令"
date: 2026-09-21
categories: [Infra, Linux]
tags: [linux, cli]
---

## 概述

- 参考文档：
  - [GNU sed Manual](https://www.gnu.org/software/sed/manual/sed.html)
  - [GNU Awk User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html)
  - [GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/)
- 这批命令的共同点：**用过不等于会用**，各有自己的执行模型，值得专门学一遍
- 三个模型一句话：
  - `sed`：逐行流编辑，模式空间里改完就走
  - `awk`：记录与字段模型，天然是门小语言
  - coreutils 组合子：`sort`/`uniq`/`xargs`等在管道里各司其职
- 一条典型流水线：

  ```mermaid
  flowchart LR
    A["输入"] --> B["sed 改"]
    B --> C["awk 算"]
    C --> D["sort 排"]
    D --> E["uniq 聚"]
    E --> F["xargs 执行"]
  ```

## 快速开始

  ```console
  $ sed 's/foo/bar/g' file > out         # 全局替换
  $ awk '{sum += $1} END {print sum}' data.txt    # 第一列求和
  $ cat words | sort | uniq -c | sort -rn | head  # 词频统计
  ```

## `sed`

- 工作模型：逐行读入模式空间，执行完脚本后输出，**默认每行都回显**，`-n`关掉回显后用`p`精确控制输出
- 替换`s/正则/替换/标志`：
  - 默认只换每行第一处，`g`全局，`2g`从第`2`处起，`i`忽略大小写
  - `&`指代整个匹配，`\1`到`\9`指代分组，默认 BRE 里分组写`\(...\)`，`-E`切到 ERE 后直接`(...)`
  - 分隔符可换：`s|/usr|/opt|`替换路径时免转义
- 地址限定：
  - `5s/a/b/`只改第`5`行，`$`末行，`/start/,/end/s/…`区间，`!`取反
  - 多条命令用分号串联：`sed '/^#/d; /^$/d' config`删注释与空行
- 常用命令：`d`删除行、`p`打印、`a text`行后追加、`i text`行前插入、`c text`整行替换、`y/abc/xyz/`字符映射
- 高级特性：`N`把下一行并进模式空间做多行处理、`h`/`G`借助保持空间暂存、`:label`与`b`/`t`做循环，**简单场景够用时不要上这些**
- 原地修改：`-i`直接写回文件，实现是重写整个文件，**先裸跑确认输出，重要文件用`-i.bak`留备份**

## `awk`

- 模型：按记录处理，默认一行一记录，空白切字段：
  - `$0`整行，`$1`到`$n`各字段，`NF`当前行字段数，`NR`全局行号，`FNR`当前文件内行号
- 程序结构是`模式 { 动作 }`，模式命中才执行动作：
  - `BEGIN {}`处理输入前执行，放初始化与表头
  - `END {}`全部处理完执行，放汇总统计
  - 只有动作没有模式等于每行都执行

  ```console
  $ awk -F: '{print $1, $7}' /etc/passwd            # 冒号分隔取两列
  $ awk '$3 > 100 {print $1}' data.txt              # 条件过滤
  $ awk '{s += $2} END {print s, s/NR}' data.txt    # 求和与均值
  ```

- 变量与数组：变量无需声明，数组是**关联数组**，下标可以是任意字符串：

  ```console
  $ awk '{c[$1]++} END {for (k in c) print k, c[k]}' access.log   # 按键计数
  ```

- 内置函数：`length`、`substr`、`split`、`index`、`sub`/`gsub`替换、`printf`格式化输出、`sprintf`返回字符串
- 双文件操作：`NR==FNR`只在第一个文件里成立，配合`next`做交集与补集：

  ```console
  $ awk 'NR==FNR {a[$1]; next} $1 in a' f1 f2       # 输出两文件第一列的交集
  ```

- 命令行传参：`-v var=value`在`BEGIN`里可用，`-F,`设输入分隔，`-v OFS='|'`设输出分隔：

  ```console
  $ awk -F, -v OFS='|' '{$1 = $1; print}' data.csv   # 重建记录让 OFS 生效
  ```

- 实现：`gawk`功能最全，`mawk`更快但方言有差异，可移植脚本头部写`#!/usr/bin/awk -f`

## `wc` 与计数陷阱

- `wc`默认输出行数、词数、字节数，单独指定：`-l`行、`-w`词、`-c`字节、`-m`字符(受 locale 影响)、`-L`最长行宽
- **`-l`数的是换行符个数**，末行没有换行符时那一行不计入，`printf 'a' | wc -l`结果是`0`，统计日志先确认末行有换行

## 列与字符(`cut`、`tr`、`paste`、`comm`)

- `cut`按列切：
  - `-d, -f 1,3`按分隔符取第`1`、`3`列，`-f 2-`取到行尾
  - `-c1-8`按字符位置切
  - **分隔符只能是单个字符**，多字符分隔的场景换`awk -F`
- `tr`做字符级变换，不是正则：
  - `tr a-z A-Z`转大写，`tr -d '\r'`删字符，`tr -s ' '`压缩连续重复
  - 只吃 stdin，文件要用`< file`喂进去
- `paste`横向合并：两文件按行拼接，`-d`改分隔符，`paste -sd, file`把多行并成一行
- `comm`比较两个**已排序**文件，输出三列：仅前者有、仅后者有、共有，`-1 -2 -3`关掉对应列：

  ```console
  $ comm -12 <(sort a.txt) <(sort b.txt)    # 求交集
  ```

## 聚合(`sort`、`uniq`)

- `sort`是文本流水线的枢纽：
  - `-n`按数值、`-h`按人类单位(`2K`、`1M`)、`-r`降序、`-u`去重、`-R`随机、`-f`忽略大小写
  - `-t, -k2,2n`按第`2`列数值排序，**`-k`要写死终点`start,end`**，只写`-k2`会比较从第`2`列到行尾的整段
  - `-s`稳定排序，`-S 2G`加大内存缓冲
- `uniq`只比较**相邻行**，所以几乎总跟在`sort`后面：
  - `-c`带计数、`-d`只留重复行、`-u`只留唯一行
- 经典组合，词频 Top 10：

  ```console
  $ tr -cs 'a-zA-Z' '\n' < text | sort | uniq -c | sort -rn | head
  ```

## 执行与管道(`xargs`、`tee`)

- `xargs`把 stdin 转成命令参数，是"管道流"与"命令行参数"之间的桥：
  - `-n 1`每次只传一个参数，`-I{}`自定义占位符：`... | xargs -I{} cp {} backup/`
  - `-0`按`\0`切分，**接`find -print0`或`fd -0`，文件名带空格也不炸**
  - `-P 4`并行跑`4`个进程，`-r`空输入时不执行
  - 出身是`findutils`而非 coreutils，但总与它们同场出现
- `tee`把流一分为二：写文件的同时继续往 stdout 传，`-a`追加，长管道中间插一个可现场观测数据

## 文件与校验

- `stat`看完整元数据，`-c`自定义格式：`stat -c '%n %s %y' *`
- `readlink -f`把符号链接解析到底层绝对路径，`realpath`是等价的独立命令
- 校验清单：`sha256sum * > sums`生成，`sha256sum -c sums`批量校验，传完大文件的例行确认
- `split`切大文件：`-b 100M`按字节、`-l 10000`按行、`-d`用数字后缀，`cat part* > big`合并
- `mktemp`安全地创建临时文件或目录，**别自己拼`/tmp/foo.$$`**，并发冲突与符号链接攻击都会找上门
- `timeout 10s 命令`给命令限时，超时发`SIGTERM`，`-k 5s`后补`SIGKILL`
- `install`一步完成复制加权限设定，Makefile 常客：`install -m 755 app /usr/local/bin`
- `env`：`env VAR=x cmd`带变量运行命令，`env -i cmd`清空环境做最小复现

## 相关阅读

- [coreutils 入门](/2026/09/07/coreutils-beginner/)：先会用基础命令再来这批
- [查找工具(find、fd 与 rg)](/2026/09/10/cli-find-fd-rg/)：`xargs`的上游
- [Linux 进程管理命令](/2026/09/28/linux-process-cli/)：`timeout`发出的信号如何被进程处理
