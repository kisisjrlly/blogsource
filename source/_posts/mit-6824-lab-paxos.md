---
title: Mit6.824-paxos lab实现
categories:
- distributed system
tags: 
- system
- distribute
---

平常说paxos一般是指lamport提出的basic paxos算法，后续所有的multi paxos 算法或者协议都是basic paxos算法的工程实现，以raft为例，都是借鉴了basic paxos 两阶段+投票的思想，但是在集群中各个节点的角色设置，如何处理网络故障，数据snapshot等方面做了basic paxos协议未提及的优化和实现。
basic paxos是为了解决分布式系统各个节点在不能保证可靠且实时通信的条件下，各个节点可以并发的提议请求，但是最终所有节点都对一个唯一的结果达成一致的算法。这是一个充满浪漫主义的算法，类比一些multi paxos 或者raft 算法就多了很多限制，比如加了所有节点中只有一个领导者可以提议请求，那么，整个一致性过程会变的简单的多。

### paxos 回顾
basic paxos处理的基本问题：
- 有一台或多台服务器会并发的提议（propose）一些值。
- 系统必须通过一定机制从propose的值中选定（chose）一个值。
- 只有一个值能被选定（chosen）。即不允许整个系统先选定了值A，后面又改选定值B。

basic的基本过程如下：

