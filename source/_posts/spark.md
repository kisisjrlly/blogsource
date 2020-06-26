---
title: Spark
categories:
- distributed computation
tags: 
- system
- distribute
- big date
- fault tolerance
---

## Introduction
spark：hadoop的升级版，通过定义新的数据抽象解决现有的问题，类似于数据结构中有数组，高维数组，集合，map，但是现有的数据结构不适用于特定的问题，spark抛开能想到的基础数据结构，重新定义新的数据抽象解决了现有以Hadoop为基础的大数据平台的特定问题。
- 现有的大数据平台的缺点：
  - Hadoop：保存到中间文件写磁盘，且不能重复利用中间数据，较多的磁盘IO等overead。
  - 存在对hadoop的改进版本系统：Pregel，然而，这些框架只支持特定的计算模式(例如，循环一系列MapReduce步骤)，并隐式地为这些模式执行数据共享。它们不提供更一般重用的抽象，例如，让用户将多个数据集加载到内存中，并在其中运行特别的查询。
- spark的解决方法
  - 定义数据抽象RDD：具有容错，数据并行执行，数据可显示保存到内存中去，优化的数据存放策略和丰富的API的特性。
  - 设计RDD主要的挑战是如何有效的容错（Google 的GFS，MapReduce 本质上是解决FT问题，因此引领大数据技术的发展）。
   - 现有的基于传统数据库的容错方式：replication+log overhead仍然较高，RDD可以通过重新计算（Lineage）的方式恢复（本质上的方法类似造火箭：我知道你是如何造出来的，你坏掉了我重新再造一个出来就好了）。
   - 但是Lineage计算较长时还是会用到log。
   - Spark设计目的专注于批处理分析的高效编程模型，不适用于对共享变量进行异步细粒度更新的应用程序，比如分布式InMemoryDB，对于这些系统的工作还是交给RamCloud好了。。。

### scala
是一种多范式的编程语言，类似于java，python编程，设计初衷是要集成面向对象编程和函数式编程的各种特性。

### 什么是RDD
RDD是一种数据抽象：弹性分布式数据集合（Resilient Distributed Dataset），是spark的基本元素，不可变，可分区，里面的元素可以被分布式并行执行（对码农透明）。
- Resilient
  - 代表数据可以存在内存也可以存到磁盘，计算快
- Distributed
  - 一个RDD被分布式存储，容错，又可以分布式并行计算
- Dataset
  - 类似于Hadoop中的文件，是一种抽象的分布式数据集合

### RDD五大特性
具有以下性质：
- A list of partitions
  - 一个RDD有多个分区，是一组分区列表，spark的task是以分区为单位，每个分布对应一个task线程（并行计算）。运行再worker节点的executor线程中。
- A function for computing each split
  -  函数会同时作用在所有的分区上。
- A list of dependencies on other RDDs
  - 新产生的RDD依赖于前期的存在的RDD（RDD被保存在内存中，可以被重复使用；可以实现无checkpoint+log的Fault tolerance机制）
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
  - （可选项）对于KV类型的RDD可以hashparition存储（必须产生shffule），如果不是KV类型----就表示木有
  - 在spark中，有两种分区函数：
    - 第一种：HashPationer函数             对key去hashcode，然后对分区数取余得到对应的分区号----------> key.hashcode % 分区数 = 分区号 。
    - 第二种：RangeParitoner函数，按照一定的范围进行分区，相同范围的key会进入同一个分区。（A-H）----> 1号分区，（I-Z）----> 2号分区。
 - Optionally, a list of preferred locations to compute each split on (e.g. block locations for
 an HDFS file)
   - （可选项）  一组最有的数据库位置列表：数据的本地性，数据的位置最优
     - spark后期任务计算优先考虑存在数据的节点开启计算任务，也就是说数据在哪里，就在哪里开启计算任务，大大减少网络传输。

以单词划分为例：
![image](https://img-blog.csdnimg.cn/20200227120058838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

### RDD 算子操作分类
- 1. transformation（转换）
  - 它可以把一份RDD转换生成一个心的Rdd，延迟加载，不会立即触发任务的真正运行
  - 比如flatMap/map/reduceByKey
- 2. action(动作)
  - 会触发任务的真正运行
  - 比如collect/SaveAsTextFile

### RDD的依赖关系

![image](https://img-blog.csdnimg.cn/20200227121033678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

- 1. narrow dependencies 窄依赖
  - where each partition of the parent RDD is used by at most one partition of the child RDD :父RDD的每一个分区至多只被子RDD的一个分区使用
  - 如map/filter/flaMap
  - 不会产生shuffle
- 2. wide dependencies 宽依赖
  - where multiple child partitions may depend on it：子RDD的多个分区会依赖于父RDD的同一个分区
  - 比如reduceByKey
  - 会产生shuffle

- 3. lineage 血统
  - RDD的产生路径
  - 有向无环图，后期如果某个RDD的分区数据丢失，可以通过lineage重新计算恢复。

### RDD的缓存机制
可以把RDD的数据缓存在内存或者磁盘中，后期需要时从缓存中取出，不用重新计算。

- 可以设置不同的缓存级别，如DISK_ONLY,DISK_ONLY_2,MEMORY_AND_DISK_SER_2（内存和磁盘都保存两份并序列化）
- 对计算复杂的RDD设置缓存。

### DAG的构建和构建stage
- lineage
  - 它是按照RDD之间的依赖生成的有向无环图
- stage
  - 后期会根据DAG划分stage：从图的lowest节点往前，构建初始stage，往前遍历DAG，如果是窄依赖，则加入此stage，如果是宽依赖则构建新的stage。
  - stage也会产生依赖：前面stage中task差生的数据流入后面stage中的task去。
  - 划分stage的原因
   - 由于一个job任务中可能会有大量的宽依赖，由于宽依赖不会产生shufflw，宽依赖会产生shuffle。划分完stage后，在同一个stage中只有窄依赖，则可以对应task并行执行： 所有的stage中由并行执行的task组成。
- App，job，Stage，Task之间的关系：
  - application 是spark的一个应用程序，包含了客户端写好的代码以及任务运行时所需要的资源信息。后期一个app中有多个action操作，每个action对应一个job，一个job由产生了多个stage，每一个stage内部有很多并行运行的task构成的集合。
 

## ref
- [Scala 中 _ 代表什么](https://www.jianshu.com/p/7ea0a450eec8)
- [原理视频教程](https://www.bilibili.com/video/av76914891?from=search&seid=13134650878163258083)
- [RDD源代码定义](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/RDD.scala)
- [scala教程](https://www.runoob.com/scala/scala-tutorial.html)
- [6.824讲义]()
- [spark论文原文]()
- [Scala中的"- >"和" -"以及"=>"](https://blog.csdn.net/someInNeed/article/details/90047624)
- [从Hadoop进化到Spark](https://www.zhihu.com/answer/41608400)
- [子雨大数据之Spark入门教程（Scala版）](http://dblab.xmu.edu.cn/blog/spark/)
## spark实践
### docker安装方式：
- https://blog.csdn.net/u013705066/article/details/80030732

## else

### spark内存计算
- https://www.zhihu.com/question/23079001/answer/23569986

