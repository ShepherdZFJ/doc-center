# docker高级篇：镜像原理和Dockerfile

### 1.docker镜像是什么

镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。

### 2.docker镜像加载原理

**UnionFS(联合文件系统)**

UnionFS（联合文件系统）：**Union文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统**，他支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下（ unite several directories into a single virtual filesystem)。Union文件系统是 Docker镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像
特性：**一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录**

**镜像加载原理**

docker的镜像实际上由一层一层的文件系统组成，这种层级的文件系统UnionFS。

bootfs(boot file system)主要包含bootloader和kernel, bootloader主要是引导加载kernel, Linux刚启动时会加载bootfs文件系统，在Docker镜像的最底层是bootfs。这一层与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

rootfs (root file system) ，在bootfs之上。包含的就是典型 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。

<img src="https://img-blog.csdnimg.cn/20200827085820785.png" style="zoom: 50%;" />

平时我们安装进虚拟机的CentOS都是好几个G，为什么docker这里才200M？

```shell
[root@VM-4-10-centos ~]# docker images centos
REPOSITORY   TAG       IMAGE ID       CREATED      SIZE
centos       latest    5d0da3dc9764   6 days ago   231MB
```

对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供 rootfs 就行了。由此可见对于不同的linux发行版, bootfs基本是一致的, rootfs会有差别, 因此不同的发行版可以公用bootfs

**镜像使用分层结构的好处**

最大的好处，我觉得莫过于资源共享了！比如有多个镜像都从相同的Base镜像构建而来，那么宿主机只需在磁盘上保留一份base镜像，同时内存中也只需要加载一份base镜像，这样就可以为所有的容器服务了，而且镜像的每一层都可以被共享。例如mysql每个版本的改动都是一下层，但是公共部分的层都是一样，这样我们在下载mysql5.7版本之后再去下载mysql最新版时很多层就已经存在，所以不需要再下载已有的层。

下载mysql:5.7

```bash
[root@VM-4-10-centos ~]# docker pull mysql:5.7
5.7: Pulling from library/mysql
a330b6cecb98: Pull complete 
9c8f656c32b8: Pull complete 
88e473c3f553: Pull complete 
062463ea5d2f: Pull complete 
daf7e3bdf4b6: Pull complete 
1839c0b7aac9: Pull complete 
cf0a0cfee6d0: Pull complete 
fae7a809788c: Pull complete 
dae5a82a61f0: Pull complete 
7063da9569eb: Pull complete 
51a9a9b4ef36: Pull complete 
Digest: sha256:d9b934cdf6826629f8d02ea01f28b2c4ddb1ae27c32664b14867324b3e5e1291
Status: Downloaded newer image for mysql:5.7
docker.io/library/mysql:5.7
```

下载mysql:latest

```shell
[root@VM-4-10-centos ~]# docker pull mysql
Using default tag: latest
latest: Pulling from library/mysql
a330b6cecb98: Already exists 
9c8f656c32b8: Already exists 
88e473c3f553: Already exists 
062463ea5d2f: Already exists 
daf7e3bdf4b6: Already exists 
1839c0b7aac9: Already exists 
cf0a0cfee6d0: Already exists 
1b42041bb11e: Pull complete 
10459d86c7e6: Pull complete 
b7199599d5f9: Pull complete 
1d6f51e17d45: Pull complete 
50e0789bacad: Pull complete 
Digest: sha256:99e0989e7e3797cfbdb8d51a19d32c8d286dd8862794d01a547651a896bcf00c
Status: Downloaded newer image for mysql:latest
docker.io/library/mysql:latest

```

通过上面可知拉取mysql最新版镜像时很多分层提示Already exists，不需要再下载了。

同时删除镜像时，如果一个镜像有多个版本，那么只会删除这个版本特有的层，最终只有一个版本时才会删除所有层。如下所示：

删除mysql:latest

