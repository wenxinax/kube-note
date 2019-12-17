On Kubenetes无论何种类型的Volume和使用它的Pod都是一种静态绑定关系，在Pod定义文件中，同时定义了它使用的Volume。在这种情况下，Volume是Pod的附属品，我们无法像创建其他资源(例如Pod, Node, Deployment等等)一样创建一个Volume。

因此Kubernetes提出了PersistentVolume(PV)的概念。PersistentVolume和Volume一样，代表了集群中的一块存储区域，然而Kubernetes将PersistentVolume抽象成了一种集群资源，类似于集群中的Node对象，这意味着我们可以使用Kubernetes API来创建PersistentVolume。PV与Volume最大的不同是PV拥有着独立于Pod的生命周期。

而PersistentVolumeClaim(PVC)代表了用户对PV资源的请求。用户需要使用PV资源时，只需要创建一个PVC对象(包括指定使用何种存储资源， 使用多少GB，以何种模式使用PV等信息)，Kubernetes会自动为我们分配我们所需的PV。如果把PersistentVolume类比成集群中的Node，那么PersistentVolumeClaim就相当于集群中的Pod，Kubernetes为Pod分配可用的Node，为PersistenVolumeClaim分配可用的PersistentVolume。

## 1. PV的静态创建
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```
PV的访问模式(accessModes)有三种：
- ReadWriteOnce(RWO):是最基本的方式，可读可写，但只支持被单个Pod挂载。
- ReadOnlyMany(ROX):可以以只读的方式被多个Pod挂载。
- ReadWriteMany(RWX):这种存储可以以读写的方式被多个Pod共享。

> 不是每一种PV都支持这三种方式，例如ReadWriteMany，目前支持的还比较少。在PVC绑定PV时通常根据两个条件来绑定，一个时存储的大小，另一个就是访问模式。

PV的回收策略(persistentVolumeReclaimPolicy, 即PVC释放卷的时候PV该如何操作)也有三种：
- Retain，不清理，保留Volume(需要手动清理)
- Recycle，删除数据，即rm -rf /thevolume/* (只有nfs和hostPath支持)
- Delete，删除存储资源，比如删除AWS EBS卷(只有AWS EBS, GCE PD, Azure Disk和Cinder支持)

> PVC释放卷是指用户删除一个PVC对象时，那么与该PVC对象绑定的PV就会被释放。

### 1.1 PV支持的类型

定义PV时，我们需要指定其底层存储的类型，例如上文中创建的PV，底层使用nfs存储。目前Kubernetes支持以下类型：
- GCEPersistentDisk
- AWSElasticBlockStore
- AzureFile
- AzureDisk
- FC (Fibre Channel)**
- FlexVolume
- Flocker
- NFS
- iSCSI
- RBD (Ceph Block Device)
- CephFS
- Cinder (OpenStack block storage)
- Glusterfs
- VsphereVolume
- Quobyte Volumes
- HostPath (Single node testing only – local storage is not supported in any way and WILL NOT WORK in a multi-node cluster)
- VMware Photon
- Portworx Volumes
- ScaleIO Volumes
- StorageOS

## 2. PVC的创建
当我们定义好一个PV后，我们希望像使用Volume那样使用这个PV，那么我们需要做的就是创建一个PVC对象，并在Pod定义中使用这个PVC。

定义一个PVC：
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
```
Pod通过挂载Volume的方式应用PVC：
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
PVC文件的关键定义：
- storageClassName: slow 用于绑定PVC和PV。表明这个PVC希望使用storageClassName=slow的PV。返回到上文中PV的定义，也包含了storageClassName=slow的配置。
- accessModes: ReadWriteOnce 表明这个PV希望使用storageClassName=slow 并且accessModes=ReadWriteOnce的PV。
- 在上述条件都满足后，PVC还可以指定PV必须满足的Label，如matchLabels: realse: "stable"。这表明此PVC希望使用storageClassName=slow, accessModes=ReadWriteOnce并且拥有Label: realse: "stable"的PV。
- storage: 8Gi 这表明此PVC希望使用8G的Volume资源。

> 综合上面的分析，我们可以看到PVC和PV的绑定，不是简单的通过Label来进行，而是要综合storageClassName， accessModes, matchLabels以及storage来绑定。

