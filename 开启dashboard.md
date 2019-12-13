## dashboard
kebuasz部署的时候已经带好了dashboard
### 验证
```SHELL
# 查看pod运行状态
kubectl get pod -n kube-system |  grep dashboard
kubernetes-dashboard-5c7687cf8-dfs2q          1/1     Running   0          41h
# 查看dashboard service
kubectl get svc -n kube-system|grep dashboard
kubernetes-dashboard      NodePort    10.68.194.93    <none>        443:35837/TCP                 41h
# 查看集群服务
kubectl cluster-info|grep dashboard
kubernetes-dashboard is running at https://192.168.199.181:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
# 查看pod运行日志
kubectl logs kubernetes-dashboard-5c7687cf8-dfs2q -n kube-system
```

### 访问控制
跟着kubeasz部署的项目是自带访问控制的，关闭了apiserver非安全端口8080的外部访问--insecure-bind-address=127.0.0.1
新版 dashboard可以有多层访问控制，首先与旧版一样可以使用apiserver 方式登录控制：

- 第一步通过api-server本身安全认证流程，与之前1.6.3版本相同，这里不再赘述
如需（用户名/密码）认证，kubeasz 1.0.0以后使用 easzctl basic-auth -s 开启 (这个我开启的话一直让我输密码，我看了下可能是密码配置模板有点问题)
- 第二步通过dashboard自带的登录流程，使用Kubeconfig Token等方式登录

### token登录
https://NodeIP:NodePort 方式访问 dashboard，支持两种登录方式：Kubeconfig、令牌(Token)
- admin
选择“令牌(Token)”方式登录，复制下面输出的admin token 字段到输入框
```shell
# 创建Service Account 和 ClusterRoleBinding
$ kubectl apply -f /etc/ansible/manifests/dashboard/admin-user-sa-rbac.yaml
# 获取 Bearer Token，找到输出中 ‘token:’ 开头那一行
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
-readonly
# 创建Service Account 和 ClusterRoleBinding
$ kubectl apply -f /etc/ansible/manifests/dashboard/read-user-sa-rbac.yaml
# 获取 Bearer Token，找到输出中 ‘token:’ 开头那一行
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep read-user | awk '{print $1}')