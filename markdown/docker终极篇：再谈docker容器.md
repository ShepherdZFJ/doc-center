# docker终极篇：再谈docker容器

### 1.docker容器概述

**容器**

容器其实是一种沙盒技术。顾名思义，沙盒就是能够像一个集装箱一样，把你的应用“装”起来的技术。这样，应用与应用之间，就因为有了边界而不至于相互干扰；而被装进集装箱的应用，也可以被方便地搬来搬去。

对于进程来说，它的静态表现就是程序，平常都安安静静地待在磁盘上；而一旦运行起来，它就变成了计算机里的数据和状态的总和，这就是它的动态表现。而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。基于Linux 内核的**Cgroup，Namespace，以及Union FS** 等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术，由于隔离的进程独立于宿主和其它的隔离的进程，因此称其为容器。Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护，使得Docker 技术比虚拟机技术更为轻便、快捷。

### 2.Namespace技术

Namespace 技术则是用来修改进程视图的主要方法。

首先我们运行一个容器并进入，这里用的镜像是busybox，BusyBox 是一个集成了三百多个最常用Linux命令和工具的软件。BusyBox 包含了一些简单的工具，例如ls、cat和echo等等，还包含了一些更大、更复杂的工具，例grep、find、mount以及telnet。有些人将 BusyBox 称为 Linux 工具里的[瑞士军刀](https://baike.baidu.com/item/瑞士军刀/152816)。简单的说BusyBox就好像是个大工具箱，它集成压缩了 Linux 的许多工具和命令，也包含了 Linux 系统的自带的shell。

```shell
[root@VM-4-10-centos /]# docker run -it busybox /bin/sh
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    8 root      0:00 ps
/ # 

```

可以看到，我们在 Docker 里最开始执行的 /bin/sh，就是这个容器内部的第 1 号进程（PID=1），而这个容器里一共只有两个进程在运行。这就意味着，前面执行的 /bin/sh，以及我们刚刚执行的 ps，已经被 Docker 隔离在了一个跟宿主机完全不同的世界当中。宿主机中很多进程信息，肯定不止这2个，那这是怎么做到的呢？答案是**namespace实现宿主机进程之间的隔离**。

本来，每当我们在宿主机上运行了一个 **/bin/sh 程序**，操作系统都会给它分配一个进程编号，比如 PID=100。这个编号是进程的唯一标识，就像员工的工牌一样。所以 PID=100，可以粗略地理解为这个 /bin/sh 是我们公司里的第 100 号员工，而第 1 号员工就自然是马云这样统领全局的人物。而现在，我们要通过 Docker 把这个 **/bin/sh 程序运行在一个容器当中**。这时候，Docker 就会在这个第 100 号员工入职时给他施一个“障眼法”，让他永远看不到前面的其他 99 个员工，更看不到马云。这样，他就会错误地以为自己就是公司里的第 1 号员工。

这种机制，其实就是对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。这种技术，**就是 Linux 里面的 Namespace 机制**。而 Namespace 的使用方式也非常有意思：它其实只是 Linux 创建新进程的一个可选参数。我们知道，在 Linux 系统中创建进程的系统调用是 clone()

```c

int pid = clone(main_function, stack_size, SIGCHLD, NULL); 
```

这个系统调用就会为我们创建一个新的进程，并且返回它的进程号 pid。而当我们用 clone() 系统调用创建一个新进程时，就可以在参数中指定 CLONE_NEWPID 参数，比如:

```c

int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```

这时，新创建的这个进程将会“看到”一个全新的进程空间，在这个进程空间里，它的 PID 是 1。之所以说“看到”，是因为这只是一个“障眼法”，在宿主机真实的进程空间里，这个进程的 PID 还是真实的数值，比如 100。当然，我们还可以多次执行上面的 clone() 调用，这样就会创建多个 PID Namespace，而每个 Namespace 里的应用进程，都会认为自己是当前容器里的第 1 号进程，它们既看不到宿主机里真正的进程空间，也看不到其他 PID Namespace 里的具体情况。

**除了我们刚刚用到的 PID Namespace，Linux 操作系统还提供了 Mount、UTS、IPC、Network 和 User 这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作**。Linux 的命名空间机制提供了以下七种不同的命名空间，包括 CLONE_NEWCGROUP、CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER 和 CLONE_NEWUTS，通过这七个选项我们能在创建新的进程时设置新进程应该在哪些资源上与宿主机器进行隔离。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/namespace.png)

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/namespace1.png)

