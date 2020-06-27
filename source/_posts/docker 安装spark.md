---
title: docker安装Spark
categories:
- experience
tags: 
- spark
- docker
---

疫情期间身边只有笔记本，想学习一哈spark（因为正在研究declarative  ml，要看SystemML的论文），配置环境太麻烦，因此想用docker pull 一下网上现成的镜像，由于是私人hub，docker直接pull时还是网络慢到让人崩溃。。。采用github上的自己build，由于国内网络原因，安装了n次，好几个g的镜像，简直让人崩溃。

过了n天后，偶然想起阿里云提供了免费的doker 仓库并支在线build。真是爽歪歪。。。

一句话总结整个过程的安装方式：利用github上的Dockerfile，使用阿里云提供的容器镜像服务，构建对应的 Docker镜像，从阿里云中pull到本地。

从dockerfile 到 build 全部在线完成，没有网络问题，速度也杠杠的，不得不感谢阿里云云计算的NB(因为免费，所以感谢，所以NB，haha)。

现在记录一下过程：
- 利用github fork https://github.com/sequenceiq/docker-spark。
- 阿里云容器镜像服务关联上述仓库，然后build，如果可以build成功，ok，本文结束。


人生之不如意，十有八九。下面记载我直接build过程会遇到如下错误（如果用github+阿里云，都会遇到下面的问题）：

![image](https://img-blog.csdnimg.cn/20200420223722488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

找不到epel安装源，大概是/etc/yum.repos.d/epel.repo 中镜像列表是https的问题，所以我直接修正并新建了一个epel.repo文件，build时替换掉原先的文件。
- 此外按照原先的rpm安装epel方式在安装R语言时会遇到依赖问题，因此我直接把Dockerfile中的
```
RUN rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```
修改为
```
RUN yum -y install epel-release
```
因为镜像在build的过程中不能和命令交互，因此yum时别忘了参数 ==-y==。

- 因为镜像太大，阿里云build时成功率不会是百分之百，比如：
![image](https://img-blog.csdnimg.cn/20200420230859514.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4MjU4OTAz,size_16,color_FFFFFF,t_70)

具体原因是：

![image](https://img-blog.csdnimg.cn/20200420230927347.png)

所以多尝试几次就好了（毕竟免费，免费）。。。

修改完后可以直接build成功的github仓库可以直接fork：https://github.com/kisisjrlly/docker-spark/

然后在本机上可以直接pull阿里云上build好的镜像啦，然后就可以愉快的玩耍啦。。。






### ref
- [修改阿里下载镜像](https://blog.csdn.net/weixin_43569697/article/details/89279225)
- [阿里容器镜像服务](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)
- [国内下载被墙的Docker镜像]( https://mp.weixin.qq.com/s/kf0SrktAze3bT7LcIveDYw)