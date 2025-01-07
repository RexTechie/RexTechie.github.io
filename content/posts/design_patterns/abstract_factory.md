---
title: "抽象工厂模式（Abstract Factory）"
date: 2025-01-07T21:19:06+08:00
draft: false
description: "记录《大话设计模式》学习笔记，深入浅出23种设计模式，从实际应用出发理解设计模式的精髓"
categories: ["🏠设计模式"]
tags: ["设计模式",]
---

## 🎬场景

公司有一个给第三方企业做的电子商务网站，使用SQL Server数据库，已经都大致完成了。但是公司又接了另外一家公司类似的需求项目，但是这家公司想省钱，要用Access数据库。因此任务就变成了要将整个项目调整为用Access数据库。

但是替换的过程中出现了很多问题。
- SQL Server的命名空间和Access数据库不同：SQL Server上用的是System.Data.SqlClient，而Access上用的是System.Data.OleDb。
- 数据库操作语法不同：
  - Access插入数据必须用INSERT INTO，而SQL Server用INSERT（不能用INTO）。
  - SQL Server中的GetDate()函数在Access中是Now()。
  - SQL Server中有Substring()函数，而Access中是Mid()。
- 关键字不同
  - Access不能用password作为字段名，因为它是关键字，而SQL Server可以。要使用关键字需要用[]括起来。
- ...

如果今后要换成其他数据库，那么这个项目就要重新调整。这样的设计显然是不合理的。

一开始的代码结构如下：

用户实体: User.java
```java
/**
 * Description: 用户
 *
 * @author Rex
 * @date 2025-01-07
 */
public class User {
    private Integer id;
    private String name;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

SqlServerUser.java: 用于操作User表
```java
/**
 * Description: 用于操作User表
 *
 * @author Rex
 * @date 2025-01-07
 */
public class SqlServerUser {
    public void insertUser(User user) {
        System.out.println("在SQL Server中给User表增加一条记录");
    }

    public User getUser(Integer id) {
        System.out.println("在SQL Server中根据ID得到User表一条记录");
        return null;
    }
}
```  
Client.java: 客户端测试类
```java
/**
 * Description: 客户端
 *
 * @author Rex
 * @date 2025-01-07
 */
public class Client {
    public static void main(String[] args) {
        User user = new User();
        SqlServerUser su = new SqlServerUser();
        su.insertUser(user);
        su.getUser(1);
        System.out.println("-----------------------------------------");
    }
}
```


## 🎯目的

## 🚦结构

## 🛠解决
