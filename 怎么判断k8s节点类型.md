执行以下命令查看： 

```bash
# kubectl get nodes -o wide
NAME       STATUS   ROLES    AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
kuber-22   Ready    <none>   113d   v1.17.6   10.10.2.22    <none>        CentOS Linux 7 (Core)   3.10.0-1127.10.1.el7.x86_64   docker://19.3.8
noder-30   Ready    <none>   113d   v1.17.6   10.10.2.30    <none>        CentOS Linux 7 (Core)   3.10.0-1127.10.1.el7.x86_64   docker://19.3.8
```

有时候此命令查看的roles信息是none，无法判断节点属于master还是node，可以在节点上查看apiserver或者controller-manager进程，该进程在哪，哪个就是master



```bash
# ps -ef|grep -i controller-manager|grep -iv grep
# ps -ef|grep -i kube-apiserver|grep -iv grep   
```





查看节点详细信息

```bash
# kubectl describe node kuber-22
Name:               kuber-22
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    ingress-controller-schedule=true
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kuber-22
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 10.10.2.22/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 192.168.242.192
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 23 Jun 2020 15:28:02 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  kuber-22
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Oct 2020 11:10:55 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 14 Jul 2020 21:12:14 +0800   Tue, 14 Jul 2020 21:12:14 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 15 Oct 2020 11:10:58 +0800   Tue, 23 Jun 2020 15:28:02 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 15 Oct 2020 11:10:58 +0800   Tue, 23 Jun 2020 15:28:02 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 15 Oct 2020 11:10:58 +0800   Tue, 23 Jun 2020 15:28:02 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 15 Oct 2020 11:10:58 +0800   Tue, 23 Jun 2020 15:33:33 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.10.2.22
  Hostname:    kuber-22
Capacity:
  cpu:                4
  ephemeral-storage:  522982916Ki
  hugepages-2Mi:      0
  memory:             8008664Ki
  pods:               100
Allocatable:
  cpu:                3
  ephemeral-storage:  481981054588
  hugepages-2Mi:      0
  memory:             6906264Ki
  pods:               100
System Info:
  Machine ID:                 95c35ac4068d471aa3e5b6b3836633d3
  System UUID:                95c35ac4068d471aa3e5b6b3836633d3
  Boot ID:                    3aed2f68-da34-47c8-b910-f0ded8ecf704
  Kernel Version:             3.10.0-1127.10.1.el7.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.8
  Kubelet Version:            v1.17.6
  Kube-Proxy Version:         v1.17.6
PodCIDR:                      172.31.2.0/24
PodCIDRs:                     172.31.2.0/24
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  devopt                      nginx-only-limit-57fbc8997c-vxzq6             1 (33%)       1 (33%)     0 (0%)           0 (0%)         55d
  ingress-nginx               ingress-nginx-controller-xbsrn                100m (3%)     0 (0%)      90Mi (1%)        0 (0%)         61d
  kube-system                 calico-node-5n6np                             250m (8%)     0 (0%)      0 (0%)           0 (0%)         92d
  kube-system                 coredns-75b95966b7-d85zl                      100m (3%)     0 (0%)      70Mi (1%)        170Mi (2%)     92d
  kube-system                 coredns-75b95966b7-vhg7k                      100m (3%)     0 (0%)      70Mi (1%)        170Mi (2%)     92d
  kube-system                 nginx-proxy-kuber-22                          25m (0%)      0 (0%)      32M (0%)         0 (0%)         113d
  weave                       weave-scope-agent-m29lt                       100m (3%)     0 (0%)      100Mi (1%)       0 (0%)         113d
  weave                       weave-scope-app-7887fbd9ff-lb87c              0 (0%)        0 (0%)      0 (0%)           0 (0%)         113d
  weave                       weave-scope-cluster-agent-6d68855547-zrm75    25m (0%)      0 (0%)      80Mi (1%)        0 (0%)         113d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                1700m (56%)    1 (33%)
  memory             451090Ki (6%)  340Mi (5%)
  ephemeral-storage  0 (0%)         0 (0%)
Events:              <none>
```



