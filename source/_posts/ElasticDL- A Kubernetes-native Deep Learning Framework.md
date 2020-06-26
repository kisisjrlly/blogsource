---
title: ElasticDL
categories:
- distributed computation
tags: 
- system
- distribute
- ml
- fault tolerance
---
# k8s原生的深度学习框架
ElasticDL是一个kubernets原生的深度学习框架，构建在TensorFlow 2.0之上，支持容错和弹性调度。


## 介绍
TensorFlow有其本地的分布式计算特性，可以进行故障恢复。当某些进程失败时，分布式计算作业也会失败;但是，我们可以重新启动作业，并从最近的检查点文件恢复其运行时状态。

ElasticDL增强了TensorFlow的分布式训练特性，支持容错。在某些进程失败的情况下，作业将继续运行。因此，ElasticDL不需要检查点，也不需要从检查点恢复。

容错特性使得ElasticDL与Kubernetes基于优先级的抢占机制协同工作，实现弹性调度。当Kubernetes终止一个作业的某些进程，将资源释放给具有更高优先级的新作业时，当前作业不会失败，而是在资源更少的情况下继续工作。

弹性调度可以显著提高集群的整体利用率。假设一个集群有N个gpu，一个作业使用其中一个。如果没有弹性调度，一个使用N个gpu的新作业在开始之前必须等待第一个作业完成。这个等待时间可能是几个小时、几天甚至几周。在这段很长的时间内，集群的利用率是1/N。使用弹性调度，新作业可以在N-1个GPU上立即开始运行，Kubernetes可以在第一个作业完成后将其GPU消耗增加1。在本例中，总利用率为100。

ElasticDL的弹性调度特性来自于它的Kubernetes本机设计——它不依赖Kubeflow等Kubernetes扩展来运行TensorFlow程序;相反，ElasticDL作业的主进程调用Kubernetes API来启动worker和参数服务器;它还监视诸如进程/pod杀死之类的事件，并对此类事件作出响应，以实现容错。

总之，在Kubernetes集群的情况下，ElasticDL通过容错和弹性调度增强了TensorFlow。我们提供了一个教程，演示如何在谷歌云中设置Kubernetes集群并在那里运行ElasticDL作业。我们尊重TensorFlow的本地分布式计算特性，它不需要像Kubernetes这样的特定计算平台，并且允许TensorFlow在任何平台上运行。


## 主要特性

### 弹性调度和容错
- ElasticDL通过Kubernetes-native设计实现容错，并与Kubernetes基于优先级的抢占机制协同工作，实现深度学习任务的弹性调度。

### TF2.0：Eager Execution
- 分布式深度学习框架在模型更新之前需要了解局部梯度。Eager Execution允许ElasticDL在不侵入图形执行过程的情况下执行。

### 极简API
给定一个用Keras API定义的模型，用命令行训练该模型。
```
elasticdl train --model_def=mnist_functional_api.custom_model --training_data=/mnist/train --output=output
```

### 与SQLFlow整合
- ElasticDL将与SQLFlow无缝集成，将SQL与ElasticDL连接到分布式深度学习任务。
```
SELECT * FROM employee LABEL income INTO my_elasticdl_model
```


## 链接
- [github主页](https://github.com/sql-machine-learning/elasticdl)