## 一、前提和目标

- 使用prometheus对K8S集群做监控

- promethes使用coreos的**kube-prometheus**模板来创建

- 经验证：kube-prometheus和K8S集群之间存在版本兼容性问题，目前测试的结果是kube-prometheus（release-0.2）版本可以和使用kubeadm部署的V1.15.3版本相匹配（kube-prometheus文件要做少量修改）

  修改后的kube-prometheus文件已经上传到

  https://github.com/mysh1984/k8s/tree/master/kube-prometheus_release_0.2_manifests_modify

- K8S集群部署文档和版本参考

  https://github.com/mysh1984/k8s/blob/master/使用kubeadm部署k8s测试集群.md

  



## 二、部署prometheus

2.1、获取修改后的kube-prometheus文件

2.2、该目录文件已做以下修改

![image]( https://raw.githubusercontent.com/mysh1984/k8s/master/images/yml1.png)



![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/yml2.png)

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/yml3.png)



2.3 加载配置文件

```bash
# kubectl apply -f .
namespace/monitoring created

# kubectl apply -f operator/
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
service/prometheus-operator created
serviceaccount/prometheus-operator created

# kubectl -n monitoring get pod
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-7cb68545c6-z2kjn   1/1     Running   0          41s

# kubectl apply -f adapter/
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrole.rbac.authorization.k8s.io/prometheus-adapter created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-adapter created
clusterrolebinding.rbac.authorization.k8s.io/resource-metrics:system:auth-delegator created
clusterrole.rbac.authorization.k8s.io/resource-metrics-server-resources created
configmap/adapter-config created
deployment.apps/prometheus-adapter created
rolebinding.rbac.authorization.k8s.io/resource-metrics-auth-reader created
service/prometheus-adapter created
serviceaccount/prometheus-adapter created

# kubectl apply -f alertmanager/
alertmanager.monitoring.coreos.com/main created
secret/alertmanager-main created
service/alertmanager-main created
serviceaccount/alertmanager-main created

# kubectl apply -f node-exporter/
clusterrole.rbac.authorization.k8s.io/node-exporter created
clusterrolebinding.rbac.authorization.k8s.io/node-exporter created
daemonset.apps/node-exporter created
service/node-exporter created
serviceaccount/node-exporter created

# kubectl apply -f kube-state-metrics/
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
role.rbac.authorization.k8s.io/kube-state-metrics created
rolebinding.rbac.authorization.k8s.io/kube-state-metrics created
service/kube-state-metrics created
serviceaccount/kube-state-metrics created

# kubectl apply -f grafana/
secret/grafana-datasources created
configmap/grafana-dashboard-k8s-cluster-rsrc-use created
configmap/grafana-dashboard-k8s-node-rsrc-use created
configmap/grafana-dashboard-k8s-resources-cluster created
configmap/grafana-dashboard-k8s-resources-namespace created
configmap/grafana-dashboard-k8s-resources-pod created
configmap/grafana-dashboard-k8s-resources-workload created
configmap/grafana-dashboard-k8s-resources-workloads-namespace created
configmap/grafana-dashboard-nodes created
configmap/grafana-dashboard-persistentvolumesusage created
configmap/grafana-dashboard-pods created
configmap/grafana-dashboard-statefulset created
configmap/grafana-dashboards created
deployment.apps/grafana created
service/grafana created
serviceaccount/grafana created

# kubectl apply -f prometheus/
clusterrole.rbac.authorization.k8s.io/prometheus-k8s created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-k8s created
prometheus.monitoring.coreos.com/k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s-config created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s-config created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
prometheusrule.monitoring.coreos.com/prometheus-k8s-rules created
service/prometheus-k8s created
serviceaccount/prometheus-k8s created

# kubectl apply -f serviceMonitor/
servicemonitor.monitoring.coreos.com/prometheus-operator created
servicemonitor.monitoring.coreos.com/alertmanager created
servicemonitor.monitoring.coreos.com/grafana created
servicemonitor.monitoring.coreos.com/kube-state-metrics created
servicemonitor.monitoring.coreos.com/node-exporter created
servicemonitor.monitoring.coreos.com/prometheus created
servicemonitor.monitoring.coreos.com/kube-apiserver created
servicemonitor.monitoring.coreos.com/coredns created
servicemonitor.monitoring.coreos.com/kube-controller-manager created
servicemonitor.monitoring.coreos.com/kube-scheduler created
servicemonitor.monitoring.coreos.com/kubelet created
```



2.4  查看prometheus对象

http://节点IP:30090

正常情况下target目标均是蓝色

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/target.png)



坑一： 

kubelet/0和kubelet/1报红色，提示

 unable to fetch metrics from Kubelet linuxea.master-1.com (10.10.240.161): Get https://10.10.240.161:10255/stats/summary/: dial tcp 10.10.240.161:10255: connect: connection refused,



解决办法:

所有节点修改/var/lib/kubelet/kubeadm-flags.env 文件，追加--read-only-port=10255内容，再重启kubelet服务

```bash
# vi /var/lib/kubelet/kubeadm-flags.env 
KUBELET_KUBEADM_ARGS="--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --read-only-port=10255"         #追加--read-only-port=10255

#重启kubelet服务
#systemctl restart kubelet

# 出现10255端口
# netstat -ntlp|grep -i 10255 
tcp6       0      0 :::10255                :::*                    LISTEN      31963/kubelet 

#再刷新prometheus target界面，变蓝色正常
```



## 三、grafana配置

http://ip:32000进入grafana界面，默认用户名密码admin

3.1、prometheus已经在数据源中

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana1.png)



3.2  查看自带的模板

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana2.png)



![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana3.png)



3.3  导入第三方模板

https://grafana.com/grafana/dashboards下载第三方模板

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana4.png)



导入

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana4.png)



查看效果

![](https://raw.githubusercontent.com/mysh1984/k8s/master/images/grafana6.png)