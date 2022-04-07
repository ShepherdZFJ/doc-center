# docker基础篇:概念和命令

#### 1.docker产生的背景

一款产品从开发到上线，从操作系统，到运行环境，再到应用配置。作为开发+运维之间的协作我们需要关心很多东西，这也是很多互联网公司都不得不面对的问题，特别是各种版本的迭代之后，不同版本环境的兼容，对运维人员都是考验

Docker之所以发展如此迅速，也是因为它对此给出了一个标准化的解决方案。

环境配置如此麻烦，换一台机器，就要重来一次，费力费时。很多人想到，能不能从根本上解决问题，软件可以带环境安装？也就是说，安装的时候，把原始环境一模一样地复制过来。开发人员利用 Docker 可以消除协作编码时“在我的机器上可正常工作”的问题。

#### 2.什么是docker

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpS2ew9hVcUNaHLiaribePhCklnAeGlnJOfKq3eSrL59mCXQm97DO83znWLv26zSGtozOV61TBicjLoYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Docker是一种应用容器引擎。首先说一下何为容器，Linux系统提供了`Namespace`和`CGroup`技术实现环境隔离和资源控制，其中Namespace是Linux提供的一种内核级别环境隔离的方法，能使一个进程和该进程创建的子进程的运行空间都与Linux的超级父进程相隔离，注意Namespace只能实现运行空间的隔离，物理资源还是所有进程共用的，为了实现资源隔离，Linux系统提供了CGroup技术来控制一个进程组群可使用的资源（如CPU、内存、磁盘IO等），把这两种技术结合起来，就能构造一个用户空间独立且限定了资源的对象，这样的对象称为容器。

`Linux Container`是Linux系统提供的容器化技术，简称`LXC`，它结合Namespace和CGroup技术为用户提供了更易用的接口来实现容器化。LXC仅为一种轻量级的容器化技术，它仅能对部分资源进行限制，无法做到诸如网络限制、磁盘空间占用限制等。dotCloud公司结合LXC和`以下列出的技术`实现了Docker容器引擎，相比于LXC，Docker具备更加全面的资源控制能力，是一种应用级别的容器引擎。

- Chroot：该技术能在container里构造完整的Linux文件系统；
- Veth：该技术能够在主机上虚拟出一张网卡与container里的eth0网卡进行桥接，实现容器与主机、容器之间的网络通信；
- UnionFS：联合文件系统，Docker利用该技术“Copy on Write”的特点实现容器的快速启动和极少的资源占用，后面会专门介绍该文件系统；
- Iptables/netfilter：通过这两个技术实现控制container网络访问策略；
- TC：该技术主要用来做流量隔离，限制带宽；
- Quota：该技术用来限制磁盘读写空间的大小；
- Setrlimit：该技术用来限制container中打开的进程数，限制打开的文件个数等

正是因为Docker依赖Linux内核的这些技术，至少使用3.8或更高版本的内核才能运行Docker容器，官方建议使用3.10以上的内核版本。

docker的主要目标是"**Build,Ship and Run any App,Angwhere**",构建，运输，处处运行

- **构建：**做一个docker镜像
- **运输：**docker pull
- **运行：**启动一个容器

每一个容器，他都有自己的文件系统rootfs

#### 3.docker基本概念

**镜像**

类似于虚拟机镜像，一般由一个基本操作系统环境和多个应用程序打包而成，是创建容器的模板。镜像可以用来创建 Docker 容器，一个镜像可以创建多个容器。

**容器**

容器是用镜像创建的运行实例，Docker 利用容器独立运行一个或一组应用。它可以被启动、开始、停止、删除，每个容器都是相互隔离的、保证安全的平台。可以把容器看作是一个简易的 Linux 环境和运行在其中的应用程序。容器的定义和镜像几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

**仓库**

仓库可看成一个代码控制中心，用来保存镜像

#### 4.docker命令

**实用入门命令**

```bash
docker version    #显示docker的版本信息。
docker info       #显示docker的系统信息，包括镜像和容器的数量
docker 命令 --help #帮助命令
```