```shell
[root@VM-4-10-centos ~]# docker rmi mysql:latest
Untagged: mysql:latest
Untagged: mysql@sha256:99e0989e7e3797cfbdb8d51a19d32c8d286dd8862794d01a547651a896bcf00c
Deleted: sha256:0716d6ebcc1a61c5a296fcb187e71f93531e510d4e4400267e2e502103d0194c
Deleted: sha256:dc895a08d34b5b81fc4ca087d2ad52cbe1a2050e249040a22c5f2eabf2f384ba
Deleted: sha256:660229dcf1a452460127a498b9f3f161e7ca94507353ded8af92fe9ab55a32ed
Deleted: sha256:6b26fa2fc4e2150aee2f2557bcbfaf727c00d1650ea08d8ed3fe7c8a6caaa88b
Deleted: sha256:c20303553d5d2594e1655000089b12eca8db7afdcb068cc35fc47ebfe3dab5fb
Deleted: sha256:77a3d69619bfea7b30831a41a32bbf61756c9f95513743deea8daa9a83ff2646
```

删除mysql:5.7(删除所有层)

```shell
[root@VM-4-10-centos ~]# docker rmi mysql:5.7
Untagged: mysql:5.7
Untagged: mysql@sha256:d9b934cdf6826629f8d02ea01f28b2c4ddb1ae27c32664b14867324b3e5e1291
Deleted: sha256:1d7aba9171693947d53f474014821972bf25d72b7d143ce4af4c8d8484623417
Deleted: sha256:94ebbead5c58282fef91cc7d0fb56e4006a72434b4a6ae2cd5be98f369cb8c21
Deleted: sha256:989da5efad29ec59bd536cd393d277bc777f8b9b34b8e3ad9593a4b0a83b40f4
Deleted: sha256:7457ee6817c678da3cb383d27a3d79d5f3f25fbcb92958d5e8d5709e7631e23c
Deleted: sha256:fe7dac53adebe33519b4e4fc577bfcddd7372cc313c35fae681fc82fb325fdc0
Deleted: sha256:9578f1c7f00f400b3f71be0ee721cbc0892e05e454323e1a74a6e56ae1dafdab
Deleted: sha256:335f9f9fbbd8977530806ed5439a2b67f1c06117f752a2598698de4ae304c516
Deleted: sha256:e15ed274d47a7d6ddff0afcc628143254c69128a9d2379900ebb519e7c6c2bce
Deleted: sha256:51930b767631b583738d75519bed2a8cc757c5b0c904617972386462deee2ca7
Deleted: sha256:43bd682fb659113a8ab168032d8f82dee86d2cee5cee2e146af6c3a6f9ccef18
Deleted: sha256:1957f1873568b423369e0299de6c9b75a111fea807b0c07506ba45d075ca8f80
Deleted: sha256:d000633a56813933cb0ac5ee3246cf7a4c0205db6290018a169d7cb096581046

```

查看镜像分层的方式可以通过docker image inspect 命令

**Docker 镜像都是只读的，当容器启动时，一个新的可写层加载到镜像的顶部，这一层就是我们通常说的容器层，容器之下的都叫镜像层**

### 3.容器数据卷

如果数据都存储在容器中，那么容器删除之后数据就丢失了，有点和redis没有持久化服务重启之后数据没有了，那么假如我们有一个mysql的容器，容器被删除之后数据库数据就丢失那肯定是不能被接受的。容器数据卷能解决这个问题，容器之间可以有一个数据共享的技术，Docker容器中产生的数据，同步到本地，这就是容器卷技术，目录的挂载，将我们容器内的目录，挂载到Linux上面。

卷就是目录或文件，存在于一个或多个容器中，由docker挂载到容器，但不属于联合文件系统，因此能够绕过Union File System提供一些用于持续存储或共享数据的特性： 

 卷的设计目的就是数据的持久化，完全独立于容器的生存周期，因此Docker不会在容器删除时删除其挂载的数据卷

**特点：**

1）：数据卷可在容器之间共享或重用数据

2）：卷中的更改可以直接生效

3）：数据卷中的更改不会包含在镜像的更新中

4）：数据卷的生命周期一直持续到没有容器使用它为止

**使用命令直接挂载docker run -v**

