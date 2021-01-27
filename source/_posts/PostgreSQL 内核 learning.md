---
title: PostgreSQL内核分析教程
categories:
- 数据库
- 源代码
tags: 
- system
- sql
- mldatabase
---

## 前言
现有的关系型数据库在系统稳定性，安全性等方面积累数十年的优势，如何改进现有的关系型数据库使之适用于当今的OLAP，机器学习（如提高单机大规模神经网络训练的能力）场景具有较大的潜力。目前为止Tidb的Tiflash并未开源（据说其论文目前于VLDB2020在审），PostgreSQL作为开源关系数据库中的王者，其内核值得学习，虽然现在一些工作已经对pg做出改进，但是spark引领的系统展现出势不可挡的趋势，期待传统关系型数据库重新复盘审视自己的设计模式，以总结出更多的经验，展现出更强大的生命力。

## 学习资料
- [阿里云psotgresql从入门到精通](https://zhuanlan.zhihu.com/p/27704963)
- [PostgreSQL内核开发学习资料](PostgreSQL内核开发学习资料)

## 1. 系统概述
### 1.1 简介及发展历程
- 由Stonebraker于1986年创建，开源版本使用BSD许可协议
- 1994年，UCB研究生吴恩达和另一名同学开发了一个SQL解释器，并将其重命名为PostgreSQL
### 1.2 Postgresql特性
- 内置丰富的数据类型，图，json，数组，用户可自定义类型。
- GiST索引
- Postgresql对标Oracle
- 与MySQL的对比：https://www.zhihu.com/question/20010554/answer/62628256
### 1.4 安装
- https://blog.csdn.net/u011652364/article/details/79286652
### 1.3 代码结构
- https://wiki.postgresql.org/wiki/Pgsrcstructure
- https://doxygen.postgresql.org/
- https://wiki.postgresql.org/wiki/Pgsrcstructure

## 2. Postgresql 体系结构
![image](https://img-blog.csdnimg.cn/20200426162239283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

### 2.1系统表

系统表存储了postgresql中所有数据对象及其属性的描述信息和对象之间的关系的描述信息，对象属性的自然语言含义和数据库状态信息的变化历史。

#### 2.1.1 主要系统表

编号 | 系统表 | 功能
---|---|---
1 | pg_namespace | 存储所有数据对象的名字空间,比如数据库，表，索引，视图
2 | pg_tablespace | 存储表空间信息，所有数据库共享一份pg_tablespace，有助于磁盘存储布局。
3 | pg_dataspace | 存储数据库信息
4 | pg_class | 存储表空间信息，所有数据库共享一份pg_tablespace，有助于磁盘存储布局。
... | ... | ...

#### 2.1.1 系统视图
由系统表生成，用于访问系统内部信息。

### 2.2 数据集蔟

- 定义: 用户数据库和系统数据库的集合。

- 数据库是具有特殊文件名，存储位置等属性信息的文件集合。
- OID是一个无符号整数，用于唯一标识数据集蔟中的所有对象，包括数据库，表，索引，视图，元组，类型等。其分配由一个全局计数器管理，互斥锁访问。


### 2.2.1 initdb的使用

使用postgresql之前用于初始化数据集蔟的程序，负责创建数据库系统目录，系统表，模板数据库。

## 3. 存储管理

### 3.2 外存管理

#### 3.2.4 空闲空间映射表FSM
- 随着表不断删除和插入元组(record)，文件块中会产生空闲空间。
- 插入元组时优先插入到表的空闲空间中。
- 每个表文件都有一个空闲空间映射表文件：关系表OID_fsm。
- FSM按照最大堆二叉树的形式保存空闲空间，能够使空闲空间最大值提升到根节点，这样只需要判断根节点是否满足需求，那可知是否有没有满足需求的空闲空间。

#### 3.2.5 可见性映射表VM
- VM表用来加快VACUUM(快速清理操作)查找无效元组文件块的过程，为表的每一个文件块设置了标志位，用来标记该文件块是否存在无效元组。

#### 3.2.5 大数据存储
Postgresql提供了两种大数据存储方式：
- TOAST:使用数据压缩和线外存储实现。
- 大对象机制：使用专门的系统表存储。

##### 3..2.5.1 TOAST
- TOAST机制主要优点在于查询时可以有效的减少占用的存储空间。
因为在查询时只比较压缩后的数据。
- 同样，压缩后的数据可以放到内存中，加速排序的速度。
### 3.3 内存管理
- [C++内存池的简单原理及实现](https://blog.csdn.net/u012234115/article/details/89852480?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.nonecase)


## 4. 索引
- 在postgre里，postgre的查询规则器会自动优化和选择索引

### 4.1 概述


## 5. 查询编译

查询模块框架和主要流程如下：
![image](https://img-blog.csdnimg.cn/20200520112830998.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

### 5.2 查询分析
查询分析主要流程：
![image](https://img-blog.csdnimg.cn/20200520113723263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

### 5.2.1 lex 和 yacc简介
- lex进行词法分析（正则表达式）
- yacc进行语法分析（BNF范式）
- 源代码经过lex进行词法分析提取关键词，传入yacc中进行语法分析，进行相应的操作，如加减乘除。

### 5.5.2 词法和语法分析
- scan.l:lex文件用于识别SQL的关键字
- gram.y:yacc用于分析词法分析后的语法
- 本节将主要分析SQL结构和经过语法分析后在内存中的数据结构 

![image](https://img-blog.csdnimg.cn/20200626184452355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

以 select语句为例，select语句可以由简单的select语句或者接where，sort等语句构成的复杂语句，

-----
持续更新ing...
-----
## 日志

<p style="color:green">远程隐藏</p>

##### 2020/--/--
...
##### 2020/--/--
...
##### 2020/--/--
...
