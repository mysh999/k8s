## 一、需求描述

将K8S机器的flannel网络替换成calico



## 二、flannel插件卸载

每个节点执行 

```bash
#sudo systemctl stop flanneld
#sudo systemctl disable flanneld
#sudo rm -rf /var/run/flannel/
#sudo rm -rf /etc/flanneld/cert
#sudo ip link del flannel.1
#sudo rm -rf /var/lib/cni

```



## 三、修改kubelet配置

```bash
# vim /etc/kubernetes/kubelet-config.yaml
address: "节点IP"
改成
address: "0.0.0.0"

#cd /etc/systemd/system
#sed -i '/--v=2/a\ \ --network-plugin=cni' kubelet.service
#sed -i 's/--v=2/--v=2 \/' kubelet.service
#systemctl daemon-reload
#systemctl restart kubelet.service
```



## 四、安装calico

calico 的默认模式就是 IPIP，优点是可以跨网段组网，缺点是性能略有损耗

```bash
#kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml


# kubectl get pod --all-namespaces -o wide
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
default         dnsutils-ds-4zhs8                           1/1     Running   475        19d     172.30.80.14   k8s01   <none>           <none>
default         dnsutils-ds-b4xgn                           1/1     Running   475        19d     172.30.96.7    k8s03   <none>           <none>
default         dnsutils-ds-f6rg4                           1/1     Running   475        19d     172.30.200.2   k8s02   <none>           <none>
default         my-nginx-5dd67b97fb-5qx6b                   1/1     Running   7          23h     172.30.80.9    k8s01   <none>           <none>
default         my-nginx-5dd67b97fb-v8g9q                   1/1     Running   8          19d     172.30.80.3    k8s01   <none>           <none>
default         nginx-66d89c74cb-d8d58                      1/1     Running   7          4d10h   172.30.80.6    k8s01   <none>           <none>
default         nginx-66d89c74cb-mcqmt                      1/1     Running   7          4d10h   172.30.96.8    k8s03   <none>           <none>
default         nginx-66d89c74cb-z6rcn                      1/1     Running   7          23h     172.30.80.17   k8s01   <none>           <none>
default         nginx-deployment-6dd86d77d-n5rvl            1/1     Running   7          23h     172.30.80.18   k8s01   <none>           <none>
default         nginx-deployment-6dd86d77d-n94ht            1/1     Running   7          5d13h   172.30.96.9    k8s03   <none>           <none>
default         nginx-ds-4t5kt                              1/1     Running   8          20d     172.30.200.3   k8s02   <none>           <none>
default         nginx-ds-phc4r                              1/1     Running   8          20d     172.30.96.6    k8s03   <none>           <none>
default         nginx-ds-wbgn2                              1/1     Running   8          20d     172.30.80.13   k8s01   <none>           <none>
default         tomcat-7564b9dbb-mm8rl                      1/1     Running   7          23h     172.30.80.10   k8s01   <none>           <none>
default         tomcat-7564b9dbb-ndkdq                      1/1     Running   7          4d9h    172.30.96.10   k8s03   <none>           <none>
default         tomcat-7564b9dbb-tffdj                      1/1     Running   7          4d9h    172.30.80.5    k8s01   <none>           <none>
default         tomcat-app-55686c8fd9-n7sqr                 1/1     Running   7          23h     172.30.80.19   k8s01   <none>           <none>
default         tomcat-app-55686c8fd9-s9c79                 1/1     Running   7          3d19h   172.30.80.15   k8s01   <none>           <none>
ingress-nginx   nginx-ingress-controller-588f7dbb99-5ftqm   1/1     Running   68         3d18h   10.10.1.174    k8s01   <none>           <none>
kube-system     calico-kube-controllers-66d54bb77b-q9n8l    1/1     Running   0          17m     172.30.200.5   k8s02   <none>           <none>
kube-system     calico-node-5h89j                           1/1     Running   0          17m     10.10.1.175    k8s02   <none>           <none>
kube-system     calico-node-79g8f                           1/1     Running   0          17m     10.10.1.176    k8s03   <none>           <none>
kube-system     calico-node-9hmml                           0/1     Running   0          17m     10.10.1.174    k8s01   <none>           <none>
kube-system     coredns-5b969f4c88-p6bh6                    1/1     Running   23         19d     172.30.80.2    k8s01   <none>           <none>
kube-system     elasticsearch-logging-0                     1/1     Running   7          23h     172.30.80.11   k8s01   <none>           <none>
kube-system     elasticsearch-logging-1                     1/1     Running   8          17d     172.30.80.8    k8s01   <none>           <none>
kube-system     fluentd-es-v2.4.0-2fxl6                     1/1     Running   8          17d     172.30.200.4   k8s02   <none>           <none>
kube-system     fluentd-es-v2.4.0-rwvpc                     1/1     Running   9          17d     172.30.80.4    k8s01   <none>           <none>
kube-system     fluentd-es-v2.4.0-vkqs7                     1/1     Running   8          17d     172.30.96.3    k8s03   <none>           <none>
kube-system     heapster-7b8d5c68b8-wm7ll                   1/1     Running   7          23h     172.30.80.16   k8s01   <none>           <none>
kube-system     kibana-logging-f4d99b69f-fctrn              1/1     Running   8          17d     172.30.80.12   k8s01   <none>           <none>
kube-system     kubernetes-dashboard-85bcf5dbf8-ltrq2       1/1     Running   8          19d     172.30.96.2    k8s03   <none>           <none>
kube-system     metrics-server-75d6f5df96-2dpm5             1/1     Running   8          18d     172.30.96.5    k8s03   <none>           <none>
kube-system     monitoring-grafana-658976d65f-v5g72         1/1     Running   8          14d     172.30.96.4    k8s03   <none>           <none>
kube-system     monitoring-influxdb-866db5f944-mfdxv        1/1     Running   8          14d     172.30.80.7    k8s01   <none>           <none>
```



附：

```bash
#查看日志
# kubectl describe pod -n kube-system  calico-node-pmkm8 


#关闭集群
#node节点
#sudo systemctl stop kubelet kube-proxy flanneld docker kube-proxy kube-nginx

#master节点
#sudo systemctl stop kube-apiserver kube-controller-manager kube-scheduler kube-nginx

#清理etcd集群
#sudo systemctl stop etcd
```