```shell

#   docker run -it -v 主机目录:容器内目录  -p 主机端口:容器内端口
➜ ~ [root@shepherd-master ~]# docker run -p 3306:3306 --name mysql \
> -v /mydata/mysql/log:/var/log/mysql \
> -v /mydata/mysql/data:/var/lib/mysql \
> -v /mydata/mysql/conf:/etc/mysql \
> -e MYSQL_ROOT_PASSWORD=root \
> -d mysql:5.7
# -p 3306:3306：将容器的3306端口映射到主机的3306端口
# --name：给容器命名
# -v /mydata/mysql/log:/var/log/mysql：将配置文件挂载到主机/mydata/..
# -e MYSQL_ROOT_PASSWORD=root：初始化root用户的密码为root


```

即时我们**删除mysql容器，挂载到本地的数据卷依旧没有丢失，这就实现了容器数据持久化功能**

通过 docker inspect 容器id 查看容器详细信息，这时候就可以看到目录挂载信息如下：

```shell
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
        ]

```

测试挂载同步：

```shell
[root@VM-4-10-centos /]# docker run -it -d -v /test_volume/testData:/volume/data centos /bin/bash
6dfa357c04c1d6dd3d4db54f601ee318c373144f0eec48bb3bd617374e102501
[root@VM-4-10-centos /]# ls
bin  boot  data  dev  etc  home  learn_volume  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  shepherd  srv  sys  test_volume  tmp  usr  var
[root@VM-4-10-centos /]# cd test_volume
[root@VM-4-10-centos test_volume]# ls
testData
# 在宿主机测试上级目录是否会同步(否)
[root@VM-4-10-centos test_volume]#  mkdir haha 
[root@VM-4-10-centos test_volume]# ls
haha  testData
[root@VM-4-10-centos test_volume]# cd testData
[root@VM-4-10-centos testData]# ls
# 在宿主机创建文件，测试同步容器
[root@VM-4-10-centos testData]# touch hello.java
[root@VM-4-10-centos testData]# ls
hello.java


# 进入容器
[root@VM-4-10-centos learn_volume]# docker exec -it 6dfa357c04c1 /bin/bash
[root@6dfa357c04c1 /]# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  volume
[root@6dfa357c04c1 /]# cd volume 
[root@6dfa357c04c1 volume]# ls
data
[root@6dfa357c04c1 volume]# cd data
# 查看文件是否同步过来
[root@6dfa357c04c1 data]# ls
hello.java

# 停止容器
[root@VM-4-10-centos learn_volume]# docker stop 6dfa357c04c1
6dfa357c04c1
# 往宿主机挂载目录写文件
[root@VM-4-10-centos testData]# echo "after container stop, add a file in host machine, test start container again file sync is or not?" >>container_stop_after.txt
[root@VM-4-10-centos testData]# ls
container_stop_after.txt  hello.java  inner.txt
# 重启容器
[root@VM-4-10-centos learn_volume]# docker start 6dfa357c04c1
6dfa357c04c1
# 进入容器
[root@VM-4-10-centos learn_volume]# docker exec -it 6dfa357c04c1 /bin/bash
[root@6dfa357c04c1 /]# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  volume
[root@6dfa357c04c1 /]# cd volume/data
[root@6dfa357c04c1 data]# ls
container_stop_after.txt  hello.java  inner.txt
[root@6dfa357c04c1 data]# cat container_stop_after.txt
after container stop, add a file in host machine, test start container again file sync is or not?

# 在容器内增删改文件，也会同步到宿主机对应的目录文件，这里就不示例了

```

通过以上说明**停止容器之后，宿主机修改文件，再次启动容器，容器内的数据依旧是同步的。**

**具名和匿名挂载**