## 3. PV的动态创建
上文通过persistentVolume描述文件创建了一个PV，这样的创建方式称为静态创建。静态创建方式有一个弊端，那就是
假如我们创建PV时指定大小为50G，而PVC请求80G的PV，那么此PVC就无法找到合适的PV来绑定。因此产生了PV的动态创建。

PV的动态创建依赖于StorageClass对象。不需要手动创建任务PV，所有的工作都有StorageClass完成。一个例子如下:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  zones: us-east-1d, us-east-1c
  iopsPerGB: "10"
```
这个例子使用AWS提供的插件(kubernetes.io/aws-ebs)创建了一个基于AWS底层存储的StorageClass。这意味着使用这个StorageClass，那么所有的PV都是AWSElasticBlockStore类型。

StorageClass的定义包含四个部分：
- provisioner: 指定Volume插件的类型，包括内置插件(如kubernetes.io/aws-ebs)和外部插件(如external-storage提供的ceph.com/cephfs)。
- mountOptions：指定挂载选项，当PV不支持指定的选项时会直接失败。比如NFS支持hard和nfsver=4.1等选项。
- parameters：指定provisioner的选项，比如kubernetes.io/aws-ebs支持type、zone、iopsPerGB等参数。
- reclaimPolicy

手动创建PV时，我们指定了storageClassName=slow的配置像，然后早PVC定义中也指定storageClassName=slow，从而完成绑定。而通过storageClass实现动态PV时，我们只需要指定StorageClass的metadata.name。

回到上文中创建PVC的例子，PVC指定了storageClassName=slow。那么Kubernetes会在集群中寻找是否存在metadata.name=slow的StorageClass，如果存在，此StorageClass会自动为此PVC创建一个accessModes=ReadWriteOnce，并且大小为8GB的PV。

> 通过StorageClass的使用，解放了提前构建静态PV池的工作。

目前StorageClass支持如下类型:
![StorageClass类型](https://upload-images.jianshu.io/upload_images/448235-563b3a7701a5736a.PNG "StorageClass类型")

## 4. PV的生命周期

PV的生命周期包括5个阶段：
- Provisioning，即PV的创建，可以直接创建PV(静态方式)，也可以使用StorageClass动态创建。
- Binding，将PV分配给PVC
- Using，Pod通过PVC使用该Volume，并可以通过准入控制StorageProtection(1.9及以前版本为PVCProtection)阻止删除正在使用的PVC。
- Releasing，Pod释放Volume并删除PVC。
- Reclaiming，回收PV，可以保留PV以便下次使用，也可以直接从云存储中删除
- Deleting，删除PV并从云存储中删除后端存储

根据这5个阶段，PV的状态有以下4种
- Available：可用
- Bound：已经分配给PVC
- Released：PVC解绑但还未执行回收策略
- Failed：发生错误

一个PV从创建到销毁的具体流程如下:

1. 一个PV创建完后状态会变成Available，等待被PVC绑定。
2. 一旦被PVC绑定，PV的状态会变成Bound，就可以被定义了相应PVC的Pod使用。
3. Pod使用完后会释放PV，PV的状态变成Released。
4. 变成Released的PV会根据定义的回收策略做相应的回收工作。有三种回收策略，Retain、Delete和Recycle。Retain就是保留现场，K8S什么也不做，等待用户手动去处理PV里的数据，处理完后，再手动删除PV。Delete策略，K8S会自动删除该PV及里面的数据。Recycle方式，K8S会将PV里的数据删除，然后把PV的状态变成Available，又可以被新的PVC绑定使用。

## 5. DefaultStorageClass

前面说到，PVC和PV的绑定时通过StorageClassName进行的。然而如果定义PVC时没有指定StorageClassName呢？这取决于与admission插件是否开启了DefaultDefaultStorageClass功能：
- 如果DefaultDefaultStorageClass功能开启，那么此PVC的StorageClassName就会被指定为DefaultStorageClass。DefaultStorageClass从何而来呢？在定义StorageClass时，可以在Annotation中添加一个键值对：storageclass.kubernetes.io/is-dafult-class: true，那么此StorageClass就会变成默认的StorageClass了。
- 如果DefaultDefaultStorageClass功能没有开启，那么没有指定StorageClassName的PVC只能被绑定到同样没有指定StorageClassName的PV。




























