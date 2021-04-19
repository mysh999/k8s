## 一、特性说明

kube-proxy 是 Kubernetes 中的关键组件。他的角色就是在服务（ClusterIP 和 NodePort）和其后端 Pod 之间进行负载均衡。kube-proxy 有三种运行模式，每种都有不同的实现技术：**userspace**、**iptables** 或者 **IPVS**。

userspace 模式非常陈旧、缓慢，已经不推荐使用

ipvs更适合大规模使用场景

k8s默认使用iptables



## 二、ipvs替换iptables

### 2.1、查看内核模块是否加载

```bash
# lsmod|grep -i ip_vs
ip_vs_sh               12688  0 
ip_vs_wrr              12697  0 
ip_vs_rr               12600  0 
ip_vs                 145497  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139224  9 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
[root@k8s-node1 ~]# lsmod|grep -i nf_conntrack_ipv4
nf_conntrack_ipv4      15053  31 
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
nf_conntrack          139224  9 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
```

如果没有加载，使用如下命令加载ipvs相关模块

```bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
```



### 2.2、更改kube-proxy配置

```bash
# kubectl edit configmap kube-proxy -n kube-system
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: false
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: "ipvs"              #修改为ipvs
    nodePortAddresses: null
    oomScoreAdj: null
```

其中mode原来是空，默认为iptables模式，改为ipvs

scheduler默认是空，默认负载均衡算法为轮询





### 2.3、删除所有kube-proxy的pod 

```bash
# kubectl -n kube-system get pods|grep -i kube-proxy
kube-proxy-62br4                          1/1     Running   14         22d
kube-proxy-89vtp                          1/1     Running   14         22d
kube-proxy-pvmx9                          1/1     Running   15         22d



# kubectl -n kube-system delete pod kube-proxy-62br4      
# kubectl -n kube-system delete pod kube-proxy-89vtp 
# kubectl -n kube-system delete pod kube-proxy-pvmx9 


# 系统重建kube-proxy  pod
# kubectl -n kube-system get pods|grep -i kube-proxy 
kube-proxy-k2czv                          1/1     Running   0          27s
kube-proxy-trqzm                          1/1     Running   0          5s
kube-proxy-x92fb                          1/1     Running   0          43s

# 查看日志，发现有ipvs proxier 日志内容即可
# kubectl -n kube-system logs -f kube-proxy-k2czv
I0419 23:30:01.719812       1 node.go:172] Successfully retrieved node IP: 192.168.188.62
I0419 23:30:01.719980       1 server_others.go:142] kube-proxy node IP is an IPv4 address (192.168.188.62), assume IPv4 operation
I0419 23:30:01.745285       1 server_others.go:258] Using ipvs Proxier.    #显示使用新模式
E0419 23:30:01.745595       1 proxier.go:389] can't set sysctl net/ipv4/vs/conn_reuse_mode, kernel version must be at least 4.1
W0419 23:30:01.745720       1 proxier.go:445] IPVS scheduler not specified, use rr by default
I0419 23:30:01.745915       1 server.go:650] Version: v1.20.0
I0419 23:30:01.746551       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I0419 23:30:01.747407       1 config.go:315] Starting service config controller
I0419 23:30:01.747465       1 shared_informer.go:240] Waiting for caches to sync for service config
I0419 23:30:01.747487       1 config.go:224] Starting endpoint slice config controller
I0419 23:30:01.747494       1 shared_informer.go:240] Waiting for caches to sync for endpoint slice config
I0419 23:30:01.847690       1 shared_informer.go:247] Caches are synced for endpoint slice config 
I0419 23:30:01.847726       1 shared_informer.go:247] Caches are synced for service config 
```





### 2.4 、查看ipvs规则

