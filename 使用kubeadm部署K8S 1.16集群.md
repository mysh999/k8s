参考：

https://blog.csdn.net/u012921921/article/details/101122371

https://blog.csdn.net/liuxuango/article/details/102517651

https://blog.51cto.com/tryingstuff/2445675





## 一、功能简介

K8S集群中有管理节点与工作节点两种类型。

管理节点：

主要负责K8S集群管理，集群中各节点间的信息交互、任务调度，还负责容器、Pod、NameSpaces、PV等生命周期的管理。

工作节点：

主要为容器和Pod提供计算资源，Pod及容器全部运行在工作节点上，工作节点通过kubelet服务与管理节点通信以管理容器的生命周期，并与集群其他节点进行通信

![微信截图_20191215224035.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g9xsdiyvpxj30f608lmx6.jpg)



## 二、环境条件

CentOS Linux release 7.5.1804 (Core) 
Docker 18.09.7
kubeadm 1.16.1



| 主机名        | IP              | 角色   |
| ------------- | --------------- | ------ |
| k8s-master-01 | 192.168.230.100 | master |
| k8s-master-02 | 192.168.230.101 | master |
| k8s-master-03 | 192.168.230.102 | master |
| k8s-node-01   | 192.168.230.103 | work   |
| k8s-node-02   | 192.168.230.104 | work   |



## 三、系统配置

所有节点都要执行

3.1、配置hosts文件

```bash
# cat /etc/hosts
192.168.230.100 k8s-master-01
192.168.230.101 k8s-master-02
192.168.230.102 k8s-master-03
192.168.230.103 k8s-node-01
192.168.230.104 k8s-node-02
```



3.2、关闭防火墙、selinux和swap

```bash
#systemctl stop firewalld
 
#systemctl disable firewalld
 
#setenforce 0
 
#sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
 
#swapoff -a
 
#sed -i 's/.*swap.*/#&/' /etc/fstab
 
使命名生效
#sysctl --system

# 关闭dnsmasq(否则可能导致docker容器无法解析域名)
# service dnsmasq stop && systemctl disable dnsmasq
```



3.2、配置内核参数

```bash
#cat > /etc/sysctl.d/k8s.conf <<EOF
 
net.bridge.bridge-nf-call-ip6tables = 1
 
net.bridge.bridge-nf-call-iptables = 1
 
net.ipv4.ip_forward = 1
 
vm.swappiness = 0
 
EOF
 
执行命令使修改生效。
#sysctl --system
#sysctl -p /etc/sysctl.d/k8s.conf
#modprobe br_netfilter
```



3.3、配置国内yum源

```bash
#yum install -y wget
 
#mkdir /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
 
#wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo
 
#wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo
 
#yum clean all && yum makecache
```



3.4、配置国内Kubernetes源

```bash
# cat <<EOF > /etc/yum.repos.d/kubernetes.repo

[kubernetes]

name=Kubernetes

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/

enabled=1

gpgcheck=1

repo_gpgcheck=1

gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

EOF
```



3.5、配置 docker 源

```bash
# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```



3.6、kube-proxy开启ipvs的前置条件

由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：ip_vs、ip_vs_rr、ip_vs_wrr、ip_vs_sh、nf_conntrack_ipv4。

注：在所有节点上进行如下操作

保证在节点重启后能自动加载所需模块

```bash
#cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

```



修改权限

```bash
#chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

查看是否已经正确加载所需的内核模块。

```bash
#lsmod | grep -e ip_vs -e nf_conntrack_ipv4

#yum install ipset
#yum install ipvsadm
```

注：如果以上前提条件如果不满足，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式



3. 7、安装相关包

```bash
# yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp

#重置iptables
#iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
```



​	3.8、安装docker

​	这里安装18.09版本

```bash
#mkdir -p /data/kubernetes/docker && cd /data/kubernetes/dockers
#yum install -y yum-utils device-mapper-persistent-data lvm2
#yum install -y docker-ce-18.09.7 docker-ce-cli-18.09.7 containerd.io

# 启动docker服务
#systemctl restart docker
#systemctl enable docker
```



3.9  安装 kubeadm（所有节点）
kubeadm: 部署集群用的命令
kubelet: 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
kubectl: 集群管理工具（可选，只要在控制集群的节点上安装即可）

找到要安装的版本号，这里安装 1.16.4-0

```bash
# yum list kubeadm --showduplicates | sort -r
```

安装指定版本1.16.4

```bash
# yum install -y kubeadm-1.16.4-0 kubelet-1.16.4-0 kubectl-1.16.4-0 --disableexcludes=kubernetes
```



启动kubelet

```bash
#systemctl enable kubelet && systemctl start kubelet
```



## 四、系统高可用配置

使用keepalive实现

部署keepalived - apiserver高可用（master01和master02两个节点）

其中192.168.230.199为keepalived 虚拟IP

4.1、master01和master02两个节点均执行

```bash
# yum -y install keepalived
# mkdir -p /etc/keepalived
```



4.2、master01 keepalived配置文件如下：

```bash
# cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
 router_id keepalive-master
}

