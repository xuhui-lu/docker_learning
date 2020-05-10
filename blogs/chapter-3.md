---
layout:     post   				    # 使用的布局（不需要改）
title:      手撕Docker系列-第三章			# 标题 
subtitle:   肥宅程序员 #副标题
author:     By swimmingfish						# 作者
catalog: true 						# 是否归档
tags:								#标签
    - Docker
---

手撕Docker系列-第三章
===
今天在这个章节我们会讲如何应用我们前一章的知识，来创建一个容器进程。

# Linux proc文件系统

Linux的/proc目录其实不是一个真正的文件系统，因为真正的文件系统通过一系列的metadata对磁盘上的文件进行管理。而/proc下的内容包含了系统runtime的metadata，包括系统内存，mount设备以及一些硬件配置。它只存在于内存中，而不占用外存空间。事实上，它只是提供了一个接口让用户以访问文件的形式访问这些信息。

|  目录   | 内容  |
|  ----  | ----  |
| /proc/N | PID为N的进程信息 |
| /proc/N/cmdline | 进程启动命令 |
| /proc/N/cwd | 链接到进程当前工作目录 |
| /proc/N/environ | 进程环境变量列表 |
| /proc/N/exe | 链接到进程的执行命令文件 |
| /proc/N/fd | 包含进程相关的所有文件描述符 |
| /proc/N/maps | 与进程相关的内存映射信息 |
| /proc/N/mem | 指代进程持有的内存(不可读) |
| /proc/N/root | 连接到进程的根目录 |
| /proc/N/stat | 进程状态 |
| /proc/N/statm | 进程使用的进程状态 |
| /proc/N/status | 进程状态信息，比stat/statm更具可读性 |
| /proc/self/ | 链接到当前正在运行的进程 |

# Run命令启动

mydocker
├─README.md
├─main.go
├─main_command.go
├─run.go
├─network
&nbsp;|&emsp;└test_linux.go
├─container
&nbsp;|&emsp;├─container_process.go
&nbsp;|&emsp;└init.go
├─Godeps
&nbsp;|&emsp;├─Godeps.json
&nbsp;|&emsp;└Readme

文件目录如上所示。在这里我们不贴出过多的细节，主要把容器的启动过程分为几个部分。

## run
run的过程里，最主要的定义了一些运行的flag（tty）。`Run(tty, cmd)`是关键的部分，这部分代码有必要贴出来。

```
func Run(tty bool, command string) {
	parent := container.NewParentProcess(tty, command)
	if err := parent.Start(); err != nil {
		log.Error(err)
	}
	parent.Wait()
	os.Exit(-1)
}

func NewParentProcess(tty bool, command string) *exec.Cmd {
	args := []string{"init", command}
	cmd := exec.Command("/proc/self/exe", args...)
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
		syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
    }
	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	}
	return cmd
}
```
在`container.NewParentProcess(tty, command)`中，mydocker启动了一个进程`/proc/self/exe`，这个在我们之前的段落已经有提及，就是设置了一个进程变量，进程的可执行文件为当前进程的可执行文件。还有一些第二章提到的Namespace来进行资源隔离。包括UTS（hostname），PID（进程号），NS（挂载点），NET（网络资源）和IPC（进程交互通道）。在之后又设置了一些输入输出的通道。可能看到它调用自己会有点疑惑，其实更清楚的思路是，在创建这个进程的时候也传入了一个参数`init`，相当于调用./mydocker init。我们再看看init干了什么。

## init
在init的过程中，主要是init了container的process，最关键的一个函数叫做`container.RunContainerinitProcess(cmd, nil)`。

```
func RunContainerInitProcess(command string, args []string) error {
	logrus.Infof("command %s", command)

	defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
	syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
	argv := []string{command}
	if err := syscall.Exec(command, argv, os.Environ()); err != nil {
		logrus.Errorf(err.Error())
	}
	return nil
}
```

在这个函数里创建的进程本质上没有干什么事情。而函数本身首先规定了一个挂载参数，然后挂载到/proc上。之后再通过exec启动。

MountFlags的意义如下
* MS_NOEXEC： 这个文件系统中不允许运行其它程序。
* MS_NOSUID: 这个文件系统下不允许设置user_id或者group id。
* MS_NODEV: Linux2.0默认设置参数。

exec这句语句也很关键，看起来它只是简单执行了一个程序。但是其实背后有比较复杂的逻辑。首先，当我们运行完run命令之后，我们希望暴露给我们的前台进程是容器进程，然后目前为止PID为1的前台进程仍然是init进程，而syscall.Exec这个方法， 其实最终调用了Kernel的intexecve(const char filename,char *const argv[], char *const envp[]);这个系统函数会执行对应文件，并覆盖当前进程的镜像，堆栈和数据，包括PID。

在之后，我们容器已经启动了，我们可以通过`ps -ef`去查看目前的进程号是否为1。

# 增加容器资源限制
在这个部分，我们希望能够让mydocker实现资源限制。e.g. `mydocker run -ti -m IOOm -cpuset I -cpushare 512 /bin/sh`。其实在经历过第二章原理以后，这部分的实现相当容易。首先，假设我们在这里只考虑memory的限制，我们要做的就是创建一个memory subsystem。在这个memory subsystem中，我们将限制的变量写入memory挂载点下面指定的Cgroup。具体的subsystem的挂载点可以通过/proc/self/mountinfo来进行查看，值得注意的是，mountinfo里面得到的并不是文件系统下的绝对路径，我们仍然需要通过拼接等得到绝对路径。之后将限制参数写入Cgroup下的文件里。最后我们通过将进程加入挂载点下的指定Cgroup中来限制进程资源的使用。具体的代码实现可以看书中的实现

![avatar](https://docs.google.com/drawings/d/e/2PACX-1vR4tG0VAHY4bizgADLvOKWP_olEh5NMrS0_D0BeMQVmx7gabMJaqdB8xrtg_RqQunad32VTwAkCG5iw/pub?w=960&h=720)

# 增加管道和环境变量识别
首先在这个章节我们考虑一个问题，就是在容器中父子进程的通信。其实在初始化的过程中，父子进程就已经有一次通讯了。也就是我们需要启动`mydocker init --args`的时候，我们传给子进程的参数，包括init command和参数。当出现参数太长或者有特殊字符串的时候，这种办法就会失败。事实上runC采用的就是匿名管道的方法进行通信。我们在这里需要增加一个函数`NewPipe()`。

```
func NewPipe() (*os.File, *os.File, error) {
	read, write, err := os.Pipe()
	if err != nil {
		return nil, nil, err
	}
	return read, write, nil
}

func NewParentProcess(tty bool) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	cmd := exec.Command("/proc/self/exe", "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}
	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	}
	cmd.ExtraFiles = []*os.File{readPipe}
	return cmd, writePipe
}
```

NewPipe通过os创建了一个匿名管道用于父子进程交互。readPipe作为了cmd.ExtraFiles参数传给了init进程，此时，一个进程便拥有了四个文件句柄：标准输入，标准输出，标准错误以及这个readPipe。而写句柄则是传到外部，并将需要运行的command写入，从而让init进程能够读到这些参数。在这之后writePipe就被close了。

# 总结
至此，我们为进程添加了隔离环境，资源限制以及管道机制，基本上实现了一个容器进程的运行。