下面示例查询docker ps命令的用法：

```shell
[root@VM-4-10-centos ~]# docker ps --help

Usage:  docker ps [OPTIONS]

List containers

Options:
  -a, --all             Show all containers (default shows just running)
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print containers using a Go template
  -n, --last int        Show n last created containers (includes all states) (default -1)
  -l, --latest          Show the latest created container (includes all states)
      --no-trunc        Don't truncate output
  -q, --quiet           Only display container IDs
  -s, --size            Display total file sizes

```

**镜像命令**

```shell
docker images #查看所有本地主机上的镜像

docker search #搜索镜像
docker pull #下载镜像 
docker rmi #删除镜像
```

**docker search  搜索镜像**

```shell
[root@VM-4-10-centos ~]# docker search mysql
NAME                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql                             MySQL is a widely used, open-source relation…   11422     [OK]       
mariadb                           MariaDB Server is a high performing open sou…   4339      [OK]       
mysql/mysql-server                Optimized MySQL Server Docker images. Create…   847                  [OK]
centos/mysql-57-centos7           MySQL 5.7 SQL database server                   91                   
mysql/mysql-cluster               Experimental MySQL Cluster Docker images. Cr…   88                   
centurylink/mysql                 Image containing mysql. Optimized to be link…   59                   [OK]

# --filter=STARS=3000 #搜索出来的镜像就是STARS大于3000的
Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
      --no-trunc        Don't truncate output
      
[root@VM-4-10-centos ~]# docker search mysql --filter=stars=3000
NAME      DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql     MySQL is a widely used, open-source relation…   11422     [OK]       
mariadb   MariaDB Server is a high performing open sou…   4339      [OK]  

```

**docker pull 拉取镜像**

```shell
# 下载镜像 docker pull 镜像名[:tag]
[root@VM-4-10-centos ~]# docker pull tomcat:8
8: Pulling from library/tomcat #如果不写tag，默认就是latest
90fe46dd8199: Already exists   #分层下载： docker image 的核心 联合文件系统
35a4f1977689: Already exists 
bbc37f14aded: Already exists 
74e27dc593d4: Already exists 
93a01fbfad7f: Already exists 
1478df405869: Pull complete 
64f0dd11682b: Pull complete 
68ff4e050d11: Pull complete 
f576086003cf: Pull complete 
3b72593ce10e: Pull complete 
Digest: sha256:0c6234e7ec9d10ab32c06423ab829b32e3183ba5bf2620ee66de866df640a027  # 签名 防伪
Status: Downloaded newer image for tomcat:8
docker.io/library/tomcat:8 #真实地址

```

docker images  查看镜像

```shell
[root@VM-4-10-centos ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
tomcat       8         551efb60a0a2   2 days ago    678MB
mysql        5.7       1d7aba917169   2 weeks ago   448MB
mysql        8.0       0716d6ebcc1a   2 weeks ago   514MB

# 解释
#REPOSITORY			# 镜像的仓库源
#TAG				# 镜像的标签
#IMAGE ID			# 镜像的id
#CREATED			# 镜像的创建时间
#SIZE				# 镜像的大小
# 可选项
Options:
  -a, --all             Show all images (default hides intermediate images) #列出所有镜像
  -q, --quiet           Only show numeric IDs # 只显示镜像的id
  
```

**docker rmi 删除镜像**

```shell
docker rmi -f 镜像id #删除指定的镜像
docker rmi -f 镜像id 镜像id 镜像id 镜像id   #删除指定的多个镜像
docker rmi -f $(docker images -aq) #删除全部的镜像

```

**容器命令**

```shell
docker run 镜像id #新建容器并启动
docker ps #列出所有运行的容器 docker container list
docker rm 容器id #删除指定容器
docker start 容器id #启动容器
docker restart 容器id #重启容器
docker stop 容器id #停止当前正在运行的容器
docker kill 容器id #强制停止当前容器
```

**新建容器并启动**

