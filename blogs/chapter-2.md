手撕Docker系列-第二章
===
终于开始写博客啦，希望能够通过这个过程让自己能够沉淀下来，更好地深入技术吧。因为最近入职于一家SAAS公司，平时工作里不可避免的要碰到k8s和docker，趁此机会也好好熟悉这两门技术。主要的学习手段是通过陈显鹭前辈的这本《自己动手写Docker》开始，一方面是实现一个东西可以让人对它了解更深刻，另一方面是熟悉go语言的使用，毕竟是工作语言...借此机会系统地学一下。

这一章主要分为两个部分，一个是namespace，一个是Cgroup，作为docker虚拟化概念中的核心技术。

# Namespce
Namespace是Linux的Kernel的一个功能，通过这个手段，Linux可以将其，我们在这里称之为各种资源隔离开来，其中包括进程，包括UserID，还有Network等。这也是docker镜像之间能够相互不干扰的原理锁在。

Namespace根据资源类型，我们可以将其分为好几种，具体的代码可以见我的github，也可以直接参考《自己动手写Docker》。在Linux中一共实现了6种不同类型的Namespace

## UTS Namespace

UTS Namespace用于隔离nodename和domainname，前者就是所谓的主机名，后者就是域名。在创建新的UTS namespace之后内部hostname独立于外部。

## IPC Namespace

IPC Namespace用于隔离System V IPC和POSIX message queues，前者就是Unix早期进程间通信的所有集合，包括管道（同时包括有名管道）、信号、消息队列、共享内存、信号量。后者是提供了实现POSIX标准的消息队列。

## PID Namespace

PID Namespace是用来隔离进程ID的。要注意这里是进程ID不是进程.同一个进程在不同PID Namespace可以拥有不同的PID。

## Mount Namespace

Mount Namespace用来隔离各个进程看到的挂载点视图。首先比较难理解的是挂载，在这里，我将其解释为将文件系统和目录树结合在一起的

# Cgroup