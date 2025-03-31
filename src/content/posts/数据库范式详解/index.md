---
title: 数据库范式详解
published: 2024-10-10
category: '数据库'
tags: ["数据库"]
description: "数据库范式详解"
draft: false
---


It has been a long time since the last time I learning database related knowledge, but I do design database for my project often. As those project do contains some complex, I may Need to learn the DB NF again.

## 范式

范式是数据库设计的一个重要问题，无法绕开。通过范式，我们可以最大程度减少数据库表之间的冗余，实现更高效的存储、修改。

对于范式，分为6个层级，第一项讨论单一表项的规范，2NF至BCNF讨论函数依赖，4NF与6NF讨论多值依赖，而5NF(PJNF)讨论关联依赖。在具体论述范式前，需要讨论清楚一些关键词：

1. 主键：一组可以决定整个条目的属性
2. 主属性：主键中的属性
3. 非主属性：主键外的属性
4. 函数依赖：属性A可以唯一决定B，那么B函数依赖于A
5. 多值依赖：一组属性A决定一组属性B的一组值，且可以决定一组属性C的一组值，但BC独立。
6. 连接依赖：即A决定B、C、D等等，但BCD互相无关。如果表可以无损拆分，就存在连接依赖。（多值依赖即二元连接依赖）

注：多值依赖、连接依赖似乎可以从R与A两个角度来讲，当R中存在A到B、C的多值依赖，R就存在多值依赖，但若仅存在A到B的多值依赖，R就不存在多值依赖？？？连接依赖则是完全作用在R上的，由于当连接依赖仅分解出两个子集R1 R2时表现为多值依赖，所以多值依赖是特殊的连接依赖。

### 1NF

**列原子化**

即数据库的表的列仅能填写原子值，而非组合数据

### 2NF

**不要展开主键的部分主属性**

要求非主属性不存在对主属性的部分依赖

### 3NF

**不要展开非主属性**

要求非主属性不存在对主属性的传递依赖

不要太关注3NF，因为其被BCNF囊括。

### BCNF

**不要倒反天罡**

主属性不要依赖于非主属性

### 4NF

>  A [table](https://en.wikipedia.org/wiki/Table_(database) "Table (database)") is in 4NF [if and only if](https://en.wikipedia.org/wiki/If_and_only_if "If and only if"), for every one of its non-trivial multivalued dependencies _X_ ↠ _Y_, _X_ is a [superkey](https://en.wikipedia.org/wiki/Superkey "Superkey")—that is, _X_ is either a [candidate key](https://en.wikipedia.org/wiki/Candidate_key "Candidate key") or a superset thereof.

**不要组合排列**  _消除非平凡的多值依赖_

存在多值依赖例子：
{ID 爱好}
{ID 喜欢的食物}
ID->>爱好，ID->>喜欢的事物，此处存在非平凡多值依赖(X + Y != PK)。所以分解为
ID + 爱好 唯一确定一个条目，ID+喜欢的食物 唯一确定一个条目，满足4NF。

但无需过度关注4NF，因为5NF很好地囊括了4NF。

### 5NF

> A [table](https://en.wikipedia.org/wiki/Table_(database) "Table (database)") is said to be in the 5NF [if and only if](https://en.wikipedia.org/wiki/If_and_only_if "If and only if") every non-trivial [join dependency](https://en.wikipedia.org/wiki/Join_dependency "Join dependency") in that table is implied by the [candidate keys](https://en.wikipedia.org/wiki/Candidate_key "Candidate key"). It is the final normal form as far as removing redundancy is concerned.

**移除所有冗余** _所有非平凡连接依赖都由主键唯一决定_

可以无损分解就是存在连接依赖？ What about **six form**?

存在连接依赖例子：
{学生ID 学生姓名 课程名 分数 }
学生ID并不唯一决定课程名

所有涉及连接依赖的东西，都应该拆，将所有连接依赖分解到不同的表中。所有的子表都应该由相同的主键确定。

### 6NF

>A [relvar](https://en.wikipedia.org/wiki/Relvar "Relvar") R [table] is in **sixth normal form** (abbreviated 6NF) if and only if it satisfies no nontrivial join dependencies at all — where, as before, a [join dependency](https://en.wikipedia.org/wiki/Join_dependency "Join dependency") is trivial if and only if at least one of the projections (possibly U_projections) involved is taken over the set of all attributes of the relvar [table] concerned.[5](5)(<https://en.wikipedia.org/wiki/Sixth_normal_form#cite_note-FOOTNOTEDateDarwenLorentzos2003176-5>)

**表不可约** _不存在任何非平凡的连接依赖_

在5NF基础上，拆分所有可拆分的表，哪怕是{ID 姓名 年龄 地址}都要拆开成{ID 姓名} {ID 年龄} {ID 地址}（ID唯一决定姓名、年龄和地址）。6NF无助于减少冗余。

但是当我们希望研究历史数据时，这样拆分在加入历史数据后就不会造成冗余。

### 一些思索

这些依赖似乎都可以拆解为一对一决定关系、一对多决定关系的组合。

## 参考

[Join dependency - Wikipedia](https://en.wikipedia.org/wiki/Join_dependency#Formal_definition)
[Multivalued dependency - Wikipedia](https://en.wikipedia.org/wiki/Multivalued_dependency)
[Sixth normal form - Wikipedia](https://en.wikipedia.org/wiki/Sixth_normal_form)