```shell
docker run [可选参数] image | docker container run [可选参数] image 
#参书说明
--name="Name"		容器名字 tomcat01 tomcat02 用来区分容器
-d					后台方式运行
-it 				使用交互方式运行，进入容器查看内容
-p					指定容器的端口 -p 8080(宿主机):8080(容器)
		-p ip:主机端口:容器端口
		-p 主机端口:容器端口(常用)
		-p 容器端口
		容器端口
-P(大写) 				随机指定端口
-v             数据卷文件挂载

#示例
docker run -p 3306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7


# 命令 docker run -d 镜像名
➜  ~ docker run -d centos
a8f922c255859622ac45ce3a535b7a0e8253329be4756ed6e32265d2dd2fac6c
➜  ~ docker ps           
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
# 问题docker ps. 发现centos 停止了
# 常见的坑，docker容器使用后台运行，就必须要有要一个前台进程，docker发现没有应用，就会自动停止
# nginx，容器启动后，发现自己没有提供服务，就会立刻停止，就是没有程序了
```

**列出所以运行的容器**

```shell
#docker ps命令 #列出当前正在运行的容器
  -a, --all             Show all containers (default shows just running)
  -n, --last int        Show n last created containers (includes all states) (default -1)
  -q, --quiet           Only display numeric IDs

[root@localhost ~]# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED        STATUS        PORTS                                                                                  NAMES
546c10c34d3a   kibana:7.4.2          "/usr/local/bin/dumb…"   4 months ago   Up 26 hours   0.0.0.0:5601->5601/tcp, :::5601->5601/tcp                                              kibana
803d0e08328b   elasticsearch:7.4.2   "/usr/local/bin/dock…"   4 months ago   Up 26 hours   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp   elasticsearch
8ce2095189e3   redis                 "docker-entrypoint.s…"   4 months ago   Up 26 hours   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                              redis
4d97c6b94366   mysql:5.7             "docker-entrypoint.s…"   4 months ago   Up 26 hours   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp                                   mysql

```

**退出容器**

```shell
exit #容器直接退出
ctrl +P +Q #容器不停止退出
```

**删除容器**

```shell
docker rm 容器id   #删除指定的容器，不能删除正在运行的容器，如果要强制删除 rm -rf
docker rm -f $(docker ps -aq)  #删除指定的容器
docker ps -a -q|xargs docker rm  #删除所有的容器
```

**启动和停止容器的操作**

```shell
docker start 容器id	#启动容器
docker restart 容器id	#重启容器
docker stop 容器id	#停止当前正在运行的容器
docker kill 容器id	#强制停止当前容器
```

**查看容器**

```shell
docker logs --help
Options:
      --details        Show extra details provided to logs 
*  -f, --follow         Follow log output
      --since string   Show logs since timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
*      --tail string    Number of lines to show from the end of the logs (default "all")
*  -t, --timestamps     Show timestamps
      --until string   Show logs before a timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
     
-tf		#显示日志信息（一直更新）
--tail number #需要显示日志条数
docker logs -t --tail n 容器id #查看n行日志
docker logs -ft 容器id #跟着日志

#显示日志
[root@localhost ~]# docker logs -ft mysql
2021-04-24T16:19:20.971369562Z 2021-04-24 16:19:20+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.34-1debian10 started.
2021-04-24T16:19:21.043654501Z 2021-04-24 16:19:21+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2021-04-24T16:19:21.062144857Z 2021-04-24 16:19:21+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.34-1debian10 started.
2021-04-24T16:19:21.109705791Z 2021-04-24 16:19:21+00:00 [Note] [Entrypoint]: Initializing database files
2021-04-24T16:19:21.109736823Z 2021-04-24T16:19:21.100523Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).

```

**查看容器进程信息**

```shell
# 命令 docker top 容器id
[root@localhost ~]# docker top mysql
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
polkitd             2160                2054                0                   9月16                ?                   00:00:32            mysqld

```

**进入正在运行的容器**