```shell
# 匿名挂载
-v 容器内路径
docker run -d -P --name nginx01 -v /etc/nginx nginx

# 查看所有的volume的情况
➜  ~ docker volume ls    
DRIVER              VOLUME NAME
local               33ae588fae6d34f511a769948f0d3d123c9d45c442ac7728cb85599c2657e50d
local            
# 这里发现，这种就是匿名挂载，我们在 -v只写了容器内的路径，没有写宿主机挂载路径！

# 具名挂载
➜  ~ docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx
➜  ~ docker volume ls                  
DRIVER              VOLUME NAME
local               juming-nginx

# 通过 -v 卷名：容器内路径
# 查看一下这个卷
[root@VM-4-10-centos /]# docker inspect juming-nginx
[
    {
        "CreatedAt": "2021-09-27T16:48:27+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/juming-nginx/_data",
        "Name": "juming-nginx",
        "Options": null,
        "Scope": "local"
    }
]
[root@VM-4-10-centos /]# docker inspect 00523aaa42f830f62a5f5c162ffb34d6cf33c05b383bfe9c7f2dd70b3e265399
[
    {
        "CreatedAt": "2021-09-27T16:45:06+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/00523aaa42f830f62a5f5c162ffb34d6cf33c05b383bfe9c7f2dd70b3e265399/_data",
        "Name": "00523aaa42f830f62a5f5c162ffb34d6cf33c05b383bfe9c7f2dd70b3e265399",
        "Options": null,
        "Scope": "local"
    }
]


```

**所有的docker容器内的卷，没有指定目录的情况下都是在`/var/lib/docker/volumes/xxxx/_data`下如果指定了目录，docker volume ls 是查看不到的**

```shell
# 三种挂载： 匿名挂载、具名挂载、指定路径挂载
-v 容器内路径			#匿名挂载
-v 卷名：容器内路径		#具名挂载
-v /宿主机路径：容器内路径 #指定路径挂载 docker volume ls 是查看不到的

# 通过 -v 容器内路径： ro rw 改变读写权限
ro #readonly 只读
rw #readwrite 可读可写
docker run -d -P --name nginx05 -v juming:/etc/nginx:ro nginx
docker run -d -P --name nginx05 -v juming:/etc/nginx:rw nginx

# ro 只要看到ro就说明这个路径只能通过宿主机来操作，容器内部是无法操作！

```

**容器之间数据共享**

```shell
--volumes-from list              Mount volumes from the specified container(s)

docker run -it -d --volumes-from c945db0a9bda 843f4373e7c7 /bin/bash
# c945db0a9bda：父容器id，这里可以绑定多个，list
# 843f4373e7c7：镜像id
# 这时候新起来的容器就和容器c945db0a9bda数据卷内容共享了，即时其中一个容器删除了，另外的容器数据也不会丢失
```

容器之间的配置信息的传递，数据卷容器的生命周期一直持续到没有容器使用为止。但是一旦你持久化到了本地，这个时候，本地的数据是不会删除的。

### 4.Dockerfile

`dockerfile`是用来构建docker镜像的文件的，是由一系列的命令和参数构成的脚本，其使用步骤如下：

1） 编写一个dockerfile文件

2） docker build 构建称为一个镜像

3） docker run运行镜像

4） docker push发布镜像（DockerHub 、阿里云仓库)

**`dockerfile`内容特点**：

1）每个保留关键字(指令）都是必须是大写字母

2）执行从上到下顺序

3）#表示注释

4）每一个指令都会创建提交一个新的镜像层，并提交！

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL2NoZW5nY29kZXgvY2xvdWRpbWcvbWFzdGVyL2ltZy9pbWFnZS0yMDIwMDUxNjEzMTc1Njk5Ny5wbmc?x-oss-process=image/format,png)

从应用软件的角度来看，Dockerfile、Docker镜像与Docker容器分别代表软件的三个不同阶段，

\*  Dockerfile是软件的原材料

\*  Docker镜像是软件的交付品

\*  Docker容器则可以认为是软件的运行态。

Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。

![docker11](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/docker11.png)

**dockerfile命令**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/dockerfile%E6%96%87%E4%BB%B6%E5%91%BD%E4%BB%A4)

示例如下：

