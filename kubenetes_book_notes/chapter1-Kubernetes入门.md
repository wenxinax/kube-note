### 1.1 Kubernetes是什么
一个全新的基于容器技术的分布式架构领先方案。是谷歌十几年以来大规模应用容器技术的经验积累和升华的重要成果。
Kubenetes是谷歌严格保密十几年的密码武器——Borg的一个开源版本。

在Kubernetes中，Service是分布式集群架构的核心，一个Service对象拥有如下关键特征。
- 拥有唯一指定名称 (比如mysql-server)
- 拥有一个虚拟IP (Cluster IP、Serivce IP或VIP) 和端口号。
- 被映射到提供这种服务能力的一组容器应用上。

Service的服务进程目前都基于Socket通信方式对外提供服务，比如Redis、Memchache、MySQL、Web Server，或者是实现了某个具体业务的特定TCP Server进程。
虽然一个Service通常由多个相关的服务进程提供服务，每个服务进程都有一个独立的Endpoint(IP+Port)访问点，但Kubernetes内建的透明负载均衡和故障恢复机制，不管后端有多少服务进程，也不管某个服务进程是否发生故障而被重新部署到其他机器，都不会影响到对服务的正常调用。更重要的是，这个Service本身一旦创建就不再变化，这意味着我们再也不用为Kubernetes集群中服务的IP地址变来变去的问题而头疼了。

容器提供了强大的隔离功能，所以有必要把为Service提供服务的这组进程放入容器中进行隔离。为此，Kubernetes设计了Pod对象，将每个服务进程都包装到相应的Pod中，使其成为在Pod中运行的一个容器(Container)。
为了建立Service和Pod间的关联关系，Kubernetes首先给每个Pod都贴上一个标签(Label)，给运行MySQL的Pod贴上name=mysql标签，给运行PHP的Pod贴上name=php标签，然后给相应的Service定义标签选择器(Label Selector)，
比如MySQL Service的标签选择器的选择条件为name=mysql，意为该Service要作用于所有包含name=mysql Label的Pod。这样就解决了Serivce和Pod的关联问题。

关于Pod的概念。首先Pod运行在一个被称为节点(Node)的环境中，这个节点既可以是物理机，也可以是私有云或者公有云的一个虚拟机，通常在一个节点上运行几百个Pod；
其次，在每个Pod中都运行着一个特殊的被称为Pause的容器，其他容器则为业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷，因此他们之间的通信和数据交换更为高效，所以在设计的时候通常将一组密切相关的服务进程放入同一个Pod中；最后，并不是每个Pod和它里面运行的容器都能被映射到一个Service上，只有提供服务(无论是对内还是对外)的那组Pod才会被映射为一个服务。

集群管理方面。Kubenetes将集群中的机器划分为一个Master和一些Node。在Master上运行着集群管理相关的一组进程kube-apiserver、kube-controller-manager和kube-scheduler, 这些进程实现了整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错等管理功能，并且都是自动完成的。Node作为集群中的工作节点，运行真正的应用程序，在Node上Kubernetes管理的最小运行单位是Pod。在Node上运行着Kubernetes的Kubelet、kube-proxy服务进程，这些服务进程负责Pod的创建、启动、监控、重启、销毁，以及实现软件模式的负载均衡器。

Kubernetes中的服务扩容和服务升级。在Kubernetes集群中，只需为需要扩容的Service关联的Pod创建一个RC (Replication Controller)，服务扩容以至服务升级等问题都迎刃而解。在一个RC定义文件中包括以下3个关键信息。
- 目标Pod的定义。
- 目标Pod需要运行的副本数量(Replicas) 。
- 要监控的目标Pod的标签。

在创建好RC(系统将自动创建好Pod)后，Kubernetes会通过在RC中定义的Label筛选出对应的Pod实例并实时监控其状态和数量，如果实例数量少于定义的副本数量，则会根据在RC中定义的Pod模板创建一个新的Pod，然后将此Pod调度上合适的Node上启动运行，直到Pod实例的数量到达预定目标。服务扩容只需要修改RC中的副本数量即可。后续的服务升级也将通过修改RC来自动完成。