vrrp_script check_apiserver {
 script "/etc/keepalived/check-apiserver.sh"
 interval 3
 weight -2
}

vrrp_instance VI-kube-master {
   state MASTER
   interface ens33
   virtual_router_id 68
   priority 100
   dont_track_primary
   advert_int 3
   virtual_ipaddress {
     192.168.230.199
   }
   track_script {
       check_apiserver
   }
}
EOF
```



4.3、keepalived backup（master02）配置文件

```shell
# cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
 router_id keepalive-backup
}

vrrp_script check_apiserver {
 script "/etc/keepalived/check-apiserver.sh"
 interval 3
 weight -2
}

vrrp_instance VI-kube-master {
   state BACKUP
   interface ens33
   virtual_router_id 68
   priority 99
   dont_track_primary
   advert_int 3
   virtual_ipaddress {
     192.168.230.199
   }
   track_script {
       check_apiserver
   }
}
EOF
```



4.4、检测脚本（master和backup均需要）：

```bash
# cat <<EOF > /etc/keepalived/check-apiserver.sh
#!/bin/sh
netstat -ntlp|grep 6443 || exit 1
EOF

# chmod u+x /etc/keepalived/check-apiserver.sh
```



4.5、启动keepalived（master和backup均需要）：

```bash
#systemctl enable keepalived && service keepalived start
```



查看日志

```bash
# journalctl -f -u keepalived
```



查看虚拟ip

```bash
# ip a
```



## 五、部署主节点（master01）

5.1、编写kubeadm 用的 YAML 文件

```bash
# cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.4
controlPlaneEndpoint: "192.168.230.199:6443"
networking:
    # This CIDR is a Calico default. Substitute or remove for your CNI provider.
    podSubnet: "172.33.0.0/16"
imageRepository: registry.aliyuncs.com/google_containers
EOF
```



5.2、kubeadm初始化，大概需要10分钟

```bash
# kubeadm init --config=kubeadm-config.yaml --upload-certs
```



5.3、配置kubectl

```bash
#  mkdir -p $HOME/.kube
#  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



## 六、安装calico网络(master01)

```bash
# mkdir -p /etc/kubernetes/addons
# wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
# sed -i "s#192\.168\.0\.0/16#172\.33\.0\.0/16#" calico.yaml
# kubectl apply -f calico.yaml

# 查看状态
# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-778676476b-lgxx5   1/1     Running   0          13m
calico-node-8vdxt                          1/1     Running   0          13m
coredns-58cc8c89f4-t2zph                   1/1     Running   0          21m
coredns-58cc8c89f4-vxrvl                   1/1     Running   0          21m
etcd-k8s-master-01                         1/1     Running   0          20m
kube-apiserver-k8s-master-01               1/1     Running   0          20m
kube-controller-manager-k8s-master-01      1/1     Running   0          21m
kube-proxy-rghl4                           1/1     Running   0          21m
kube-scheduler-k8s-master-01               1/1     Running   0          20m

# 查看日志
# kubectl describe pods -n kube-system <your_pod_name>
```



## 七、加入其他节点

7.1、master02-03节点加入

在master02-03节点上执行

该命令在kubeadm init输出日志里有显示

```bash
 # kubeadm join 192.168.230.199:6443 --token p34jsl.4nwmu4g69t7lsj3r \
    --discovery-token-ca-cert-hash sha256:3ac67aeda24308c8b5ddb14d6680052e0525d8e557c804dd5cef6edc803e9060 \
    --control-plane --certificate-key 5e0b843984dbc9e7155218f868be29944b949e29483ae41c3b24a51c107927ad
    
    #  mkdir -p $HOME/.kube
    #  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    #  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



7.2、查看节点信息

```bash
# kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
k8s-master-01   Ready    master   36m     v1.16.4
k8s-master-02   Ready    master   8m7s    v1.16.4
k8s-master-03   Ready    master   4m25s   v1.16.4
```



7.3、work节点加入

在node01和node02上执行

```bash
#   kubeadm join 192.168.230.199:6443 --token p34jsl.4nwmu4g69t7lsj3r     --discovery-token-ca-cert-hash sha256:3ac67aeda24308c8b5ddb14d6680052e0525d8e557c804dd5cef6edc803e9060 
```



7.4、查看work节点信息

```bash
# kubectl get nodes      
NAME            STATUS   ROLES    AGE     VERSION
k8s-master-01   Ready    master   46m     v1.16.4
k8s-master-02   Ready    master   17m     v1.16.4
k8s-master-03   Ready    master   13m     v1.16.4
k8s-node-01     Ready    <none>   6m58s   v1.16.4
k8s-node-02     Ready    <none>   105s    v1.16.4
```

7.5、对worker节点添加ROLES标识

```bash
# kubectl label node k8s-node-01 node-role.kubernetes.io/worker=worker
node/k8s-node-01 labeled

