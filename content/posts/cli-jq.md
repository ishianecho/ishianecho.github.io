---
title: "jq 入门"
summary: "讲解 jq 的过滤语言：路径选择与迭代、对象构造、select 与 map 等内置过滤器、reduce 与替代运算符，以及 -r、-s、--arg 等命令行选项与实战组合"
date: 2026-09-17
categories: [Tools & Utilities, 3P Tools, jq]
tags: [tools, jq]
---

## 概述

- 参考文档：[jq Manual](https://jqlang.github.io/jq/manual/)
- `jq`是命令行 JSON 处理器，核心是一门"过滤器"语言：JSON 进、JSON 出，过滤器之间用`|`串接
- 最大的思维差异：一个输入可以产出多个输出，下游过滤器分别作用在每个输出上
- 能力清单：格式化打印、字段提取、流式变换、重组新 JSON、与`curl`等管道组合，是替代临时 Python 脚本的首选

## 快速开始

  ```console
  $ echo '{"name":"ishian","langs":["zh","en"]}' | jq .langs[0]
  "zh"
  $ jq . package.json                    # 等价于格式化打印
  $ curl -s https://api.github.com/users/octocat | jq .name
  "The Octocat"
  ```

## 路径与选择

- `.`是输入本身，`.name`取字段，`.a.b.c`逐层下钻，字段名特殊时写`.["a-b"]`
- 数组下标`.langs[0]`，负数从尾部取，切片`.langs[1:3]`
- `.[]`展开数组每个元素或对象每个值，各自成为下一个过滤器的输入：

  ```console
  $ echo '[1,2,3]' | jq '.[] | . * 10'
  10
  20
  30
  ```

- `?`抑制错误：`.name?`在输入不是对象时产出空而不是报错，遍历混合类型的数组用`.[]?`
- `..`递归展开所有层级，`.. | .price?`把任意深处的`price`都收出来
- 逗号`,`并行产出多个结果：`.name, .age`一条过滤器拿两个字段

## 构造与拼装

- `{}`与`[]`字面量里可以直接嵌过滤器，重组出新的 JSON：

  ```console
  $ echo '{"user":{"name":"a","age":15}}' | jq '{name: .user.name, adult: .user.age > 18}'
  {
    "name": "a",
    "adult": false
  }
  ```

  - 简写`{user}`等价于`{user: .user}`
- 数组收集：`[.[] | .name]`把多个输出收回一个数组，管道终点想要单结果时用它
- 字符串插值：`"hello \(.name)"`在字符串里嵌表达式

## 内置过滤器

- 筛选与映射是两大主力：
  - `select(cond)`：条件为真才产出，`.[] | select(.age >= 18)`
  - `map(f)`：等价`[.[] | f]`，`map_values(f)`保持键只变换值
  - `empty`：什么都不产出，配合`if`做丢弃
- 聚合与排序：`length`、`add`、`min`、`max`、`sort`、`sort_by(.k)`、`group_by(.k)`、`unique`、`unique_by(.k)`、`reverse`
- 对象操作：`keys`、`has("k")`、`to_entries`与`from_entries`互逆，`with_entries(f)`是"打散、变换、装回"的组合拳
- 字符串与类型：`tonumber`、`tostring`、`ascii_downcase`、`split(",")`、`join("-")`、`startswith`、`endswith`、`test("re")`与`capture`做正则
- 删除与兜底：`del(.a[0])`删除，替代运算符`a // b`在左侧为`false`或`null`时取右侧，给字段设默认值极常用
- 常用一行式：

  ```console
  $ jq '[.[] | select(.ok)] | length'                     # 计数满足条件的元素
  $ jq 'group_by(.type) | map({key: .[0].type, n: length})'  # 分组统计
  $ jq 'to_entries | map(select(.value == null) | .key)'  # 找出值为 null 的键
  ```

## 控制流与变量

- `if cond then a elif b then c else d end`，Go 风格不需要括号
- `reduce`做累积：`reduce .[] as $x (0; . + $x)`把数组求和
- `try f catch g`捕获错误，等价后缀`f?`只吞错不处理
- `limit(n; f)`截断输出条数，`first`/`last`取首尾
- 变量绑定：`表达式 as $x | ...`把中间结果命名后复用

## 命令行选项

- 输出形态：
  - `-r`输出裸字符串，**拼 shell 命令前必加**，否则结果带引号
  - `-c`压缩成单行，`-e`按有无结果设置退出码供脚本判断
  - `--tab`或`--indent n`控制缩进
- 输入形态：
  - `-n`忽略输入从`null`起步，构造数据用
  - `-s`把整个输入收成单个数组，配合`inputs`处理多段流
  - `-f prog.jq`过滤器放文件里复用
- 变量注入：

  ```console
  $ jq --arg id 42 '.items[] | select(.id == $id)'
  $ jq --argjson limit 5 '.[:$limit]'
  ```

  - `--arg`传字符串，`--argjson`传 JSON 值，`$ENV`访问环境变量
- 多输入流：`-n`模式下`inputs`逐个消费后续输入，`input`只取一个

## 相关阅读

- [yq 入门](/2026/09/19/cli-yq/)：同一套思路处理 YAML 与多文档
- [正则表达式](/2024/04/02/regex/)：`test`与`capture`所用的语法