```shell
# 我们通常容器都是使用后台方式运行的，需要进入容器，修改一些配置

# 命令 
# docker exec -it 容器id bashshell
[root@localhost ~]# docker exec -it mysql /bin/bash
root@4d97c6b94366:/# 

# 方式二
docker attach 容器id
#测试
docker attach 55321bcae33d 
正在执行当前的代码...
区别
#docker exec #进入当前容器后开启一个新的终端，可以在里面操作。（常用）
#docker attach # 进入容器正在执行的终端

```

**从容器内拷贝文件到宿主机**

```shell
docker cp 容器id:容器内路径   主机目的路径
#进入docker容器内部
➜  ~ docker exec -it  55321bcae33d /bin/bash 
[root@55321bcae33d /]# ls
bin  etc   lib    lost+found  mnt  proc  run   srv  tmp  var
dev  home  lib64  media       opt  root  sbin  sys  usr
#新建一个文件
[root@55321bcae33d /]# echo "hello" > java.java
[root@55321bcae33d /]# cat java.java 
hello
[root@55321bcae33d /]# exit
exit
➜  ~ docker cp 55321bcae33d:/java.java /    #拷贝
➜  ~ cd /              
➜  / ls  #可以看见java.java存在
bin   home            lib         mnt   run       sys  vmlinuz
boot  initrd.img      lib64       opt   sbin      tmp  vmlinuz.old
dev   initrd.img.old  lost+found  proc  srv       usr  wget-log
etc   java.java       media       root  swapfile  var  

```

**查看镜像的元数据**

```shell
[root@localhost ~]# docker inspect mysql
[
    {
        "Id": "4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85",
        "Created": "2021-04-24T16:19:20.422080971Z",
        "Path": "docker-entrypoint.sh",
        "Args": [
            "mysqld"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 2160,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2021-09-16T14:26:49.518997711Z",
            "FinishedAt": "2021-09-16T22:26:50.586893563+08:00"
        },
        "Image": "sha256:87eca374c0ed97f0f0b504174b0d22b0a0add454414c0dbf5ae43870369f6854",
        "ResolvConfPath": "/var/lib/docker/containers/4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85/hostname",
        "HostsPath": "/var/lib/docker/containers/4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85/hosts",
        "LogPath": "/var/lib/docker/containers/4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85/4d97c6b94366f91a3a9810677b750ddcd9f3589b3cc4399a18a47d5ed76b2e85-json.log",
        "Name": "/mysql",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": [
                "/mydata/mysql/conf:/etc/mysql",
                "/mydata/mysql/log:/var/log/mysql",
                "/mydata/mysql/data:/var/lib/mysql"
            ],
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {
                "3306/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "3306"
                    }
                ]
            },
            "RestartPolicy": {
                "Name": "always",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "host",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/9d87eb36af3727b9d7656c38ef1707fd0bb93f9f5dc347affec2f6e82003135a-init/diff:/var/lib/docker/overlay2/cbb7c48119917338af81bb5879b80599833e956374e7ad06bdfa7221f813520f/diff:/var/lib/docker/overlay2/81a94ca16caad73cc90bd466b2ef72e91f208ef6a4f305658cb57e0ebb70db35/diff:/var/lib/docker/overlay2/f16efdbe8f3d67584e8ce15298d4eab6ccaf395dfa875250e33b3492bfb284f4/diff:/var/lib/docker/overlay2/33abf2252724480d34e62fabc5b866e6a928adb5b484d9c8e0834f06390acfce/diff:/var/lib/docker/overlay2/612cbe26c808535b89a5912e4f2c408e5eb982416ebf9db29dc921226f343e80/diff:/var/lib/docker/overlay2/794da1754a236ec20376f989f94ccab2873fb65f9bb4dafe122ed6b442c37aee/diff:/var/lib/docker/overlay2/62bb0447cffa310b96b67746a0d1841af19e592806c8faee17cea84a7a8e67f1/diff:/var/lib/docker/overlay2/1ad3c3ab7994279bdb78f39f70d5f55b9e42eadffc51ea7fa9d88a21a0c26ff0/diff:/var/lib/docker/overlay2/bf8cbda5602994933085eea38ad0d5c4e32829008e98906082b78dfeeb046690/diff:/var/lib/docker/overlay2/6c799dc31fed78ee30c9836dff67ff57b6fbf8126fa53114686e01059b5de4f9/diff:/var/lib/docker/overlay2/9fe43cd250b956c75ee3a4e77859d4325a81774961eacbd9e76fdf6de3bd6b0c/diff",
                "MergedDir": "/var/lib/docker/overlay2/9d87eb36af3727b9d7656c38ef1707fd0bb93f9f5dc347affec2f6e82003135a/merged",
                "UpperDir": "/var/lib/docker/overlay2/9d87eb36af3727b9d7656c38ef1707fd0bb93f9f5dc347affec2f6e82003135a/diff",
                "WorkDir": "/var/lib/docker/overlay2/9d87eb36af3727b9d7656c38ef1707fd0bb93f9f5dc347affec2f6e82003135a/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/mydata/mysql/conf",
                "Destination": "/etc/mysql",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/mydata/mysql/data",
                "Destination": "/var/lib/mysql",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/mydata/mysql/log",
                "Destination": "/var/log/mysql",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
        "Config": {
            "Hostname": "4d97c6b94366",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "3306/tcp": {},
                "33060/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "MYSQL_ROOT_PASSWORD=root",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "GOSU_VERSION=1.12",
                "MYSQL_MAJOR=5.7",
                "MYSQL_VERSION=5.7.34-1debian10"
            ],
            "Cmd": [
                "mysqld"
            ],
            "Image": "mysql:5.7",
            "Volumes": {
                "/var/lib/mysql": {}
            },
            "WorkingDir": "",
            "Entrypoint": [
                "docker-entrypoint.sh"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "430c908a220129e69c65a95eda282d41122463b69291137ff79c0907ebbb7e79",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "3306/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "3306"
                    },
                    {
                        "HostIp": "::",
                        "HostPort": "3306"
                    }
                ],
                "33060/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/430c908a2201",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "6a9e24351415c49da65c43b79640282f7e4531f8d66b101fb9ae8a1ece8b5dd3",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.4",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:04",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "0e462979753f70100eebf26349c7950bc422bc5b3180ad2662b4c4877f2d5efc",
                    "EndpointID": "6a9e24351415c49da65c43b79640282f7e4531f8d66b101fb9ae8a1ece8b5dd3",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.4",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:04",
                    "DriverOpts": null
                }
            }
        }
    }
]

```