所以，Docker 容器这个听起来玄而又玄的概念，实际上是在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，**容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置**。而对于宿主机以及其他不相关的程序，它就完全看不到了。

所以说，**容器，其实是一种特殊的进程而已**。

说到容器，很多人肯定会联想到虚拟机，下面是一张虚拟机和容器的对比图：

![](https://static001.geekbang.org/resource/image/d1/96/d1bb34cda8744514ba4c233435bf4e96.jpg?wh=2242*1163)

这幅图的左边，画出了虚拟机的工作原理。其中，名为 Hypervisor 的软件是虚拟机最主要的部分。它通过硬件虚拟化功能，模拟出了运行一个操作系统需要的各种硬件，比如 CPU、内存、I/O 设备等等。然后，它在这些虚拟的硬件上安装了一个新的操作系统，即 Guest OS。这样，用户的应用进程就可以运行在这个虚拟的机器中，它能看到的自然也只有 Guest OS 的文件和目录，以及这个机器里的虚拟设备。这就是为什么虚拟机也能起到将不同的应用进程相互隔离的作用。

在理解了 Namespace 的工作方式之后，你就会明白，跟真实存在的虚拟机不同，在使用 Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。Docker 项目帮助用户启动的，还是原来的应用进程，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数。这时，这些进程就会觉得自己是各自 PID Namespace 里的第 1 号进程，只能看到各自 Mount Namespace 里挂载的目录和文件，只能访问到各自 Network Namespace 里的网络设备，就仿佛运行在一个个“容器”里面，与世隔绝。

其实不应该把 Docker Engine 或者任何容器管理工具放在跟 Hypervisor 相同的位置，因为它们并不像 Hypervisor 那样对应用进程的隔离环境负责，也不会创建任何实体的“容器”，真正对隔离环境负责的是宿主机操作系统本身。

基于 Linux Namespace 的隔离机制相比于虚拟化技术也有很多不足之处，其中最主要的问题就是：**隔离得不彻底。**

首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。其次，在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。这就意味着，如果你的容器中的程序使用 settimeofday(2) 系统调用修改了时间，整个宿主机的时间都会被随之修改，这显然不符合用户的预期。相比于在虚拟机里面可以随便折腾的自由度，在容器里部署应用的时候，“什么能做，什么不能做”，就是用户必须考虑的一个问题。

**namespace常用操作命令**

```shell
# 查看当前系统的namespace： lsns
[root@VM-4-10-centos /]# lsns --help

用法：
 lsns [options] [<namespace>]

List system namespaces.

选项：
 -l, --list             use list format output
 -n, --noheadings       don't print headings
 -o, --output <list>    define which output columns to use
 -p, --task <pid>       print process namespaces
 -r, --raw              use the raw output format
 -u, --notruncate       don't truncate text in columns
 -t, --type <name>      namespace type (mnt, net, ipc, user, pid, uts)

 -h, --help     显示此帮助并退出
 -V, --version  输出版本信息并退出

可用列(用于 --output)：
          NS  namespace identifier (inode number)
        TYPE  kind of namespace
        PATH  path to the namespace
      NPROCS  number of processes in the namespace
         PID  lowest PID in the namespace
        PPID  PPID of the PID
     COMMAND  command line of the PID
         UID  UID of the PID
        USER  username of the PID

更多信息请参阅 lsns(8)。
[root@VM-4-10-centos /]# lsns -t pid
        NS TYPE NPROCS   PID USER COMMAND
4026531836 pid     117     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532174 pid       3 25964 root nginx: master process nginx -g daemon off
4026532237 pid       1 23358 root /bin/sh
4026532300 pid       3 14851 root nginx: master process nginx -g daemon off
4026532363 pid       1 18607 root /bin/bash
4026532426 pid       4  6592 root /bin/bash
[root@VM-4-10-centos /]# 


# 查看某进程的namespace：ls -la /proc/<pid>/ns/
[root@VM-4-10-centos /]# ls -la /proc/25964/ns
总用量 0
dr-x--x--x 2 root root 0 9月  27 17:28 .
dr-xr-xr-x 9 root root 0 9月  27 17:28 ..
lrwxrwxrwx 1 root root 0 10月 22 15:28 ipc -> ipc:[4026532173]
lrwxrwxrwx 1 root root 0 10月 22 15:28 mnt -> mnt:[4026532171]
lrwxrwxrwx 1 root root 0 9月  27 17:28 net -> net:[4026532176]
lrwxrwxrwx 1 root root 0 9月  27 19:20 pid -> pid:[4026532174]
lrwxrwxrwx 1 root root 0 10月 22 15:28 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 10月 22 15:28 uts -> uts:[4026532172]
[root@VM-4-10-centos /]# 
```

可以看到，一个进程的每种 Linux Namespace，都在它对应的 /proc/[进程号]/ns 下有一个对应的虚拟文件，并且链接到一个真实的 Namespace 文件上。

这也就意味着：**一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。**

而这个操作所依赖的，是一个名叫 **setns()方法 的 Linux 系统调用**

### 3.Cgroups技术

通过前面使用namespace技术实现容器的隔离，只能看到当前容器进程的相关信息，但是对于宿主机的资源(cpu、内存)，容器进程和其他进程还是共用着宿主机的cpu、内存等资源，是平等的竞争关系。虽然容器进程表面上被隔离了起来，但是它所能够使用到的资源（ CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的。这显然不是一个“沙盒”应该表现出来的合理行为。

**Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能**。**Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。**

此外，Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。在今天的分享中，我只和你重点探讨它与容器关系最紧密的“限制”能力，并通过一组实践来带你认识一下 Cgroups

在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下，如下所示：

```shell
[root@VM-4-10-centos /]# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
[root@VM-4-10-centos /]# 
```

可以看到，在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。比如，对 CPU 子系统来说，我们就可以看到如下几个配置文件，这个指令是：

```shell
[root@VM-4-10-centos /]# ls /sys/fs/cgroup/cpu
cgroup.clone_children  cgroup.sane_behavior  cpuacct.usage_percpu  cpu.rt_period_us   cpu.stat           release_agent  user.slice
cgroup.event_control   cpuacct.stat          cpu.cfs_period_us     cpu.rt_runtime_us  docker             system.slice   YunJing
cgroup.procs           cpuacct.usage         cpu.cfs_quota_us      cpu.shares         notify_on_release  tasks
[root@VM-4-10-centos /]# 
```

**cpu.shares**：可出让的能获得CPU 使用时间的相对值。

**cpu.cfs_period_us**：cfs_period_us 用来配置时间周期长度，单位为us（微秒）。

**cpu.cfs_quota_us**：cfs_quota_us 用来配置当前Cgroup 在cfs_period_us 时间内最多能使用的CPU时间数，单位为us（微秒）。

**cpu.stat** ：Cgroup 内的进程使用的CPU 时间统计。

cfs_period 和 cfs_quota 这两个参数需要组合使用，可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间

那cgroup下的配置文件又是怎样实现资源限制的呢？

首先我们/sys/fs/cgroup/cpu目录下，新建一个container目录：

```shell
[root@VM-4-10-centos cpu]# mkdir container
[root@VM-4-10-centos cpu]# cd container
[root@VM-4-10-centos container]# ls
cgroup.clone_children  cgroup.procs  cpuacct.usage         cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.event_control   cpuacct.stat  cpuacct.usage_percpu  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    tasks

```

这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的 container 目录下，自动生成该子系统对应的资源限制文件

接下来执行这样这样一条脚本

```shell
[root@VM-4-10-centos container]# while : ; do : ; done &
[1] 22405
```

显然，它执行了一个死循环，可以把计算机的 CPU 吃到 100%，根据它的输出，我们可以看到这个脚本在后台运行的进程号（PID）是 22405，使用top命令可以看到22405这个进程的cpu到100%。

而此时，我们可以通过查看 container 目录下的文件，看到 container 控制组里的 CPU quota 还没有任何限制（即：-1），CPU period 则是默认的 100  ms（100000  us）

```shell

[root@VM-4-10-centos container]# cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us 
-1
[root@VM-4-10-centos container]# cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us 
100000
```

这时我们可以通过修改这些文件的内容来设置限制。比如，向 container 组里的 cfs_quota 文件写入 20  ms（20000  us）：

```shell

[root@VM-4-10-centos container]# echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
```

在每 100 ms 的时间里，被该控制组限制的进程只能使用 20 ms 的 CPU 时间，也就是说这个进程只能使用到 20% 的 CPU 带宽。

最后，我们把被限制的进程的 PID 写入 container 组里的 tasks 文件，上面的设置就会对该进程生效了

```shell

[root@VM-4-10-centos container]# echo 22405 > /sys/fs/cgroup/cpu/container/tasks 
```

我们再次通过top查看进程资源消耗时，会发现进程22405的cpu消耗在20%

**Linux Cgroups 的设计还是比较易用的，简单粗暴地理解呢，它就是一个子系统目录加上一组资源限制文件的组合**。而对于 Docker 等 Linux 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个**控制组**（即创建一个新目录），然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 tasks 文件中就可以了。而至于在这些控制组下面的资源文件里填上什么值，就靠用户执行 docker run 时的参数指定了，比如这样一条命令：

```shell

[root@VM-4-10-centos cpu]# cd docker
[root@VM-4-10-centos docker]#  docker run -it -d --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677
[root@VM-4-10-centos docker]# ls
032c0b708b120b707e51f3b0ff95d76e8ee0b8f680edb6f938fe495baa8f65e4  cgroup.procs          cpu.rt_runtime_us
1a952391a49cf9a03a96bf4a8071cf5d860ffdea724c89a69d928288707221b6  cpuacct.stat          cpu.shares
1dc9690dc5d6663daacd3deaf4139962077aa13b6d7607cae11ddb838f24c1df  cpuacct.usage         cpu.stat
9ba13b73fe4ded53bd3dcd04a11a3cf63d47af5e5baf231c61133c9bb10eefd2  cpuacct.usage_percpu  f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677
b4e1ad5dd619af0db5e41a64d3032c52f578146394b053db62e9ee3cd42efddf  cpu.cfs_period_us     notify_on_release
cgroup.clone_children                                             cpu.cfs_quota_us      tasks
cgroup.event_control                                              cpu.rt_period_us
[root@VM-4-10-centos docker]# cd f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677/
[root@VM-4-10-centos f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677]# ls
cgroup.clone_children  cgroup.procs  cpuacct.usage         cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.event_control   cpuacct.stat  cpuacct.usage_percpu  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    tasks
[root@VM-4-10-centos f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677]# cat cpu.cfs_period_us
100000
[root@VM-4-10-centos f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677]# cat cpu.cfs_quota_us
20000
[root@VM-4-10-centos f219437891aedfbb51c6c3155f49a6d00499a5034e84afe618acb04f186f9677]# 

```

这就意味着这个 Docker 容器，只能使用到 20% 的 CPU 带宽。

另外，跟 Namespace 的情况类似，Cgroups 对资源的限制能力也有很多不完善的地方，被提及最多的自然是 **/proc 文件系统的问题**。众所周知，Linux 下的 /proc 目录存储的是记录当前内核运行状态的一系列特殊文件，用户可以通过访问这些文件，查看系统以及当前正在运行的进程的信息，比如 CPU 使用情况、内存占用率等，这些文件也是 top 指令查看系统信息的主要数据来源。但是，你如果在容器里执行 top 指令，就会发现，它显示的信息居然是宿主机的 CPU 和内存数据，而不是当前容器的数据。造成这个问题的原因就是，/proc 文件系统并不知道用户通过 Cgroups 给这个容器做了什么样的资源限制，即：/proc 文件系统不了解 Cgroups 限制的存在。

在生产环境中，这个问题必须进行修正，否则应用程序在容器里读取到的 CPU 核数、可用内存等信息都是宿主机上的数据，这会给应用的运行带来非常大的困惑和风险。比如说宿主机内存为64G，设置容器的内存为4G，在容器上运行java应用程序，由于没有设置jvm参数，即默认的，最后oom了，用命令查看容器的内存占用情况，竟然发现内存竟然有60多g。 那应该显示的是宿主机的内存了，jvm按照宿主机内存大小分配的默认内存应该大于4g 所以还没full gc 就oom了。这也是在企业中，容器化应用碰到的一个常见问题，也是容器相较于虚拟机另一个不尽如人意的地方。

### 4.挂载点

虽然我们已经通过 Linux 的namespace和cgroups解决了进程的的隔离和资源限制问题，但是 Docker 容器中的进程仍然能够访问或者修改宿主机器上的其他目录，这是我们不希望看到的。

首先有一个问题：**容器里的进程看到的文件系统又是什么样子的呢？**

可能我们马上想到 Mount Namespace 的解决：容器里的应用进程，理应看到一份完全独立的文件系统。这样，它就可以在自己的容器目录（比如 /tmp）下进行操作，而完全不会受宿主机以及其他容器的影响。

```c
int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | SIGCHLD , NULL); 
```

使用CLONE_NEWNS创建一个容器进程之后并进入，如果在容器里执行一下 ls 指令的话，我们就会发现一个有趣的现象： /tmp 目录下的内容跟宿主机的内容是一样的。

也就是说：即使开启了 Mount Namespace，容器进程看到的文件系统也跟宿主机完全一样。这是怎么回事呢？

**Mount Namespace 修改的，是容器进程对文件系统“挂载点”的认知。但是，这也就意味着，只有在“挂载”这个操作发生之后，进程的视图才会被改变。而在此之前，新创建的容器会直接继承宿主机的各个挂载点**。**Mount Namespace 跟其他 Namespace 的使用略有不同的地方：它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效。**

每当创建一个新容器时，我希望容器进程看到的文件系统就是一个独立的隔离环境，而不是继承自宿主机的文件系统。怎么才能做到这一点呢？不难想到，我们可以在容器进程启动之前重新挂载它的整个根目录“/”。而由于 Mount Namespace 的存在，这个挂载对宿主机不可见，所以容器进程就可以在里面随便折腾了。在 Linux 操作系统里，有一个名为 **chroot** 的命令可以帮助你在 shell 中方便地完成这个工作。顾名思义，它的作用就是帮你“change root file system”，即改变进程的根目录到你指定的位置。它的用法也非常简单。

```shell

[root@VM-4-10-centos docker-file]#  chroot $HOME/test /bin/bash
```

执行 chroot 命令，告诉操作系统，我们将使用 $HOME/test 目录作为 /bin/bash 进程的根目录。你如果执行 "ls /"，就会看到，它返回的都是 $HOME/test 目录下面的内容，而不是宿主机的内容。

实际上，Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace。当然，为了能够让容器的这个根目录看起来更“真实”，我们一般会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如 Ubuntu16.04 的 ISO。这样，在容器启动之后，我们在容器里通过执行 "ls /" 查看根目录下的内容，就是 Ubuntu 16.04 的所有目录和文件。而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。

现在，你应该可以知道，对 Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：

1）**启用 Linux Namespace 配置；**

2）**设置指定的 Cgroups 参数；**

3）**切换进程的根目录（Change Root）**

这样，一个完整的容器就诞生了。不过，Docker 项目在最后一步的切换上会优先使用 pivot_root 系统调用，如果系统不支持，才会使用 chroot。这两个系统调用虽然功能类似，但是也有细微的区别，这里不做区别阐述。

