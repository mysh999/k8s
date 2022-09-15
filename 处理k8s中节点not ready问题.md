## 一、现象描述

部署一套K8S后，查看状态报如下：

```bash
# kubectl get nodes
NAME    STATUS     ROLES    AGE   VERSION
k8s01   NotReady   <none>   21h   v1.20.4
k8s02   Ready      <none>   20h   v1.20.4
k8s03   Ready      <none>   20h   v1.20.4



[root@k8s01 ~]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2022-09-15 11:25:31 CST; 18s ago
 Main PID: 1866 (kubelet)
    Tasks: 9
   Memory: 32.2M
   CGroup: /system.slice/kubelet.service
           └─1866 /opt/kubernetes/bin/kubelet --logtostderr=false --v=2 --log-dir=/opt/kubernetes/logs --hostname-override=k8s01 --network-plugin=cni --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig --bootstrap-kubeconfig=...

9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*Framer).ReadFrame(0xc0001e42a0, 0xc000120ca0, 0x0, 0x0, 0x0)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/frame.go:492 +0xa5
9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*clientConnReadLoop).run(0xc000751fa8, 0x0, 0x0)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:1819 +0xd8
9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*ClientConn).readLoop(0xc0008c8480)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:1741 +0x6f
9月 15 11:25:31 k8s01 kubelet[1614]: created by k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*Transport).newClientConn
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:705 +0x6c5
9月 15 11:25:30 k8s01 systemd[1]: kubelet.service failed.
9月 15 11:25:31 k8s01 systemd[1]: Started Kubernetes Kubelet.


```







查看日志：

```bash
[root@k8s01 ~]# kubectl describe node k8s01
Name:               k8s01
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s01
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.0.201/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.244.73.64
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Sep 2022 19:18:41 +0800
Taints:             node.kubernetes.io/unreachable:NoExecute
                    node.kubernetes.io/unreachable:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  k8s01
  AcquireTime:     <unset>
  RenewTime:       Wed, 14 Sep 2022 19:58:21 +0800
Conditions:
  Type                 Status    LastHeartbeatTime                 LastTransitionTime                Reason              Message
  ----                 ------    -----------------                 ------------------                ------              -------
  NetworkUnavailable   False     Wed, 14 Sep 2022 19:48:40 +0800   Wed, 14 Sep 2022 19:48:40 +0800   CalicoIsUp          Calico is running on this node
  MemoryPressure       Unknown   Wed, 14 Sep 2022 19:55:00 +0800   Wed, 14 Sep 2022 19:59:03 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  DiskPressure         Unknown   Wed, 14 Sep 2022 19:55:00 +0800   Wed, 14 Sep 2022 19:59:03 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  PIDPressure          Unknown   Wed, 14 Sep 2022 19:55:00 +0800   Wed, 14 Sep 2022 19:59:03 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  Ready                Unknown   Wed, 14 Sep 2022 19:55:00 +0800   Wed, 14 Sep 2022 19:59:03 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
Addresses:
  InternalIP:  192.168.0.201
  Hostname:    k8s01
Capacity:
  cpu:                4
  ephemeral-storage:  51175Mi
  hugepages-2Mi:      0
  memory:             2821492Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  48294789041
  hugepages-2Mi:      0
  memory:             2719092Ki
  pods:               110
System Info:
  Machine ID:                 2a63c7ea047b419ebb7a140a6c19d7d9
  System UUID:                78BA4D56-1D82-D5CC-DBAA-D31D9D17A96A
  Boot ID:                    3e316c21-1686-474d-9b31-06ef11b03dd2
  Kernel Version:             3.10.0-1160.el7.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.9
  Kubelet Version:            v1.20.4
  Kube-Proxy Version:         v1.20.4
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (2 in total)
  Namespace                   Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                 ------------  ----------  ---------------  -------------  ---
  default                     dns-test             0 (0%)        0 (0%)      0 (0%)           0 (0%)         15h
  kube-system                 calico-node-xtbtn    250m (6%)     0 (0%)      0 (0%)           0 (0%)         15h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                250m (6%)  0 (0%)
  memory             0 (0%)     0 (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
Events:
  Type    Reason    Age        From        Message
  ----    ------    ----       ----        -------
  Normal  Starting  3m47s      kube-proxy  Starting kube-proxy.
  Normal  Starting  <invalid>  kube-proxy  Starting kube-proxy.
  Normal  Starting  <invalid>  kube-proxy  Starting kube-proxy.
[root@k8s01 ~]# systemctl status kubelet   
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2022-09-15 11:25:31 CST; 26s ago
 Main PID: 1866 (kubelet)
    Tasks: 9
   Memory: 32.2M
   CGroup: /system.slice/kubelet.service
           └─1866 /opt/kubernetes/bin/kubelet --logtostderr=false --v=2 --log-dir=/opt/kubernetes/logs --hostname-override=k8s01 --network-plugin=cni --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig --bootstrap-kubeconfig=...

9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*Framer).ReadFrame(0xc0001e42a0, 0xc000120ca0, 0x0, 0x0, 0x0)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/frame.go:492 +0xa5
9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*clientConnReadLoop).run(0xc000751fa8, 0x0, 0x0)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:1819 +0xd8
9月 15 11:25:31 k8s01 kubelet[1614]: k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*ClientConn).readLoop(0xc0008c8480)
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:1741 +0x6f
9月 15 11:25:31 k8s01 kubelet[1614]: created by k8s.io/kubernetes/vendor/golang.org/x/net/http2.(*Transport).newClientConn
9月 15 11:25:31 k8s01 kubelet[1614]: /workspace/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go:705 +0x6c5
9月 15 11:25:30 k8s01 systemd[1]: kubelet.service failed.
9月 15 11:25:31 k8s01 systemd[1]: Started Kubernetes Kubelet.
```



## 二、处理过程

检查配置发现2个问题

1、时间不对

​     时间同步



2、缺少  /opt/kubernetes/cfg/kubelet.kubeconfig  文件

按照部署文档配置kubelet生成kubelet.kubeconfig文件



3、重启kubelet

```bash
[root@k8s01 ~]# systemctl daemon-reload
[root@k8s01 ~]# systemctl enable kubelet
[root@k8s01 ~]# systemctl restart kubelet
```



4、批准csr

```bash
[root@k8s01 ~]# kubectl get csr
NAME                                                   AGE         SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-w62Q-3b49kl0HbGxPRfOKxe82ONHZZcNklMBIgwDLeo   <invalid>   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
[root@k8s01 ~]# kubectl certificate approve node-csr-w62Q-3b49kl0HbGxPRfOKxe82ONHZZcNklMBIgwDLeo
certificatesigningrequest.certificates.k8s.io/node-csr-w62Q-3b49kl0HbGxPRfOKxe82ONHZZcNklMBIgwDLeo approved
[root@k8s01 ~]# kubectl get csr
NAME                                                   AGE         SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-w62Q-3b49kl0HbGxPRfOKxe82ONHZZcNklMBIgwDLeo   <invalid>   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

```





5、查看站点

恢复正常

```bash
[root@k8s01 ~]# kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
k8s01   Ready    <none>   21h   v1.20.4
k8s02   Ready    <none>   20h   v1.20.4
k8s03   Ready    <none>   20h   v1.20.4
```

