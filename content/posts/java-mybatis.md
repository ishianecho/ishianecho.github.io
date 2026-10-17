---
title: "MyBatis"
summary: "介绍 MyBatis 的 SqlSessionFactory 与 SqlSession 接口的使用"
date: 2025-07-16
categories: [Programming, Java, 3P Framework]
tags: [java, database]
---
## `SqlSessionFactory`

- 参考文档：[MyBatis](https://mybatis.org/mybatis-3/)
- `SqlSessionFactory`通常是单例模式，它的实例用于获取`SqlSession`，并与其他事务控制

## `SqlSession`接口

- `SqlSession`包含所有执行`Sql`的方法，类似`JDBC`的`Connection`，但它是后者的进一步封装，它将一些繁琐的错误处理，结果处理，参数绑定简化了
