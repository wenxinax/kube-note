### 部署Kubeflow
#### 安装ksonnet
Kubeflow 利用 ksonnet 打包和部署其组件。
```shell
# curl -o ks_0.9.2_linux_amd64.tar.gz http://kubeflow.oss-cn-beijing.aliyuncs.com/ks_0.9.2_linux_amd64.tar.gz
# tar -xvf ks_0.9.2_linux_amd64.tar.gz
# cp ks_0.9.2_linux_amd64/ks /usr/local/bin/
# ks version
```
#### 准备GithubToken
- 登录https://github.com/settings/tokens创建token。无须提供任何权限给这个token
- 请保存好该token，因为你无法再查看到这个token。如果没有记录好token，你就只能重现创建一个新的token
- 可以将该token保存到shell启动脚本中,比如
```shell
echo "export GITHUB_TOKEN=${GITHUB_TOKEN}" >> ~/.bashrc
export GITHUB_TOKEN=${GITHUB_TOKEN}
```
#### 安装Kubeflow
```shell
# 创建Kubeflow运行的namespace
NAMESPACE=kubeflow
kubectl create namespace ${NAMESPACE}

# 指定特有版本
VERSION=jupyterhub-alibaba-cloud

# 初始化Kubeflow应用，并且将其namespace设置为default环境
APP_NAME=my-kubeflow
ks init ${APP_NAME} --api-spec=version:v1.9.3
cd ${APP_NAME}
ks env set default --namespace ${NAMESPACE}

# 安装 Kubeflow 模块
ks registry add kubeflow github.com/cheyang/kubeflow/tree/${VERSION}/kubeflow
ks registry list

ks pkg install kubeflow/core@${VERSION}
ks pkg install kubeflow/tf-serving@${VERSION}
ks pkg install kubeflow/tf-job@${VERSION}

# 创建核心模块的模板
ks generate kubeflow-core kubeflow-core

# 支持运行在阿里云Kubernetes容器服务
ks param set kubeflow-core cloud ack

ks param set kubeflow-core jupyterHubImage registry.aliyuncs.com/kubeflow-images-public/jupyterhub-k8s:1.0.1
ks param set kubeflow-core tfJobImage registry.cn-hangzhou.aliyuncs.com/kubeflow-images-public/tf_operator:v20180326-6214e560
ks param set kubeflow-core tfAmbassadorImage registry.aliyuncs.com/datawire/ambassador:0.34.0
ks param set kubeflow-core tfStatsdImage registry.aliyuncs.com/datawire/statsd:0.34.0

ks param set kubeflow-core jupyterNotebookRegistry registry.aliyuncs.com
ks param set kubeflow-core JupyterNotebookRepoName kubeflow-images-public

# 这里为了使用简单，将服务以LoadBalancer的方式暴露，这样可以直接通过阿里云SLB的ip访问。
ks param set kubeflow-core jupyterHubServiceType LoadBalancer
ks param set kubeflow-core tfAmbassadorServiceType LoadBalancer
ks param set kubeflow-core tfJobUiServiceType LoadBalancer


# 部署KubeFlow
ks apply default -c kubeflow-core
```

### 验证部署
查看相关的Pod是否处于Running状态。这时间要等挺久的，等了大概十分钟。
```shell
root@R740-2-1:~/myspace/opensource/my-kubeflow# kubectl get po -n kubeflow
NAME                                READY   STATUS    RESTARTS   AGE
ambassador-7ffd7f744f-2zr9z         2/2     Running   0          13m
ambassador-7ffd7f744f-lwjmz         2/2     Running   0          13m
ambassador-7ffd7f744f-zqkzq         2/2     Running   0          13m
centraldashboard-57464949f9-9skzd   1/1     Running   0          13m
tf-hub-0                            1/1     Running   0          13m
tf-job-dashboard-5657b58f85-w9682   1/1     Running   0          13m
tf-job-operator-54d8f994b5-9h2vl    1/1     Running   0          13m
```

### 体验Jupyter Hub
可以直接通过外网ip访问JupyterHub
```shell
root@R740-2-1:~/myspace/opensource/my-kubeflow# kubectl get svc -n kubeflow tf-hub-lb
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)        AGE
tf-hub-lb   LoadBalancer   10.68.16.150   192.168.199.242   80:22120/TCP   22m
```