### 1.2 为什么要用Kubernetes

IT行业从来都是由新技术驱动的。

云计算的蓬勃发展加速了容器化技术的大规模应用。Kubernetes作为当前被业界广泛认可和看好的基于Docker的大规模容器化分布式系统解决方案，得到了以谷歌为首的IT巨头们的大力宣传和持续推进。

2015年，谷歌联合20多家公司一起建立了CNCF(Cloud Native Computing Foundation, 云原生计算基金会)开源组织来推广Kubernetes，并由此开创了云原生(Cloud Native Application)的新时代。

使用Kubernetes会收获哪些好处？
- 轻装上阵开发复杂系统。Kubernetes已经帮我们做的足够多，团队里只需要一名架构师负责系统中服务组件的架构设计，几名开发工程师负责业务代码的开发，一名系统兼运维工程师Kubernetes的部署和运维。
- 全面拥抱微服务架构。微服务架构的核心是将一个巨大的单体应用分解为很多小的互相连接的微服务，一个微服务可能有多个实例副本支撑，副本的数量可以随着系统的负荷变化进行调整。微服务架构使得每个服务都可以独立开发、升级和扩展，因此系统具备很高的稳定性和快速迭代能力，开发者也可以自由选择开发技术。谷歌将微服务架构的基础设施直接打包到Kubernetes解决方案中，让我们可以直接应用微服务架构解决复杂业务系统的架构问题。
- 可以随时随地将系统整体“搬迁”到公有云上。Kubernetes最初的设计目标就是让用户的应用运行在谷歌自家的公有云GCE中。在Kubernetes的架构方案中完全屏蔽了底层网络的细节，基于Service的虚拟IP地址(Cluster IP)的设计思路让架构与底层的硬件拓扑无关，我们无须改变运行期的配置文件，就能将系统从现有的物理机环境无缝迁移到公有云中。
- Kubernetes内在的服务弹性扩容机制可以轻松应对突发流量。在服务高峰期，可以选择在公有云中快速扩容某些Service的实例副本以提升系统的吞吐量。
- Kubernetes系统架构超强的横向扩容能力。利用Kubernetes提供的工具，不用修改代码，就能将一个Kubernetes集群从只包含几个Node的小集群平滑扩展到拥有上百个Node的大集群，甚至可以在线完成集群扩容。只要微服务架构设计的合理，就能在多个云环境中进行弹性伸缩，系统就能承受大量用户并发访问带来的巨大压力。

### 1.3 一个简单的Java Web应用例子
创建一个简单的java web应用，是一个运行在Tomcat里的Web App。此应用需要启动两个容器：Web App容器和MySQL容器，并且Web App容器需要访问MySQL容器。在Docker时代，假设我们在一个宿主机上启动了这两个容器，就需要把MySQL容器的IP地址通过环境变量注入Web App容器；同时，需要将Web App容器的8080端口映射到宿主机的8080端口，以便在外部访问。以下演示在On Kubernetes是如何实现的。

#### 1.3.1 环境准备
参考kubeasz使用ansible脚本安装K8s集群

#### 1.3.2 启动MySQL服务
为MySQL服务创建一个RC定义文件mysql-rc.yaml
```yaml
apiVersion: v1
kind: ReplicationController   # 副本控制器RC
metadata: 
  name: mysql                 # RC的名称，全局唯一
spec: 
  replicas: 1                 # 副本的期待数量
  selector:
    app: mysql                # 符合目标的Pod拥有此标签
  template:                   # 根据此模板创建Pod的副本(实例)
    metadata: 
      labels: 
        app: mysql            # Pod副本拥有的标签，对应RC的Selector
    spec: 
      containers:             # Pod内容器的定义部分
      - name: mysql           # 容器的名称
        image: hub.c.163.com/library/mysql:5.5          # 容器对应的Docker Image
        ports:
        - containerPort: 3306 # 容器应用监听的端口号
        env: 
        - name: MYSQL_ROOT_PASSWORD
          value: '123456'
```
以上的YAML定义文件中的kind属性用来表明此资源对象的类型，比如这里的值为ReplicationController，表示这是一个RC；spec是RC的相关属性定义，比如spec.selector是RC的Pod标签选择器，即监控和管理拥有这些标签的Pod实例，确保在当前集群中始终有且仅有replicas个Pod实例在运行，这里设置replicas=1，表示只能运行一个MySQL Pod实例。当在集群中运行的Pod数量少于replicas是，RC会根据spec.template一节中定义的Pod模板来生成一个新的Pod实例，spec.template.metadata.labels指定了该Pod的标签，需要特别注意的是：这里的labels必须匹配之前的spec.selector，否则RC每创建一个无法匹配Label的Pod，就会不停尝试创建新的Pod中，陷入恶性循环中。

