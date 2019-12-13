### 安装Helm
``` shell
ansible-playbook /etc/ansible/roles/helm/helm.yml
```
### 安装Prometheus
```shell
source ~/.bashrc
cd /etc/ansible/manifests/prometheus
# 安装 prometheus chart，如果你的helm安装没有启用tls证书，请忽略--tls参数
helm install --tls \
        --name monitor \
        --namespace monitoring \
        -f prom-settings.yaml \
        -f prom-alertsmanager.yaml \
        -f prom-alertrules.yaml \
        prometheus
# 安装 grafana chart
helm install --tls \
	--name grafana \
	--namespace monitoring \
	-f grafana-settings.yaml \
	-f grafana-dashboards.yaml \
	grafana
```

### 验证安装
```shell
# 查看相关pod和svc
kubectl get pod,svc -n monitoring
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/grafana-78747c76-wm8qd                                   0/1     Running   0          103s
pod/monitor-prometheus-alertmanager-59cc5c4b6-wlhtg          2/2     Running   0          4m30s
pod/monitor-prometheus-kube-state-metrics-7556696ff7-df8x6   1/1     Running   0          4m30s
pod/monitor-prometheus-node-exporter-6zz9z                   1/1     Running   0          4m30s
pod/monitor-prometheus-node-exporter-8qs5r                   1/1     Running   0          4m30s
pod/monitor-prometheus-node-exporter-gg7gx                   1/1     Running   0          4m29s
pod/monitor-prometheus-node-exporter-xvflg                   1/1     Running   0          4m30s
pod/monitor-prometheus-server-579954d46d-ncbjp               2/2     Running   0          4m29s

NAME                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/grafana                                 NodePort    10.68.250.185   <none>        80:39002/TCP   104s
service/monitor-prometheus-alertmanager         NodePort    10.68.11.250    <none>        80:39001/TCP   4m32s
service/monitor-prometheus-kube-state-metrics   ClusterIP   None            <none>        80/TCP         4m31s
service/monitor-prometheus-node-exporter        ClusterIP   None            <none>        9100/TCP       4m31s
service/monitor-prometheus-server               NodePort    10.68.193.154   <none>        80:39000/TCP   4m31s
```
- 访问prometheus的web界面：http://192.168.199.181:39000
- 访问alertmanage的web界面：http://192.168.199.181:390001
- 访问grafana的web界面：http://192.168.199.181:39002 (默认用户密码 admin:admin)
- 查看grafana的admin密码(怕自己改过之后忘记)
```shell
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 管理操作
- 升级（修改配置）：修改配置请在prom-settings.yaml prom-alertsmanager.yaml 等文件中进行，保存后执行：
```shell
# 修改prometheus
$ helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
# 修改grafana
$ helm upgrade --tls grafana -f grafana-settings.yaml -f grafana-dashboards.yaml grafana
```
- 回退：具体可以参考helm help rollback文档
```shell
$ helm rollback --tls monitor [REVISION]
```
- 删除
```shell
$ helm del --tls monitor --purge
$ helm del --tls grafana --purge
```
### 验证告警
- 修改prom-alertsmanager.yaml文件中邮件告警为有效的配置内容，并使用 helm upgrade更新安装
- 手动临时关闭 master 节点的 kubelet 服务，等待几分钟看是否有告警邮件发送(妹给我发啊，没配置好)
```shell
# 在 master 节点运行
$ systemctl stop kubelet
```

