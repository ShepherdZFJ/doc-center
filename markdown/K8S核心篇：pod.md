# K8S核心篇：Pod

### 1.pod概述

Pod 是k8s 系统中可以创建和管理的最小单元，是资源对象模型中由用户创建或部署的最小资源对象模型，也是在k8s 上运行容器化应用的资源对象，其他的资源对象都是用来支撑或者扩展Pod 对象功能的，比如控制器对象是用来管控Pod 对象的，Service 或者Ingress 资源对象是用来暴露Pod 引用对象的，PersistentVolume 资源对象是用来为Pod提供存储等等，k8s 不会直接处理容器，而是Pod，Pod 是由一个或多个container 组成。

接下来，详述pod之前先讲讲创建pod用到的kubectl命令和定义pod的资源清单。

### 2.资源清单

在之前基础篇我们讲到创建一个pod的两种方式：直接使用命令和使用yaml定义资源声明式创建。在 K8S 中，一般使用 yaml 格式的文件来创建符合我们预期期望的 pod，这样的 yaml 文件我们一般称为资源清单，

#### 2.1定义资源的模式

```yaml
apiVersion: group/version　　## api资源属于的组和版本，同一个组可以有多个版本       
kind: 		##  标记创建的资源类型，
##  k8s主要支持以下资源类别：Pod,ReplicaSet,Deployment,StatefulSet,DaemonSet,Job,Cronjob    
metadata:	## 元数据
        name：	 ## 对像名称
        namespace：	## 对象所属命名空间
        labels：    ##指定资源标签，标签是一种键值数据

spec: 	 ## 定义目标资源的期望状态
status   ## 显示资源的当前状态，应该与spec 一致

```

#### 2.2常用参数

**必需属性**：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-%E8%B5%84%E6%BA%90%E6%B8%85%E5%8D%95.png)

**spec主要属性**：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-spec1)

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-spec2)

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-spec3)

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-spec4)

**其他属性**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/k8s-yaml-other)

**使用案例**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
   matchLabels:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

```

**快速编写资源yaml的方式**

```shell
# 使用kubectl create生成yaml文件
[root@shepherd-master k8s-learn]# kubectl create deployment nginx-test --image=nginx --dry-run -o yaml > fast-deploy.yaml 

# 使用kubectl get导出yaml文件
[root@shepherd-master k8s-learn]# kubectl get deployment nginx-deployment --export > deploy-export.yaml

```

### 3.kubectl

#### 3.1概述

kubectl 是Kubernetes 集群的命令行工具，通过kubectl 能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署

#### 3.2语法

```shell
kubectl [command] [TYPE] [NAME] [flags]
```

 **command**：指定要在一个或多个资源执行的操作，例如操作create，get，describe，delete。

 **TYPE**：指定资源类型，资源类型是大小写敏感的，开发者能够以单数、复数和缩略的形式，如下所示：

```shell
$ kubectl get pod pod1 
$ kubectl get pods pod1 
$ kubectl get po pod1
```

都能查出名为pod1的pod相关信息。

**NAME**：指定资源的名称(如上面的pod1)，名称也大小写敏感的。如果省略名称，则会显示所有的资源，

**flags**：指定可选的参数。例如，可用-s 或者–server 参数指定Kubernetes APIserver 的地址和端口

#### 3.3使用kubectl help

```shell
[root@shepherd-master k8s-learn]# kubectl help
kubectl controls the Kubernetes cluster manager.

 Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/

Basic Commands (Beginner):
  create        Create a resource from a file or from stdin.
  expose        使用 replication controller, service, deployment 或者 pod 并暴露它作为一个 新的 Kubernetes
Service
  run           在集群中运行一个指定的镜像
  set           为 objects 设置一个指定的特征

Basic Commands (Intermediate):
  explain       查看资源的文档
  get           显示一个或更多 resources
  edit          在服务器上编辑一个资源
  delete        Delete resources by filenames, stdin, resources and names, or by resources and label selector

Deploy Commands:
  rollout       Manage the rollout of a resource
  scale         Set a new size for a Deployment, ReplicaSet or Replication Controller
  autoscale     自动调整一个 Deployment, ReplicaSet, 或者 ReplicationController 的副本数量

Cluster Management Commands:
  certificate   修改 certificate 资源.
  cluster-info  显示集群信息
  top           Display Resource (CPU/Memory/Storage) usage.
  cordon        标记 node 为 unschedulable
  uncordon      标记 node 为 schedulable
  drain         Drain node in preparation for maintenance
  taint         更新一个或者多个 node 上的 taints

