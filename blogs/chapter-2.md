---
layout:     post   				    # 使用的布局（不需要改）
title:      手撕Docker系列-第二章			# 标题 
subtitle:   肥宅程序员 #副标题
author:     By swimmingfish						# 作者
catalog: true 						# 是否归档
tags:								#标签
    - Docker
---

手撕Docker系列-第二章
===
终于开始写博客啦，希望能够通过这个过程让自己能够沉淀下来，更好地深入技术吧。因为最近入职于一家SAAS公司，平时工作里不可避免的要碰到k8s和docker，趁此机会也好好熟悉这两门技术。主要的学习手段是通过陈显鹭前辈的这本《自己动手写Docker》开始，一方面是实现一个东西可以让人对它了解更深刻，另一方面是熟悉go语言的使用，毕竟是工作语言...借此机会系统地学一下。

这一章主要分为两个部分，一个是namespace，一个是Cgroup，作为docker虚拟化概念中的核心技术。

# Namespce
Namespace是Linux的Kernel的一个功能，通过这个手段，Linux可以将其，我们在这里称之为各种资源隔离开来，其中包括进程，包括UserID，还有Network等。这也是docker镜像之间能够相互不干扰的原理所在。

Namespace根据资源类型，我们可以将其分为好几种，具体的代码可以见我的github，也可以直接参考《自己动手写Docker》。在Linux中一共实现了6种不同类型的Namespace

## UTS Namespace

UTS Namespace用于隔离nodename和domainname，前者就是所谓的主机名，后者就是域名。在创建新的UTS namespace之后内部hostname独立于外部。

## IPC Namespace

IPC Namespace用于隔离System V IPC和POSIX message queues，前者就是Unix早期进程间通信的所有集合，包括管道（同时包括有名管道）、信号、消息队列、共享内存、信号量。后者是提供了实现POSIX标准的消息队列。

## PID Namespace

PID Namespace是用来隔离进程ID的。要注意这里是进程ID不是进程.同一个进程在不同PID Namespace可以拥有不同的PID。

## Mount Namespace

Mount Namespace用来隔离各个进程看到的挂载点视图。首先比较难理解的是挂载，在这里，我将其解释为将文件系统和目录树结合在一起的一种结构，相当于将一个磁盘挂载至一个挂载点后，你就可以通过文件目录访问磁盘，并且文件系统也提供了inode、block等信息，更多的是一个关于怎么管理这片磁盘区域的配置信息。

## User Namespace

User Namespace相对来说就好理解很多。众所周知，在Linux里面每个User有自己的UID和GroupID，后者规定了用户所在的用户组（权限控制相关）。因此User Namespace就是来划分这个UID和GroupID的。在不同的User Namespace中，不同的用户可以有不同的UID和GroupID，而且在不同User Namespace中的ID也是没有关联的，比如UID=1在两个User Namespace中就是完全相互独立的。

## Network Namespace

这个书里讲得很清楚。Network Namespace就是用来隔离网络设备，IP端口等网络栈的命名空间。每个Network Namespace中可以拥有独立的虚拟网络设备和自己的端口，且与其他的Network Namespace不冲突。

# Cgroup

Linux Cgroup，全程Control Group。它的出现为的是解决一个什么问题呢？之前的Namespace，针对的只是Namespace之间的隔离。而我们如果要求能够对资源进行限制呢？现有的Namespace是无法做到这点的。于是Linux中就引入了Cgroup这个概念。Cgroup针对的是进程，相当于是一个进程分组框架。在Cgroup中主要有两个概念：hierarchy和subsystem。

## Hierarchy

Hierachy表达的是一个层次结构，Hierachy是一个树状结构，而在一个Hierachy下，可以绑定多个子Hierachy。这样就实现的进程的层级控制，所以一棵树更多的是相当于将进程进行分组。

## Subsystem

首先声明一个约束，一个subsystem只能挂载到一个Cgroup Hierachy节点上。而这个subsystem可以根据约束的资源，分为9种类型：
+ cpu subsystem （CPU使用率）
+ cpuacct subsystem（进程的CPU使用报告）
+ cpuset subsystem（为进程分配单独的CPU节点或者内存节点）
+ memory subsystem（内存分配）
+ blkio subsystem（设备io资源分配）
+ devices subsystem（设备访问控制）
+ net_cls subsystem（标记Cgroup下的进程数据包，使用tc模块（traffic control）进行数据包控制）
+ freezer subsystem （挂起恢复进程）
+ ns subsystem （使得不同cgroup下面的进程使用不同的namespace）

（net_cls和freezer还不是很懂）