在master上发布mysql-rc
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl create -f mysql-rc.yaml 
replicationcontroller/mysql created
```
查看刚刚创建的rc, pull image和create container需要点时间，所以可能一开始看rc的READY状态为0，等一会儿就好了。
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get rc
NAME    DESIRED   CURRENT   READY   AGE
mysql   1         1         1       21m
```
查看Pod的创建情况,Pod的创建和调度需要一点时间，所以一开始Pod的状态是Pending，成功创建之后会更新为Running。
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
mysql-jp822                  1/1     Running   0          23m
```
在Node节点查看正在运行的mysql容器。可以看到，除了MySQL Pod对应的容器还多创建了一个来自谷歌的pause容器，这就是Pod的“根容器”。详见1.4.3节的说明。
```shell
root@R740-1-3:~# docker ps | grep mysql
eb8baed4dd90        mysql                                                                  "docker-entrypoint.s…"   24 minutes ago      Up 24 minutes                           k8s_mysql_mysql-jp822_default_63de6933-b1f8-4433-8201-e7362699a660_0
653c28dbe6dd        mirrorgooglecontainers/pause-amd64:3.1                                 "/pause"                 26 minutes ago      Up 26 minutes                           k8s_POD_mysql-jp822_default_63de6933-b1f8-4433-8201-e7362699a660_0
```
最后，创建一个与之关联的Service，mysql-svc.yaml。
```yaml
apiVersion: v1
kind: Service         # 表明是Kubernetes Service
metadata:
  name: mysql         # Service的全局唯一名称
spec:
  ports:
  - port: 3306        # Service提供服务的端口号
  selector:           # Service对应的Pod拥有这里定义的标签
    app: mysql
```
其中，metadata.name是Service的服务名(ServiceName)；port属性则定义了Service的虚端口；
spec.selector确定了哪些Pod副本(实例)对应本服务。接下来通过kubectl create命令创建Service对象。
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl create -f mysql-svc.yaml 
service/mysql created
```
查看刚刚创建的Service。
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get svc | grep mysql
mysql        ClusterIP      10.68.191.59    <none>            3306/TCP       56s
```
可以看到，MySQL服务被分配了一个值为10.68.191.59的Cluster IP地址。随后，Kubernetes集群中其他新创建的Pod就可以通过Service的Cluster IP + 端口号3306来连接和访问它了。

通常，Cluster IP是在Service创建后又Kubernetes系统自动分配的，其他Pod无法预先知道某个Service的Cluster IP地址，因此需要一个服务发现机制来找到这个服务。最初，Kubernetes使用
Linux环境变量(Environment Variable)来解决这个问题。根据Service的唯一名称，容器可以从环境变量中获取Service对应的Cluster IP地址和端口，从而发起TCP/IP连接请求。

#### 1.3.3 启动Tomcat应用
创建对应的RC文件myweb-rc.yaml
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 1
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
```
创建RC。
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl create -f myweb-rc.yaml 
replicationcontroller/myweb created
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get pods
NAME                         READY   STATUS              RESTARTS   AGE
myweb-9zgll                  0/1     ContainerCreating   0          96s
myweb-ngfbd                  0/1     ContainerCreating   0          96s
```
创建对应的Service，myweb-svc.yaml。以下用NodePort的方式暴露端口到外网。
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: myweb
spec:
  type: NodePort  
  ports:
    - port: 8080
      nodePort: 30001
  selector:
    app: myweb
```
查看创建的svc
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
mysql        NodePort       10.68.64.227    <none>            3306:29929/TCP   147m
myweb        NodePort       10.68.69.205    <none>            8080:30001/TCP   147m
```
现在可以用master的IP:30001的方式访问tomcat，比如我这里是http://192.168.199.183:30001/demo/ 可以看到表格。
如果实现配置好了LoadBalancer，也可以用LoadBalancer的方式把服务暴露给外网，myweb-svc.yaml如下
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: myweb
spec:
  type: LoadBalancer  
  ports:
    - port: 80
      targetPort: 8080
  selector:
```
```shell
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl create -f myweb-svc.yaml 
service/myweb created
root@R740-2-1:~/myspace/workspace/java_web_test# kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
mysql        NodePort       10.68.64.227    <none>            3306:29929/TCP   155m
myweb        LoadBalancer   10.68.205.217   192.168.199.245   80:31457/TCP     9s
```
我这里就可以用http://192.168.199.245/demo/ 访问服务了。

