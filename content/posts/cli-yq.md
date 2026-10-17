---
title: "yq 入门"
summary: "讲解 mikefarah/yq v4 的表达式语法：取值与 select 过滤、赋值与删除、多文档处理、格式转换与注释保留，以及 K8s 清单的常用改写"
date: 2026-09-19
categories: [Tools & Utilities, 3P Tools, yq]
tags: [tools, yq]
---

## 概述

- 参考文档：[yq (mikefarah) Documentation](https://mikefarah.gitbook.io/yq/)
- 名为`yq`的工具不止一个，主流是 Go 编写的`mikefarah/yq` v4，表达式语法刻意贴近`jq`
- `yq --version`输出`mikefarah`字样才是本文语法，v4 起子命令与表达式用空格分隔
- 能力清单：字段取值、原地修改、多文档(`---`分隔)遍历与拆分、跨格式转换，**改值时保留注释**是它相对 sed 的核心优势

## 快速开始

  ```console
  $ yq '.metadata.name' deploy.yaml          # 取值，字符串裸输出
  $ yq '.spec.replicas = 3' -i deploy.yaml    # 原地改值
  $ yq -o=json '.' deploy.yaml               # YAML 转 JSON
  ```

- v4 的`e`/`eval`是默认子命令可省略，`ea`/`eval-all`跨全部文件与全部文档求值

## 取值与过滤

- 路径语法：`.a.b.c`逐层下钻，`.items[0]`数组下标，`.items[].name`迭代后继续下钻
- `select`过滤：`yq '.[] | select(.env == "prod")'`
- 常用函数：`keys`列键、`length`计数、`tag`看类型标签、`line`看节点所在行号，排错好用
- 变量：`--arg name val`配合`$name`引用，`env(VAR)`读环境变量：

  ```console
  $ yq '.image = env(IMG)' deploy.yaml
  ```

- 文件参数可以省略，从 stdin 读：`cat a.yaml | yq '.x'`

## 修改与赋值

- 赋值运算符：
  - `=`整体赋值：`.spec.replicas = 3`
  - `|=`基于现值变换：`yq '.name |= sub("v1"; "v2")'`
  - `+=`追加元素，`.labels += {"k": "v"}`
- 删除：`del(.metadata.annotations)`，多个路径一次删：`del(.a, .b)`
- **改完落盘必须加`-i`**，不加时只打印结果不动文件
- 合并与拆分：
  - 深度合并：`yq '. *= load("override.yaml")' -i base.yaml`，右侧覆盖左侧
  - 数组拆成多文档：`yq '.[] | split_doc' in.yaml`
- 注释：修改已有键的值时原注释保留，这是管线里敢用`-i`的前提

## 格式转换

- 输入输出格式分别用`-p`与`-o`指定，JSON 与 YAML 互转最常用：

  ```console
  $ yq -p=json -o=yaml '.' config.json    # JSON 转 YAML
  $ yq -P '.' config.json                 # -P 强制 pretty 风格输出
  ```

- 其它支持：`xml`、`props`、`csv`、`tsv`，XML 额外选项查手册
- JSON Lines 输入：`-p=json`逐行解析，配合`ea`做流式汇总

## 多文档处理

- YAML 文件里`---`分隔多个文档，普通`eval`只作用于第一个文档，**跨文档要`ea`**：

  ```console
  $ yq ea 'select(.kind == "Deployment")' all-in-one.yaml   # 只留 Deployment
  $ yq ea '.metadata.namespace = "prod"' -i *.yaml          # 批量改命名空间
  ```

- 按文档序号定位：`yq 'select(di == 1)'`取第二个文档
- 配合`fd`等工具批量处理目录下全部清单，是 K8s 日常的固定搭配

## K8s 实战

- 改镜像 tag：

  ```console
  $ yq '.spec.template.spec.containers[0].image |= sub(":.*$"; ":v2")' -i deploy.yaml
  ```

- 补 label 与注解：

  ```console
  $ yq '.metadata.labels.app = "web"' -i deploy.yaml
  ```

- 提取全部镜像名：

  ```console
  $ yq ea '[.spec.template.spec.containers[].image]' *.yaml
  ```

- 与`jq`的分工：纯 JSON 场景直接`jq`，涉及 YAML、多文档或注释保留时用`yq`，表达式思路互相迁移

## 相关阅读

- [jq 入门](/2026/09/17/cli-jq/)：语法的源头