Troubleshooting and Debugging Commands:
  describe      显示一个指定 resource 或者 group 的 resources 详情
  logs          输出容器在 pod 中的日志
  attach        Attach 到一个运行中的 container
  exec          在一个 container 中执行一个命令
  port-forward  Forward one or more local ports to a pod
  proxy         运行一个 proxy 到 Kubernetes API server
  cp            复制 files 和 directories 到 containers 和从容器中复制 files 和 directories.
  auth          Inspect authorization

Advanced Commands:
  diff          Diff live version against would-be applied version
  apply         通过文件名或标准输入流(stdin)对资源进行配置
  patch         使用 strategic merge patch 更新一个资源的 field(s)
  replace       通过 filename 或者 stdin替换一个资源
  wait          Experimental: Wait for a specific condition on one or many resources.
  convert       在不同的 API versions 转换配置文件
  kustomize     Build a kustomization target from a directory or a remote url.

Settings Commands:
  label         更新在这个资源上的 labels
  annotate      更新一个资源的注解
  completion    Output shell completion code for the specified shell (bash or zsh)

Other Commands:
  alpha         Commands for features in alpha
  api-resources Print the supported API resources on the server
  api-versions  Print the supported API versions on the server, in the form of "group/version"
  config        修改 kubeconfig 文件
  plugin        Provides utilities for interacting with plugins.
  version       输出 client 和 server 的版本信息

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

```

#### 3.4kubectl命令使用分类

​    **基础命令**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/kubectl_1)

​     **部署和管理命令**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/kubectl_2)

   **故障和调试命令**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/kubectl_3)

​    **其他命令**

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/kubectl_4)

### 4.pod原理

Pod 是Kubernetes 的最重要概念，每一个Pod 都有一个特殊的被称为”根容器“的Pause容器。Pause 容器对应的镜像属于Kubernetes 平台的一部分，除了Pause 容器，每个Pod还包含一个或多个紧密相关的用户业务容器。

首先，**关于 Pod 最重要的一个事实是：它只是一个逻辑概念**。也就是说，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和 Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。

**Pod实质上是一组共享了某些资源的容器**。

Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。这样的组织关系，可以用下面这样一个示意图来表达：

![img](https://img2018.cnblogs.com/blog/1433472/201908/1433472-20190830182018230-1576158031.png)



如上图所示，这个 Pod 里有两个用户容器 A 和 B，还有一个 Infra 容器。很容易理解，在 Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。而在 Infra 容器“Hold 住”Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。所以，如果你查看这些容器在宿主机上的 Namespace 文件（这个 Namespace 文件的路径，我已经在前面的内容中介绍过），它们指向的值一定是完全一样的。infrastucture container（又叫infra）基础容器，也就是**Pause容器**。

这也就意味着，对于 Pod 里的容器 A 和容器 B 来说：

1）它们可以直接使用 localhost 进行通信；

2）它们看到的网络设备跟 Infra 容器看到的完全一样；

3）一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；

4）当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；

5）Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。

**共享 Volume** 就简单多了：Kubernetes 项目只要把所有 Volume 的定义都设计在 Pod 层级即可。这样，一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录

你现在可以这么理解 Pod 的本质：Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。

所以下一次，当你需要把一个运行在虚拟机里的应用迁移到 Docker 容器中时，一定要仔细分析到底有哪些进程（组件）运行在这个虚拟机里。然后，你就可以把整个虚拟机想象成为一个 Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 **Init Container**。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式

在 Pod 中，所有 **Init Container** 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

### 5.pod生命周期

Pod 生命周期的变化，主要体现在 Pod API 对象的 Status 部分，这是它除了 Metadata 和 Spec 之外的第三个重要字段。其中，pod.status.phase，就是 Pod 的当前状态，它有如下几种可能的情况：

**Pending**：这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。

**Running**：这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。

**Succeeded**：这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。

**Failed**：这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。

**Unknown**：这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

注意，Pod（即容器）的状态是 Running，并不代表应用能正常提供服务，如以下几种情况：

1）程序本身有 bug，本来应该返回 200，但因为代码问题，返回的是500；

2）程序因为内存问题，已经僵死，但进程还在，但无响应；

3）Dockerfile 写的不规范，应用程序不是主进程，那么主进程出了什么问题都无法发现；

4）程序出现死循环。