![image](https://img-blog.csdnimg.cn/2020122914594029.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70&ynotemdtimestamp=1616403019329)

伪代码如下：

```
  proposer(v):
  while not decided:
    choose n, unique and higher than any n seen so far
    send prepare(n) to all servers including self
    if prepare_ok(n_a, v_a) from majority:
      v' = v_a with highest n_a; choose own v otherwise
      send accept(n, v') to all
      if accept_ok(n) from majority:
        send decided(v') to all

acceptor's state:
  n_p (highest prepare seen)
  n_a, v_a (highest accept seen)

acceptor's prepare(n) handler:
  if n > n_p
    n_p = n
    reply prepare_ok(n_a, v_a)
  else
    reply prepare_reject

acceptor's accept(n, v) handler:
  if n >= n_p
    n_p = n
    n_a = n
    v_a = v
    reply accept_ok(n)
  else
    reply accept_reject
```

paxos算法本身的过程很简单，都说paxos难，难点有二：
- paxos难以理解，主要难以理解为什么是正确的
- 难以实现一个完全正确的paxos过程，上述伪代码中，集群中所有节点都会同时运行着所有的过程，每行代码之间都可能会并发和同步。

### Mit6.824 paxos实验
Mit6.824 关于 paxos的实验就是为了实现一个基本正确的paxos协议。之所以说基本正确是因为其提供的测试样例肯定没有考虑的所有的边界情况。但是，实现一个基本正确的paxos算法有助于理解上述提到的两个难点。整个算法的实现完全按照理论部分的过程，对于各个节点上唯一的提议号，我采用了初始序号+节点id的形式，如下：
```
type ProposeId struct {
	Ballot int
	Id     int
}

func greaterProposeId(a, b ProposeId) bool {
	if (a.Ballot > b.Ballot) || ((a.Ballot == b.Ballot) && a.Id > b.Id) {
		return true
	}
	return false
}

func equalProposeId(a, b ProposeId) bool {
	return a.Ballot == b.Ballot && a.Id == b.Id
}
```

用到的实验的版本是2014年版。整个实验过程大体如下：


这个实现中必须要实现以下的接口：
```
  // Make是用来创建一个paxos节点的，也就是说一个paxos节点的初始化函数
  px = paxos.Make(peers []string, me int)
  
  // 这个是用来对一个实例发起一致性过程的函数，seq代表paxos中的序列号，v代表这个序列号需要达成的内容是什么。。比如 100，“asdf”，都行
  px.Start(seq int, v interface{}) // start agreement on new instance
  
  // 用来获取一个节点是否认为已经决定了一个实例，如果已经决定了，那么商定的值是多少。Status()应该只检查本地节点的状态;它不应该与其他Paxos节点联系。
  px.Status(seq int) (decided bool, v interface{}) // get info about an instance
  
  px.Done(seq int) // ok to forget all instances <= seq
  
  px.Max() int // highest instance seq known, or -1
  
  px.Min() int // instances before this have been forgotten
  ```
  
  - 程序调用Make(peer,me)来创建Paxos peers。peer参数包含所有peers的端口(包括这个)，me参数是对等点数组中这个对等点的索引。Start(seq,v)请求Paxos启动实例seq协议，建议值v;Start()应该立即返回，而不是等待协议完成。应用程序调用Status(seq)来查明Paxos对等方是否认为实例已经达成协议，如果已经达成协议，那么商定的值是多少。Status()应查询本地Paxos对等点的状态并立即返回;它不应该与其他同行通信。应用程序可以为旧实例调用Status()(但是请参阅下面关于Done()的讨论)。
  
  - 实现需要能够同时在多个实例的协议上进行。也就是说，如果程序中各个peer使用不同的序列号同时调用Start()，那么实现应该同时为它们运行Paxos协议。在启动协议(例如i +1)之前，不应该等待协议(例如i)完成。每个实例都应该有自己独立的Paxos协议执行。
  
  - 长时间运行的基于paxos的服务器必须忘记不再需要的实例，并释放存储这些实例信息的内存。如果应用程序仍然希望能够调用该实例的Status()，或者另一个Paxos对等方可能尚未就该实例达成协议，则需要一个实例。您的Paxos应该通过以下方式实现实例的释放。当某个特定的对等应用程序不再需要为任何实例调用Status()时，它应该调用Done(x)。该Paxos对等点还不能丢弃实例，因为其他一些Paxos对等点可能还没有同意该实例。因此，每个Paxos对等点应该告诉其他对等点其本地应用程序提供的最高已完成参数。然后，每个Paxos对等点之间都有一个Done值。它们应该找到最小的那个值，并丢弃序列号<=该最小值的所有实例。Min()方法返回这个最小序列号加1。
  
  - 实现可以使用协议协议包中的Done值;也就是说，对等点P1可以在下一次P2向P1发送协议消息时只学习P2的最新Done值。如果调用Start()的序列号小于Min()，则应该忽略Start()调用。如果使用小于Min()的序列号调用Status()，则Status()应该返回false(表示没有达成一致了)。
  
  这里给出一个比较合理的实现步骤：
  - 在Paxos中向Paxos结构体添加元素。根据课程伪代码，去保存你需要的状态。您需要定义一个结构来保存关于每个协议实例的信息。
  - 基于lecture伪代码，为Paxos协议消息定义RPC参数/应答类型。rpc必须包含它们所引用的协议实例的序列号。记住RPC结构中的字段名必须以大写字母开头。
  - 为实例编写一个驱动Paxos协议的proposal函数，以及实现acceptor的RPC处理程序。根据需要在自己的线程中为每个实例启动一个proposal函数(例如，in Start())。

  
### 提示：
- 一个以上的Paxos实例可能在一个给定的时间执行，start 和 decide 过程需要容忍乱序执行。
- 为了通过假定网络不可靠的测试，您的paxos应该通过函数call调用本地接受器，而不是RPC。
- 请记住，多个peer可能在同一个实例上调用Start()，可能使用不同的提议值。peer序甚至可以为已经确定的实例调用Start()。
- 在开始编写代码之前，考虑一下您的paxos将如何忘记(丢弃)关于旧实例的信息。每个Paxospeer将需要在允许删除单个实例记录的数据结构中存储实例信息(以便Go垃圾收集器可以释放/重用内存)。
- 不需要来处理Paxos程序在崩溃后需要重新启动的情况。如果一个Paxos peer崩溃，它将永远不会重新启动。
- 让每个Paxos 节点为每个未决定的实例启动一个线程，该实例的任务是作为一个提议者最终驱动实例达成协议。
- 一个Paxos对等点可以同时作为同一个实例的接受者和提议者。将这两个活动尽可能分开。
- 一个提议者需要一种方法来选择一个比目前所见的任何一个都要高的提议数。这是提议人和接受人应该分开的规则的合理例外。如果提案RPC处理程序拒绝RPC，则返回已知的最高提案号，以帮助调用者下次选择更高的提案号。


 
  
  