**总结**

基于下面这张交互图：

![](https://raw.githubusercontent.com/chengcodex/cloudimg/master/img/image-20200514214313962.png)

```shell
  attach      Attach local standard input, output, and error streams to a running container
  #当前shell下 attach连接指定运行的镜像
  build       Build an image from a Dockerfile # 通过Dockerfile定制镜像
  commit      Create a new image from a container's changes #提交当前容器为新的镜像
  cp          Copy files/folders between a container and the local filesystem #拷贝文件
  create      Create a new container #创建一个新的容器
  diff        Inspect changes to files or directories on a container's filesystem #查看docker容器的变化
  events      Get real time events from the server # 从服务获取容器实时时间
  exec        Run a command in a running container # 在运行中的容器上运行命令
  export      Export a container's filesystem as a tar archive #导出容器文件系统作为一个tar归档文件[对应import]
  history     Show the history of an image # 展示一个镜像形成历史
  images      List images #列出系统当前的镜像
  import      Import the contents from a tarball to create a filesystem image #从tar包中导入内容创建一个文件系统镜像
  info        Display system-wide information # 显示全系统信息
  inspect     Return low-level information on Docker objects #查看容器详细信息
  kill        Kill one or more running containers # kill指定docker容器
  load        Load an image from a tar archive or STDIN #从一个tar包或标准输入中加载一个镜像[对应save]
  login       Log in to a Docker registry #
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

```

#### 5.安装教程

教程：https://www.runoob.com/docker/ubuntu-docker-install.html

个人总结的docker思维导图：https://github.com/ShepherdZFJ/doc-center/tree/main/xmind