#### 1.3.4 通过浏览器访问网页
http://192.168.199.245/demo/ 访问网页，能看到以下表格。
![table.png](https://upload-images.jianshu.io/upload_images/12769286-d8f33e30767c15ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以通过add按钮添加一条记录，点击submit之后就数据就被添加到MySQL数据库中了。
![add.png](https://upload-images.jianshu.io/upload_images/12769286-b5e26a72a46943a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在运行了mysql服务的node节点上登录数据库，查看数据。
```shell
root@R740-1-3:~# docker ps | grep mysql
6d5b9c339326        hub.c.163.com/library/mysql                                            "docker-entrypoint.s…"   2 hours ago         Up 2 hours                              k8s_mysql_mysql-gsq4l_default_3f2e0bae-a4b4-4df9-be43-3c853c6840c1_0
8717c683fc18        mirrorgooglecontainers/pause-amd64:3.1                                 "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_mysql-gsq4l_default_3f2e0bae-a4b4-4df9-be43-3c853c6840c1_0
root@R740-1-3:~# docker exec -it 6d5b9c339326 bash
root@mysql-gsq4l:/# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.5.56 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| HPE_APP            |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)

mysql> use HPE_APP
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-------------------+
| Tables_in_HPE_APP |
+-------------------+
| T_USERS           |
+-------------------+
1 row in set (0.00 sec)

mysql> select * from T_USERS
    -> ;
+----+------------+-------+
| ID | USER_NAME  | LEVEL |
+----+------------+-------+
|  1 | me         | 100   |
|  2 | our team   | 100   |
|  3 | HPE        | 100   |
|  4 | teacher    | 100   |
|  5 | docker     | 100   |
|  6 | google     | 100   |
|  7 | Wenxin Xie | 100   |
+----+------------+-------+
7 rows in set (0.00 sec)
```

### 1.4 Kubernetes的基本概念和术语
Kubernetes中大部分概念如Node、Pod、Replication Controller、Service等都可以被看做一种资源对象，几乎所有资源对象都可以通过Kubernetes提供的kubectl工具(或者
API编程调用)执行增删改查等操作并将其保存在etcd中持久化存储。从这个角度来看，Kubernetes其实是一个高度自动化的资源控制系统，它通过跟踪对比etcd库里保存的“资源期望状态”
与当前环境中的“实际资源状态”的差异来实现自动控制和自动纠错的高级功能。

在声明一个Kubernetes资源对象的时候，需要注意一个关键属性：apiVersion。以下面的Pod声明为例，可以看到Pod这种资源对象归属于v1这个核心API。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  labels:
    name: myweb
spec:
  containers:
  - name: myweb
    image: kuberguide/tomcat-app:v1
    ports:
    - containerPort: 8080
```
Kubernetes平台采用了“核心+外围扩展”的设计思路，在保持平台核心稳定的同时具备持续演进升级的优势。Kubernetes大部分常见的核心资源对象都归属于v1这个核心API。在版本迭代过程中，Kubernetes先后扩展了extension/v1beta1、apps/v1beta1等AOU。

我们可以采用YAML或者JSON格式声明(定义或创建)一个Kubernetes资源对象。

#### 1.4.1 Master
Kubernetes里的Master指的是集群控制节点，在每个Kubernetes集群里都需要一个Master来负责整个集群的管理和控制，基本上Kubernetes的所有控制命令都发给它，它负责具体的执行过程。高可用部署通常会设置两到三台Master独立服务器。

在Master上运行着以下关键进程。
- Kubernetes API Server(kube-apiserver): 提供了HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增删改查等操作的唯一入口，也是集群控制的入口进程。
- Kubernetes Controller Manager(kube-controller-manager)：Kubernetes里所有资源对象的自动化控制中心。
- Kubernetes Scheduler(Kube-scheduler)：负责资源调度(Pod 调度)的进程。

另外，在Master上通常还需要部署etcd服务，因为Kubernetes里所有资源对象的数据都被保存在etcd中。

#### 1.4.2 Node
除了Master, Kubernetes集群中的其他机器被称为Node，在较早的版本中也被称为Minion。Node是Kubernetes集群中的工作负载节点，会被Master分配一些工作负载(Docker 容器)，当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上。

在每个Node上都运行着以下关键进程。
- kubelet: 负责Pod对应容器的创建、启停等任务，同时与Master密切协作，实现集群管理的基本功能。
- kube-proxy：实现Kubernetes Service的通信与负载均衡机制的重要组件。
- Docker Engine：Docker引擎，负责本机的容器创建和管理工作。

Node可以在运行期间动态增加到Kubernetes集群中，前提是这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下kubelet会向Master注册自己。
一旦Node被纳入集群管理范围，kubelet进程就会定时向Master汇报自身情况，例如OS，Docker Version、CPU、Memory，以及当前那些Pod在运行，这样Master就可以获知每个Node的资源使用情况，并实现高效均衡的资源调度策略。而某个Node在超过时间不上报信息时，会被Master判定为“失联”，Node的状态被标记为不可用(Not Ready)，随后Master会触发“工作负载大转移”的自动流程。

通过kubectl get nodes查看集群有多少node。

通过kubectl describe node <node_name>查看某个Node的详细信息。

#### 1.4.3 Pod
Pod是Kubernetes最重要的基本概念，如下是Pod的组成示意图。在Pod的组成中有一个特殊的被称为“跟容器”的Pause容器。Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或多个紧密相关的用户业务容器。
![image.png](https://upload-images.jianshu.io/upload_images/12769286-7fd437f37c3ab17a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Kubernetes为每个Pod都分配了唯一的IP地址，称为Pod IP, 一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现，例如Flannel、Open vSwitch等。在Kubernetes里，一个Pod里的容器与另外主机上的Pod容器能够直接通信。

普通的Pod一旦被创建，就会被放入etcd中存储，随后会被Kubernetes Master调度到某个具体的Node上并进行binding，随后该Pod会被对应的Node上的kubelet进程实例化成一组相关的Docker容器并启动。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod(重启Pod里的所有容器)，如果Pod所在的Node宕机，就会将这个Node上的所有Pod重新调度到其他节点上。Pod、容器与Node的关系如下所示。

![image.png](https://upload-images.jianshu.io/upload_images/12769286-5bdc5d75356d39fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于Pod的定义YAML文件解释，以myweb这个Pod为例。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  labels: 
    name: myweb
spec: 
  containers:
  - name: myweb
    image: kubeguide/tomcat-app:v1
    ports:
    - containerPort: 8080
    env:
    - name: MYSQL_SERVICE_HOST
      value: 'mysql'
    - name: MYSQL_SERVICE_PORT
      value: '3306'
 ```       
 Kind表明这是一个Pod的定义，metadata里的name属性为Pod的名称，在metadata里还能定义资源对象的标签，这里声明myweb拥有一个name=myweb的标签。在Pod里所包含的容器组的定义则在spec一节中声明，这里定义了一个名为Myweb、对应镜像为kubeguide/tomcat-app:v1 的容器，该容器注入了名为MYSQL_SERVICE_HOST=‘mysql’和MYSQL_SERVICE_PORT=‘3306’的环境变量，并且在8080端口启动容器进程。Pod 的IP加上这宫里的containerPort组成了一个新的概念——Endpoint，它代表此Pod里的一个服务进程的对外通信地址。一个Pod也存在具有多个Endpoint的情况。

 在Kubernetes里，一个计算资源进行配额限定时需要设定以下两个参数。
 - Request:该资源的最小申请量，系统必须满足要求。
 - Limits: 该资源的最大允许使用量，不能被突破，否则可能会被Kubernetes“杀掉”并重启。

 以下这段定义表明MySQL容器申请最小0.25个CPU及64MiB内存，在运行过程中MySQL容器能使用的资源配额为0.5个CPU及128MiB内存。
 ```yaml
 sepc: 
   containers:
   - name: db
     image: mysql
     resources:
       requests:
         memory: "64Mi"
         cpu: "250m"
       limits: 
         memory: "128Mi"
         cpu: "500m"
```
Pod及周边对象。

![image.png](https://upload-images.jianshu.io/upload_images/12769286-82e1cbc223debdd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.4.4 Label
我们可以通过给指定资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

给某个资源对象定义一个Label，就相当于给它打了一个标签，随后可以通过Label Selector查询和筛选某些Label的资源对象。

以myweb Pod为例，Label被定义在其metadata中：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myweb
  labels:
    app: myweb
```
管理对象RC和Service则通过Selector字段设置需要管理Pod的Label。
```yaml
apiVersion: v1
kind: ReplicationController
metadata: 
  name: myweb
spec:
  replicas: 1
  slector:
    app: myweb
 ......
 
 apiVersion: v1
 kind: Service
 metadata:
   name: myweb
 spec:
   selector:
     app: myweb
     ports:
     - ports: 8080
 ```

 #### 1.4.5 Replication Controller
 RC是Kubernetes系统中的核心概念之一，声明某种Pod的副本数量在任意时刻都符合某个预期值，RC的定义包括如下部分：
 - Pod的期待数量
 - 用于筛选Pod的Label Selector
 - 当Pod的副本数量小于预期数量时，用于创建新Pod的Pod模板(template)

在运行时，可以通过修改RC的副本数量，来实现Pod的动态缩放(Scaling)，这可以通过执行kubectl scale命令来一键完成
```shell
kubectl scale rc myweb --replicas=3
```
在Kubernetes1.2中，RC被升级为一个新概念——Replica Set。支持基于集合的Label selector。
```yaml
apiVersion: extensions/v1betal
kind: ReplicaSet
metadata: 
  name: mywey
spec:
  selector:
    matchLabels: 
      app: myweb
    matchExpressions:
      - {key: app, operator: In, values: [myweb] 
  template:
  ......
```
我们当前很少单独使用Replica Set，它主要被Deployment这个更高层的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制。

#### 1.4.6 Deployment
Deployment可以看作RC的一次升级，一个最大的升级点时我们可以随时知道当前Pod的部署进度。
```yaml
# tomcat-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec: 
  replicas: 1
  selector: 
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
      spec:
        containers:
        - name: tomcat-demo
          image: tomcat
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 8080        
```

#### 1.4.7 Horizontal Pod Autoscaler
HPA与之前的RC、Deployment一样，也属于一种Kubernetes资源对象。通过追踪分析指定RC控制的所有目标Pod的负载变化情况，来确定是否需要有针对性地调整目标Pod的副本数量。当前HPA有以下两种方式作为Pod负载的度量指标。
- CPUUtilizationPercentage
- 应用程序自定义的度量指标，比如服务在每秒内的相应请求数(TPS或QPS)

HPA定义的一个例子。
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    kind: Deployment
    name: php-apache
  targetCPUUtilizationPercentage: 90
```
可以使用kubectl create创建，也可以通过kubectl autoscale deployment php-apache --cpu-percen=90--min=1--max=10 创建一个等价的HPA对象。

#### 1.4.8 StatefulSet
StatefulSet特性：
- StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名称为kafka，那么第1个Pod叫kafka-0，第2个叫kafka-1, 以此类推。
- StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod时，前n-1个Pod已经是运行且准备好的状态。
- StatefulSet里的Pod采用采用稳定的持久化存储卷，通过PV或者PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷。

StatefulSet除了要与PV卷捆绑使用以存储Pod的状态数据，还要与Headless Service配合。Headless Service相比普通的Service，没有Cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表。StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod实例都创建了一个DNS域名，格式为：
> ${podname}.$(headless service name)

#### 1.4.9 Service
Pod、RC与Service的逻辑关系。
![image.png](https://upload-images.jianshu.io/upload_images/12769286-5cdd1c10c1b5414f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Service定义了一个服务的访问入口地址，前端的应用(Pod)通过这个入口地址访问背后的一组由Pod副本组成的集群实例，Service与其后端Pod副本集群之间则是通过label Selector来实现无缝对接的。

Kubernetes微服务网格架构。
![image.png](https://upload-images.jianshu.io/upload_images/12769286-b54d64261be98cd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每个Service都被分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP，这样每个服务都变成了具备唯一IP地址的通信节点。

Kubernetes的服务发现，只要用Service的Name与Service的Cluster IP地址做一个DNS域名映射。

服务多端口定义。
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: tomcat-service
spec:
  ports:
  - port: 8080
  name: service-port
  - port: 8005
  name: shutdown-port
  selector:
    tier: frontend
```

Kubernetes里的3种IP。
- Node IP: Node的IP地址
- Pod IP： Pod的IP地址
- Cluster IP：Service的IP地址

首先，Node IP是一个真实存在的物理网络。在Kubernetes集群之外的节点访问Kubernetes集群之内的某个节点或者TCP/IP服务时，都必须通过Node IP通信。

其次，Pod IP是每个Pod的IP地址，通常是一个虚拟的二层网络。

Service的Cluster IP也是一种虚拟的IP，但更像一个“伪造”的IP网络。
- Cluster IP仅仅作用于Kubernetes Service对象，并由Kubernetes管理和分配IP地址。
- 无法被Ping。
- Cluster IP只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备TCP/IP通信的基础，而且属于Kubernetes集群的封闭空间，集群外的节点如果要访问这个端口，则需要做一些额外的工作。
-  在Kubernetes集群内，Node IP网，Pod IP网与Cluster IP网之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊路由规则，与我们熟知的IP路由有很大的不同。

既然Cluster IP只能在集群中通信，那么用户如何访问它？

采用NodePort是解决以上问题的最直接方法。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 31002
  selector: 
    tier: frontend
 ```
 绑定NodePort为31002，在浏览器访问http://<nodeIP>:31002/，就可以访问tomcat了。

 NodePort的实现方式是在Kubernetes集群里的每个Node上都为需要外部访问的Service开启了一个对应的TCP监听端口，外部系统只要用任意一个Node的IP地址+NodePort即可访问此服务。

 加负载均衡器，type=LoadBalancer，Kubernetes就会自动创建一个对应的Load Balancer实例并返回它的IP地址供外部客户端使用。

#### 1.14.10 Job
Job控制一组Pod容器。
- Job所控制的Pod副本是短暂运行的。
- Job所控制的Pod副本的工作模式能够多实例并行运算。以Tensorflow为例，可以将一个机器学习的计算任务分布到10台机器上，在每台机器上都运行一个worker执行计算任务，这很适合通过Job生成10个Pod副本同时启动运算。     

#### 1.14.11 Volume
#### 1.14.12 Persistent Volume
- PV只能是网络存储，不属于任何Node，但可以在每个Node上访问
- PV并不是被定义在Pod上的，而是独立于Pod之外定义的。

PV的YAML实例
```yaml
apiVersion: v1
kind: PersostemtVolume
metadata: 
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /somepath
    server: 172.17.0.2
```
PVC对象实例。
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```
然后在Pod的Volume定义中引用上述PVC即可。
```yaml
volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
```

#### 1.14.13 Namespace
Namespace在很多情况下用于实现多租户的资源隔离。Namespace通过将集群内部的资源对象“分配”到不同的Namespace中，形成逻辑上分组的不同项目、小组或用户组，以便不同的分组在共享使用整个集群的资源的同时还能被分别管理。