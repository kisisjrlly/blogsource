---
title: Mit6.824-MapReduce 原型实现
categories:
- distributed system
tags: 
- system
- distribute
---

进入21世纪以来，谷歌带领着工业界和学术界一起迈向大数据时代。大数据的核心为分布式，而在谷歌的三驾马车三篇论文发表之前，工业界和学术界都没有意识到分布式系统中的重点在于容错，无论是分布式计算，分布式调度还是分布式存储。mapreduce是谷歌在分布式计算领域的开篇之作，主要介绍了在计算调度以及容错方面做的工作。原理方面，一图以蔽之：

![image](https://img-blog.csdnimg.cn/20210313160912137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

## mapreduce 原型框架
MIT 6.824 lab1 主要实现了一个简化版的mapreduce框架，包括计算，调度和容错过程。实验已经完成并通过了所有测试，代码见[github](https://github.com/kisisjrlly/6.824-all)：
![image](https://img-blog.csdnimg.cn/20210312141030581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

下面介绍一下整个实验的框架理解一下上面原理图的各个过程：

1. 客户端提供了多个输入文件、一个map函数、一个reduce函数和reduce任务的数量(nReduce)。
2. maprecude的中心角色master首先它启动一个RPC服务器（master_rpc.go)，并等待工作节点worker注册。 当tasks可用时（在步骤4和5中），schedule()函数决定如何将这些tasks分配给workers，以及如何处理worker故障。
3. master将每个输入文件视为一个map任务，并调用doMap()对每个map任务执行至少一次，可以直接（在使用Sequential（）时）执行对应的操作，也可以通过向worker发出DoTask RPC来执行。 每次对doMap（）的调用都会读取相应的文件，对该文件的内容调用map函数，并将生成的key/value对写入nReduce个中间文件中。doMap()通过hash每个key以选择中间文件，从而选择将处理该key的reduce任务。所有map任务完成后，会产生nMap x nReduce个文件。
4. 每个worker必须能够读任何其他worker写入的文件。实际部署中使用分布式存储系统(如GFS)来允许这种访问，即使worker运行在不同的节点上。在这个实验中，所有的worker在同一个节点上，并公用本地文件系统。
5. master接下来调用doReduce()，对每一项reduce任务至少执行一次。与doMap()一样，它可以直接执行，也可以通过一个worker执行。reduce任务r的doReduce()从每个map任务收集第r个中间文件，并为这些文件中出现的每个key调用reduce函数。reduce任务生成nReduce个结果文件。
6. master调用mr.merge() 将前面步骤生成的所有nReduce个文件合并到一个输出中文件中。
7. master向每个worker发送一个关闭RPC，然后关闭它自己的RPC服务

下面简要介绍一个本实验各个部分的内容，主要分为计算，调度和容错三个部分。

## Part1 实现map和reduce过程
本part主要实现划分map任务输出的函数，以及为reduce任务收集所有输入的函数。这个功能主要由common_map中的doMap()函数完成以及common_reduce中的doReduce()函数完成。

## Part3 分布式的mapreduce任务调度
part1中主要实现了一次运行一个map和reduce任务。Map/Reduce最大的卖点之一是它可以自动并行化执行各个tasks，而不需要使用人员做任何额外的工作。在本part中将完成一个分布式版本的MapReduce，它通过一组在多个核上并行运行的工作线程来分割工作。
虽然不像在实际的Map/Reduce部署中那样分布在多台机器上，将使用RPC来模拟分布式计算。

- mapreduce/master.go完成了管理MapReduce任务的大部分工作。mapreduce/worker中主要实现了worker线程的完整逻辑。mapreduce/common_rpc.go用于处理RPC。
- 本part主要工作是在mapreduce/schedule.go中实现schedule()。在MapReduce作业中，master调用schedule()两次，一次是在Map阶段，一次是在Reduce阶段。schedule()的工作是将任务分配给可用的worker。通常会有比工作线程更多的任务，因此schedule()必须给每个工作线程一个任务序列，一次一个。schedule()应该等待所有任务完成，然后返回。
- schedule()通过读取它的registerChan参数来了解worker集合。该channel保存了每个worker的RPC地址。有些worker可能在schedule()调用之前就存在，有些worker可能在schedule()运行时启动;所有的都会出现在registerChan上。
- schedule()应该使用所有的worker，包括在它启动后出现的worker。schedule()通过发送一个DoTask RPC来告诉worker执行一个任务。这个RPC的参数是由mapreduce/common_rpc.go中的DoTaskArgs定义的。File参数仅用于Map任务，是要读取的文件的名称;schedule()可以在mapFiles中找到这些文件名。


## Part4 处理worker故障
本part中，主要实现master处理失败的worker以进任务的重新分配。
- 如果一个worker在处理来自master的RPC时失败，master对应的调用函数call()将最终由于超时而返回false。在这种情况下，master应该把分配给失败worker的任务重新分配给另一个worker。
- RPC失败并不一定代表worker没有执行任务;worker可能已经执行了但是响应丢失了，或者worker可能仍然在执行，但是master的RPC超时了。因此，可能会发生两个worker接收相同的任务，计算并生成输出。对于给定的输入，需要两次调用map或reduce函数来生成相同的输出(即map和reduce函数是“幂等性”)，所以如果后续处理有时读取一个输出，有时读取另一个输出，就不会出现不一致的情况。
