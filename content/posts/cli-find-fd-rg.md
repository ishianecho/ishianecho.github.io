---
title: "查找工具(find、fd 与 rg)"
summary: "对比 find、fd 与 rg 三款查找工具的定位，讲解 find 的表达式与 exec、fd 的文件名搜索与批量执行、rg 的内容搜索与输出控制"
date: 2026-09-10
categories: [Infra, Linux]
tags: [linux, cli]
---

## 概述

- 参考文档：
  - [GNU Findutils Manual](https://www.gnu.org/software/findutils/manual/html_node/find.html)
  - [fd](https://github.com/sharkdp/fd)
  - [ripgrep](https://github.com/BurntSushi/ripgrep)
- 三者定位不同：`find`按条件遍历目录树，`fd`是更快更好用的文件名查找，`rg`专攻文件内容搜索
- `find`来自`findutils`，`fd`与`rg`是`Rust`编写的第三方工具，主流发行版仓库都有包
- 选择路径：

  ```mermaid
  flowchart LR
    A{"找什么"} -->|"文件名"| B{"条件复杂?"}
    B -->|"简单"| C["fd"]
    B -->|"size/mtime/perm 组合"| D["find"]
    A -->|"文件内容"| E["rg"]
  ```

## 快速开始

  ```console
  $ fd "\.md$"                        # 按正则找文件名，自动忽略隐藏与 ignore 文件
  $ rg "TODO"                         # 递归搜内容，自动忽略 .gitignore
  $ find . -name "*.log" -mtime +7    # 找 7 天前修改过的日志
  ```

## `find`

- 语法是`find 路径... 表达式`，**路径必须写在表达式前面**，不给路径默认当前目录，不给表达式默认`-print`
- 按名字：`-name "*.c"`通配符匹配，`-iname`忽略大小写，`-regex`对整个路径做正则
- 按类型与大小：`-type f/d/l`，`-size +10M`大于、`-10k`小于
- 按时间：`-mtime -7`表示`7`天内修改过，`+7`表示`7`天前，`-mmin`按分钟，`-newer file`比某文件新
- 按属性：`-user root`、`-group g`、`-perm -u+s`包含 setuid 位
- 深度：`-maxdepth 2`限制遍历层数，**属于全局选项，要放在测试类表达式之前**
- 组合逻辑：默认`-a`与，`-o`或，`!`非，`\( \)`分组：

  ```console
  $ find . \( -name "*.tmp" -o -name "*.swp" \) -type f
  ```

- 动作：
  - `-print0`以`\0`分隔输出，**文件名可能含空格或换行时必须配`xargs -0`**：

    ```console
    $ find . -name "*.log" -print0 | xargs -0 rm
    ```
  - `-exec 命令 {} \;`每个文件执行一次，`{} +`尽量合并成一次调用，批量场景更快
  - `-delete`直接删除，注意它也是普通表达式，位置放错会删错范围
- 陷阱：一旦表达式里出现了显式`-print`或`-exec`，**隐式的整包`-print`就没了**，`find . -name a -o -name b -print`只会打印匹配`b`的文件，要两侧都打印得加括号

## `fd`

- 默认对文件名做正则匹配，智能大小写：模式全小写则忽略大小写，含大写字母则精确匹配
- 常用选项：
  - `-e md`按扩展名过滤，可叠加：`-e c -e h`
  - `-H`包含隐藏文件，`-I`无视`.gitignore`，**默认跳过这两类与`.git`目录**
  - `-t f/d/l/x`按类型：文件、目录、符号链接、可执行文件
  - `-S +1m`按大小，`--changed-within 1week`按修改时间
  - `-E node_modules`额外排除某目录，`-d 2`限制深度
- 找到后执行命令：`-x`逐个执行，`-X`汇总成一次调用，占位符控制路径形式：

  ```console
  $ fd -e jpg -x convert {} {.}.webp    # {} 完整路径，{.} 去掉扩展名
  $ fd -e log -X rm                     # 等价 xargs 的汇总模式
  ```

  - 其它占位符：`{/}`仅文件名、`{//}`仅目录、`{/.}`去扩展名的文件名
- 与`find`的取舍：日常找名字用`fd`，需要`-perm`、复杂时间组合或老系统没有`fd`时回`find`

## `rg`

- 默认递归搜当前目录、跳过隐藏与`.gitignore`命中的文件，所以裸命令就够用
- 基本形态：`rg 模式 [路径...]`，模式是`Rust`正则，默认不支持反向引用，`-P`启用 PCRE2
- 大小写：**智能匹配是默认行为**，全小写模式自动忽略大小写，`-s`关闭，`-S`强制开启，`-w`按整词匹配
- 过滤范围：
  - `-t py`只搜某语言，`-T js`排除某语言，`--type-list`列出全部别名
  - `-g "*.toml"`按 glob 包含，`-g "!*.lock"`排除
  - `--hidden`搜隐藏文件，`--no-ignore`无视 ignore 文件，`-z`连压缩包一起搜
- 输出控制：
  - `-l`只列命中文件，`-c`每文件计数，`-o`只输出匹配部分
  - `-A 3`、`-B 3`、`-C 3`显示匹配之后、之前、两侧的上下文行
  - `-e a -e b`多模式取或，`-F`按字面量匹配不解析正则
- 替换与结构化：`-r`只替换显示内容不动文件，`--json`输出机器可读结果，`--vimgrep`输出编辑器 quickfix 格式
- 管道用法，从 stdin 逐行过滤：

  ```console
  $ ps aux | rg nginx
  $ rg -l "TODO" | wc -l               # 统计有多少文件含 TODO
  ```

## 相关阅读

- [coreutils 入门](/2026/09/07/coreutils-beginner/)：找到之后的文件操作
- [sed、awk 与进阶 coreutils](/2026/09/21/cli-sed-awk-coreutils/)：找到之后如何批量子处理