```bash
# yum -y install ipvsadm   #安装包

# ipvsadm -Ln     # 查看规则
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  127.0.0.1:31356 rr
  -> 10.244.36.127:2181           Masq    1      0          0         
  -> 10.244.169.180:2181          Masq    1      0          0         
TCP  172.17.0.1:30001 rr
  -> 10.244.169.185:8443          Masq    1      0          0         
TCP  172.17.0.1:30577 rr
  -> 10.244.169.136:27017         Masq    1      0          0         
TCP  172.17.0.1:31356 rr
  -> 10.244.36.127:2181           Masq    1      0          0         
  -> 10.244.169.180:2181          Masq    1      0          0         
TCP  192.168.188.61:30001 rr
  -> 10.244.169.185:8443          Masq    1      0          0         
TCP  192.168.188.61:30355 rr
  -> 10.244.169.139:80            Masq    1      0          0         
  -> 10.244.169.149:80            Masq    1      0          0         
TCP  192.168.188.61:30577 rr
  -> 10.244.169.136:27017         Masq    1      0          0         
TCP  192.168.188.61:31356 rr
  -> 10.244.36.127:2181           Masq    1      0          0         
  -> 10.244.169.180:2181          Masq    1      0          0         
TCP  10.96.0.1:443 rr
  -> 192.168.188.61:6443          Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.36.125:53             Masq    1      0          0         
  -> 10.244.36.126:53             Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.36.125:9153           Masq    1      0          0         
  -> 10.244.36.126:9153           Masq    1      0          0         
TCP  10.100.199.66:1883 rr
  -> 10.244.169.187:1883          Masq    1      0          0         
TCP  10.100.199.66:8081 rr
  -> 10.244.169.187:8081          Masq    1      0          0         
TCP  10.100.199.66:8083 rr
  -> 10.244.169.187:8083          Masq    1      0          0         
TCP  10.100.199.66:8084 rr
  -> 10.244.169.187:8084          Masq    1      0          0         
TCP  10.100.199.66:8883 rr
  -> 10.244.169.187:8883          Masq    1      0          0         
TCP  10.100.199.66:18083 rr
  -> 10.244.169.187:18083         Masq    1      0          0         
TCP  10.101.206.247:6379 rr
  -> 10.244.169.191:6379          Masq    1      0          0         
TCP  10.103.162.125:80 rr
  -> 10.244.36.66:80              Masq    1      0          0         
TCP  10.103.252.234:3306 rr
  -> 10.244.169.140:3306          Masq    1      0          0         
TCP  10.104.67.53:80 rr
  -> 10.244.169.139:80            Masq    1      0          0         
  -> 10.244.169.149:80            Masq    1      0          0         
TCP  10.107.16.143:27017 rr
  -> 10.244.169.136:27017         Masq    1      0          0         
TCP  10.107.237.203:8000 rr
  -> 10.244.169.190:8000          Masq    1      0          0         
TCP  10.110.72.180:443 rr
  -> 10.244.169.185:8443          Masq    1      0          0         
TCP  10.110.105.192:2181 rr
  -> 10.244.36.127:2181           Masq    1      0          0         
  -> 10.244.169.180:2181          Masq    1      0          0         
TCP  10.244.235.192:30001 rr
  -> 10.244.169.185:8443          Masq    1      0          0         
TCP  10.244.235.192:30355 rr
  -> 10.244.169.139:80            Masq    1      0          0         
  -> 10.244.169.149:80            Masq    1      0          0         
TCP  10.244.235.192:30577 rr
  -> 10.244.169.136:27017         Masq    1      0          0         
TCP  10.244.235.192:31356 rr
  -> 10.244.36.127:2181           Masq    1      0          0         
  -> 10.244.169.180:2181          Masq    1      0          0         
TCP  127.0.0.1:30001 rr
  -> 10.244.169.185:8443          Masq    1      0          0         
TCP  127.0.0.1:30355 rr
  -> 10.244.169.139:80            Masq    1      0          0         
  -> 10.244.169.149:80            Masq    1      0          0         
TCP  127.0.0.1:30577 rr
  -> 10.244.169.136:27017         Masq    1      0          0         
TCP  172.17.0.1:30355 rr
  -> 10.244.169.139:80            Masq    1      0          0         
  -> 10.244.169.149:80            Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.36.125:53             Masq    1      0          0         
  -> 10.244.36.126:53             Masq    1      0          0  
```