# kubectl label node k8s-node-02 node-role.kubernetes.io/worker=worker
node/k8s-node-02 labeled

# kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
k8s-master-01   Ready    master   82m   v1.16.4
k8s-master-02   Ready    master   53m   v1.16.4
k8s-master-03   Ready    master   49m   v1.16.4
k8s-node-01     Ready    worker   42m   v1.16.4
k8s-node-02     Ready    worker   37m   v1.16.4
```







## 八、部署dashboard

8.1、# 下载最新的dashboard

截止到2019.12.16 最新版本是v2.0.0-beta8

版本兼容性列表：

https://github.com/kubernetes/dashboard/releases



```bash
# wget https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
```



将Service改成NodePort类型，注意 YAML 中最下面的 Service 部分新增一个type=NodePort和nodePort: 30005，并暴露30005端口：

```bash
#vi kubernetes-dashboard.yaml
... ...
 spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30005   -->新增
  type: NodePort        -->新增
```





8.2、创建dashboard

```bash
# kubectl apply -f recommended.yaml 
```



8.3、查看状态

```bash
# 查看服务运行情况
#  kubectl get deployment kubernetes-dashboard -n kubernetes-dashboard
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1/1     1            1           2m8s


# kubectl --namespace kubernetes-dashboard  get pods -o wide           
NAME                                         READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-76585494d8-9pm7v   1/1     Running   0          93s   172.33.154.195   k8s-node-01   <none>           <none>
kubernetes-dashboard-5996555fd8-fv6w2        1/1     Running   0          94s   172.33.44.195    k8s-node-02   <none>           <none>

# kubectl get services kubernetes-dashboard -n kubernetes-dashboard
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   ClusterIP   10.105.82.211   <none>        443/TCP   2m36s

#netstat -ntlp|grep 30005
```



8.4、创建登录token

```bash
# 创建service account
# kubectl create sa dashboard-admin -n kube-system

# 创建角色绑定关系
# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin

# 查看dashboard-admin的secret名字
# ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')

# 打印secret的token
# kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}'
eyJhbGciOiJSUzI1NiIsImtpZCI6IkNvVF9FQUZoODJMQ0VXOVB1Y2JYM3h3b01xSVNyV1RzSjBVNVhWZ19OXzQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tNHI1azgiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWQ0YzAxNjUtYWIyNy00MzU1LWE1ZDMtYmZiYmQzN2VkZGZlIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.F6T0nQ8wdba_jeLNXp5vh0HRSkvgvUExGw1A6OowLITN9N37PMwdsLWoVqPpzBoy0ZGYOcLFjoAuTnmMSnBE5LFUlQTfIkqOyM6wFYCJbGl7mPgC4SbMYYTi4X3dH2uON7FPmHpaQVoU9ntfBI5iXlFWb1_RgzNi8SmPd2fD5TsCAYODCqPjsFLnI1kB-G0ozCpqBZnq80e6KCFf2GOzOiNwYcj8x0nUHcS2JmIicZuOiLuQsqCYQ0Lf8XPlm0oBtsbZk4h9a5rZG9pIGpC_O_MLOTytrOPRV8X36OHheCSc5YwCh77k8-KOVNmwuhmd5DztVxoc6VABM4AN2iRn_w
```



8.5、登录dashboard

在firefox浏览器中输入https://192.168.230.199:30005

再输入以上获取的token，登录

![20191216_1654_001.jpg](http://ww1.sinaimg.cn/large/007Xg1efgy1g9ynztg61wj311b0jgjz4.jpg)



## 九、集群可用性测试

9.1、创建nginx镜像

   ```bash
# cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
   ```



9.2、执行创建

```bash
# kubectl apply -f nginx-ds.yml 
```



9.3、检查状态

```bash
# 检查各 Node 上的 Pod IP 连通性
# kubectl get pods  -o wide

# 检查service可达性
# kubectl get svc

# 在每个节点上访问服务
# curl <service-ip>:<port>

# 在每个节点检查node-port可用性
# curl <node-ip>:<port>
```



9.4、检查DNS的可用性

```bash
检查DNS的可用性
创建一个nginx pod

# cat > pod-nginx.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
EOF

# 创建pod
# kubectl create -f pod-nginx.yaml

# 进入pod，查看dns
# kubectl exec  nginx -i -t -- /bin/bash

# 查看dns配置
root@nginx:/# cat /etc/resolv.conf 
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

# 查看名字是否可以正确解析
root@nginx:/# ping nginx-ds
```

