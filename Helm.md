#### 安装Helm
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

NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.monitoring.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services grafana)
     export NODE_IP=$(kubectl get nodes --namespace monitoring -o jsonpath="{.items[0].status.addresses[0].address}")
     echo http://$NODE_IP:$NODE_PORT


3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####

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

