## Volume
 A Kubernetes volume, on the other hand, has an explicit lifetime - the same as the Pod that encloses it. Consequently, a volume outlives any Containers that run within the Pod, and data is preserved across Container restarts. Of course, when a Pod ceases to exist, the volume will cease to exist, too. 

 ### types of volume
 #### emptyDir
  一个emptyDir volume是在Pod分配到Node时创建的。顾名思义，它的初始内容为空，在同一个Pod中的所有容器均可以读写这个emptyDir volume。当 Pod 从 Node 上被删除（Pod 被删除，或者 Pod 发生迁移），emptyDir 也会被删除，并且数据永久丢失。
  ```yaml
apiVersion: v1
kind: Pod
metadata:
 name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
 ```
 emptyDir类型的volume适合于以下场景
 - 临时空间
 - 一个容器需要从另一容器中获取数据的目录(多容器共享目录)

 #### hostPath
 hostPath类型的volumn允许用户挂在Node上的文件系统到Pod中，如果Pod需要使用Node上的文件，可以使用hostPath。
 ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```
hostPath支持type属性，可选值如下:
Value | Behavior
-|-
&nbsp;| Empty string(default) is for backward compatibility, which means that no checks will be performed before mounting the hostPath volume. 
DirectoryOrCreate | If noting exists at the given path, an empty directory will be created there as needed with permission set to 0755, having the same group and ownership with Kubelet.
Directory | A directory must exist at the given path.
FileOrCreate | If noting exists at the given path, an empty file will be created there as needed wht permission set to 0644. having the same group and ownership with Kubelet.
File | A file must exist at the given path.
Socket | A UNIX socket must exist at the given path.
CharDevice | A character device must exist at the given path.
BlockDevice | A block device must exist at the given path. 

hostPath volume通常用于以下场景：
- 容器中的应用程序产生的日志文件需要永久保存，可以使用宿主机的文件系统进行存储。
- 需要访问宿主机上Docker引擎内部数据结构的容器应用，通过定义hostPath为/var/lib/docker目录，使容器内应用可以直接访问Docker的文件系统

在使用hostPath volume时，需要注意：
- 在不同的Node上具有相同配置的Pod，可能会因为宿主机上的目录和文件不同，而导致对Volume上目录和文件的访问结果不一致。

#### gcePersistentDisk
gcePersistentDisk可以挂在在GCE(Google Compute Engine) 上的永久磁盘到容器， 需要Kubernetes运行在GCE的VM中，需要注意的是，你必须先创建一个永久磁盘(Persistent Disk, PD)才能使用gcePersistentDisk volume。
```yaml
volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

#### awsElasticBlockStore
与gcePersistentDisk类似，awsElasticBlockStore使用Amazon Web Service(AWS)提供的EBS Volume， 挂载到Pod中。需要Kubernetes运行在AWS的EC2上。另外，需要先创建一个EBS Volume才能使用awsElasticBlockStore。
```yaml
volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

#### nfs
NFS是Network File System的缩写，即网络文件系统。Kubernetes中通过简单的配置就可以挂在NFS到Pod中，而NFS中的数据是可以永久保存的，同时NFS支持同时写操作。
```yaml
volumes:
- name: nfs
  nfs:
    # FIXME: use the right hostname
    server: 10.254.234.223
    path: "/"
```

#### gitRepo
通过挂载一个空目录，并从Git仓库克隆一个repository供Pod使用
```yaml
 volumes:
  - name: git-volume
    gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```

#### 使用subPath

Pod的多个容器使用同一个Volume时，subPath非常有用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```