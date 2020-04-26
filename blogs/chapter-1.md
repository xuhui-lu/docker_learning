---
layout:     post   				    # 使用的布局（不需要改）
title:      手撕Docker系列-第一章			# 标题 
subtitle:   肥宅程序员 #副标题
author:     By swimmingfish						# 作者
catalog: true 						# 是否归档
tags:								#标签
    - Docker
---

手撕Docker系列-第一章
===
之前看到第一章是介绍go和docker相关知识就没有过多深入，今天有空看了看，觉得一些东西还是值得记录，因此写了这一章。

# Docker

能看到这篇文章的想必对docker也有一定了解，不必要的话就无需讲了。Docker作为virtualization的核心技术，可以将应用程序以及相应的依赖资源打包成一个标准的镜像，并以容器的方式运行在任何支持docker engine的系统上。Docker总体采用的是典型的C/S结构，并没有涉及很多底层或者分布式系统的东西。具体架构图如下。

![avatar](https://wiki.aquasec.com/download/attachments/2854889/Docker_Architecture.png?version=1&modificationDate=1520172700553&api=v2)

可以看到比较核心的是Docker Deamon，此外客户端和docker的交互就是通过api的形式进行。具体Deamon和其他组件做了什么，在后面的章节我们具体学习。

Docker的优势主要有三个，

- 轻量级：在同一台宿主机的容器共享系统的kernal，我们无需再搭建一个OS，因此启动速度快（秒级启动），占用系统内存少。又因为AUFS使得镜像之间能够通过分层结构共享文件，提高了磁盘的利用率和镜像下载速度。
- 开放：Docker容器基于开放标准，因此Docker可以在主流Linux和windows操作系统上运行。
- 安全：通过Namespace达到了资源的隔离，docker之间无法相互干扰，提供了额外的保障机制。

# Docker和VM
同样是模拟出一个独立的操作系统环境，VM虚拟机经常拿来和Docker进行比较。
![avatar](https://wiki.aquasec.com/download/attachments/2854889/Container_VM_Implementation.png?version=1&modificationDate=1520172703952&api=v2)

这个图就很明显了。VM的核心技术集中于Hypervisor上，Hypervisor更像是一个软件，基于OS的基础上，模拟出各种硬件的行为（包括CPU，硬盘等）。在Hypervisor的基础上，我们再搭建OS。这个缺点很明显，就是我们每需要打开一个虚拟机，我们就需要在Hypervisor上安装一个OS，动辄就是几个GB。

而Docker克服了VM的缺点，对开发人员开发效率来说，主要有三个帮助：

- 加速开发：Docker Registry提供了各种标准化的镜像，同时也提供了开发者定制镜像的能力，再也无需花费很久重新设置环境。
- 赋能创造力：对这一点，我理解为，由于启动一个docker成本低而且docker之间相互不干扰，使得我们可以为每一个程序设置最好的环境，避免依赖之间相互冲突，以及复杂的版本管理。
- 消除环境不一致：Docker的标准使得我们开发不用受开发环境的限制，无论在什么环境下达到开箱即用的效果。

此外，Docker Hub的存在使得一个团队可以通过共享镜像的方式实现协作开发。此外，docker秒级启动的特性能让服务迅速扩容。