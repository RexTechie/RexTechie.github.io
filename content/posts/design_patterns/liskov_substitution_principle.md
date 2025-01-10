---
title: "里氏替换原则(Liskov Substitution Principle)"
date: 2025-01-10T09:58:23+08:00
draft: false
categories: ["🏠设计模式"]
tags: ["设计模式",]
---

## 🚏 导论

> 里氏替换原则（LSP）：子类型必须能够替换掉它们的父类型。

里氏替换原则是Barbara Liskov女士在1988年发表的，[A behavioral notion of subtyping](https://dl.acm.org/doi/pdf/10.1145/197320.197383), 具体的数学定义如下所示：If S is a declared subtype of T, objects of type S should behave as objects of type T are expected to behave, if they are treated as objects of type T。用大白话翻译就是一个软件实体如果使用的是一个父类的话，那么一定适用于其子类，而且它察觉不出父类对象和子类对象的区别。也就是说，在软件里面，把父类都替换成它的子类，程序的行为没有变化。

由于有里氏替换原则，才使得[开放-封闭](../open_closed_principle)成为了可能，正是由于子类性的可替换性才使得使用父类类型的模块在无需修改的情况下可以扩展。

---

## 🎬 场景

### 场景一：🐒 动物世界

因为了有了这个原则，使得继承复用成为了可能，只有当子类可以替换掉父类，软件单位的功能不受到影响时，父类才能真正被复用，而子类也能够在父类的基础上增加新的行为。猫是继承动物类的，以动物的身份拥有吃、喝、跑、叫等行为，可当某一天，我们需要狗、牛、羊也拥有类似的行为，由于它们都是继承于动物，所以除了更改实例化的地方，程序其他处不需要改变。