```shell
# 1.编写dockerfile
[root@VM-4-10-centos docker-file]# vim Dockerfile01
FROM centos
MAINTAINER cheng<1204598429@qq.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80

CMD echo $MYPATH
CMD echo "-----end----"
CMD /bin/bash

# 2.根据dockerfile文件生成docker镜像
[root@VM-4-10-centos docker-file]# docker build -f /docker-file/Dockerfile01 -t shepherd/mycentos:v1 .

# 3.运行镜像并进入查询当前工作目录
[root@VM-4-10-centos docker-file]# docker run -it shepherd/mycentos:v1
[root@95c831d011fc local]# pwd
/usr/local


#测试cmd命令，流程和上面一样，编写dockerfile文件，构建镜像，运行容器
[root@VM-4-10-centos docker-file]# vi Dockerfile-test-cmd
FROM centos
CMD ["ls","-a"]

[root@VM-4-10-centos docker-file]# docker build -f /docker-file/Dockerfile-test-cmd -t shepherd/test-cmd:v1 .

[root@VM-4-10-centos docker-file]# docker run -it shepherd/test-cmd:v1
.   .dockerenv  dev  home  lib64       media  opt   root  sbin  sys  usr
..  bin         etc  lib   lost+found  mnt    proc  run   srv   tmp  var
#从上面可以运行容器是执行ls -a命令

#指定命令运行报错  cmd的情况下 -l 替换了CMD["ls","-l"] -l 不是命令所有报错
[root@VM-4-10-centos docker-file]# docker run -it shepherd/test-cmd:v1 -l
docker: Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "-l": executable file not found in $PATH: unknown.
ERRO[0000] error waiting for container: context canceled 

#ENTRYPOINT指令 使用ENTRYPOINT指令上面的就可以正常追加命令
类似于 CMD 指令，但其不会被 docker run 的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序。
但是, 如果运行 docker run 时使用了 --entrypoint 选项，将覆盖 CMD 指令指定的程序。
优点：在执行 docker run 的时候可以指定 ENTRYPOINT 运行所需的参数。
注意：如果 Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效

# RUN指令 是在 docker build， 而 CMD指令是在docker run

```

**发布镜像**

```shell
# 1.登录账户
[root@VM-4-10-centos docker-file]# docker login -u shepherdzfj
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

# 2.提交镜像
[root@VM-4-10-centos docker-file]# docker images
REPOSITORY           TAG       IMAGE ID       CREATED          SIZE
shepherd/test-cmd    v1        6109c4d016c1   19 minutes ago   231MB
shepherd/mycentos    v1        8fe3b81e9e88   39 minutes ago   336MB
shepherdzfj/centos   v1        843f4373e7c7   47 hours ago     231MB
centos               latest    5d0da3dc9764   13 days ago      231MB
tomcat               8         551efb60a0a2   2 weeks ago      678MB
nginx                latest    ad4c705f24d3   2 weeks ago      133MB
mysql                5.7       1d7aba917169   3 weeks ago      448MB
[root@VM-4-10-centos docker-file]# docker push shepherdzfj/centos:v1
74ddd0ec08fa: Mounted from library/centos 
v1: digest: sha256:7d11239479e896f7a8a5d67ab834d3bd06ded6b7d5d06a5a74c793dd6ab38c9a size: 529

```

### 5.docker网络原理

我们每启动一个docker容器，docker就会给docker容器分配一个ip，我们只要安装了docker，就会有一个docker0桥接模式，使用的技术是veth-pair技术

```shell
# 查看宿主机网络
[root@VM-4-10-centos docker-file]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:00:fa:9e:9b brd ff:ff:ff:ff:ff:ff
    inet 10.0.4.10/22 brd 10.0.7.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fefa:9e9b/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:5f:e5:51:75 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:5fff:fee5:5175/64 scope link 
       valid_lft forever preferred_lft forever
9: veth65b897b@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 7e:5c:8f:81:b4:b4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::7c5c:8fff:fe81:b4b4/64 scope link 
       valid_lft forever preferred_lft forever
21: veth9b0ce49@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether a2:88:1e:f1:16:55 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::a088:1eff:fef1:1655/64 scope link 
       valid_lft forever preferred_lft forever
23: veth4c98de6@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 9e:00:ff:d0:23:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::9c00:ffff:fed0:23b7/64 scope link 
       valid_lft forever preferred_lft forever
25: veth04ff936@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether aa:0c:53:e4:fc:87 brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::a80c:53ff:fee4:fc87/64 scope link 
       valid_lft forever preferred_lft forever
       
 # 查看容器网络     
[root@VM-4-10-centos ~]# docker exec -it 032c0b708b12 /bin/bash
[root@032c0b708b12 /]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
24: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:06 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.6/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever


```

