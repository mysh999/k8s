## 一、需求

k8s 集群新增一个node节点

备注：

1、目前有3个节点，既是master又是node

2、以下默认情况下都是在node节点上执行，如果是在master主机执行，会加master前缀

3、思路

​      新节点上部署kubelet、nginx、kube-proxy组件

4、参考来源：

```html
https://k8s-install.opsnull.com/07-3.kube-proxy.html
```





## 二、系统初始化

2.1 hosts文件新增记录

```bash
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.1.174 ubt-k8s-master-01 k8s01
10.10.1.175 ubt-k8s-master-02 k8s02
10.10.1.176 ubt-k8s-master-03 k8s03
10.10.1.177 ubt-k8s-node-01 k8snode01  #新增
```



2.2 新增docker用户

```bash
# useradd -m docker -g docker
```



2.3  设置新节点无密码登陆 

```bash
# ssh-copy-id root@ubt-k8s-node-01
```



2.4 更新path变量

```bash
#echo 'PATH=/opt/k8s/bin:$PATH' >>/root/.bashrc
#source /root/.bashrc
```



2.5 安装依赖包

```bash
#yum install -y epel-release
#yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget
```



2.6 关闭防火墙

```bash
# systemctl stop firewalld
# systemctl disable firewalld
```



2.7 关闭swap分区 

```bash
#swapoff -a
#sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 
```



2.8 关闭selinux

```bash
#setenforce 0
#sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```



2.9 关闭 dnsmasq（可选）
linux 系统开启了 dnsmasq 后(如 GUI 环境)，将系统 DNS Server 设置为 127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它：

```bash
#systemctl stop dnsmasq
#systemctl disable dnsmasq
```



2.10 加载内核模块

```bash
#modprobe ip_vs_rr
#modprobe br_netfilter
```



2.11 优化内核参数

```bash
#cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
#cp kubernetes.conf  /etc/sysctl.d/kubernetes.conf
#sysctl -p /etc/sysctl.d/kubernetes.conf
```



2.12 设置时区

```bash
# 调整系统 TimeZone
#timedatectl set-timezone Asia/Shanghai

# 将当前的 UTC 时间写入硬件时钟
#timedatectl set-local-rtc 0

# 重启依赖于系统时间的服务
#systemctl restart rsyslog 
#systemctl restart crond
```



2.13 关闭无关服务

```bash
#systemctl stop postfix && systemctl disable postfix
```



2.14 设置 rsyslogd 和 systemd journald

```bash
mkdir /var/log/journal # 持久化保存日志的目录
mkdir /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 压缩历史日志
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald
```



2.15 创建相关目录

```bash
#mkdir -p  /opt/k8s/{bin,work} /etc/{kubernetes,etcd}/cert
```



2.16 升级内核

```bash
#rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
#yum --enablerepo=elrepo-kernel install -y kernel-lt
# 设置开机从新内核启动
#grub2-set-default 0
```



2.17 关闭 NUMA

```bash
#cp /etc/default/grub{,.bak}
#vim /etc/default/grub # 在 GRUB_CMDLINE_LINUX 一行添加 `numa=off` 参数，如下所示：

重新生成 grub2 配置文件：
#cp /boot/grub2/grub.cfg{,.bak}
#grub2-mkconfig -o /boot/grub2/grub.cfg
```





## 三、部署kubelet组件

3.1 部署kubernetes-server

```bash
# cd /opt/k8s/work/
# tar -zxvf kubernetes-server-linux-amd64.tar.gz 
#拷贝二进制文件
#cp -r kubernetes/server/bin/{apiextensions-apiserver,cloud-controller-manager,kube-apiserver,kube-controller-manager,kube-proxy,kube-scheduler,kubeadm,kubectl,kubelet,mounter} /opt/k8s/bin/
```



3.2  master主机创建kubelet bootstrap kubeconfig文件



```bash
【master】#source /opt/k8s/bin/environment.sh

#export node_name=k8snode01    
# export node_ip=10.10.1.177

# 创建 token

【master】#export BOOTSTRAP_TOKEN=$(kubeadm token create \
  --description kubelet-bootstrap-token \
  --groups system:bootstrappers:${node_name} \
  --kubeconfig ~/.kube/config)

# 设置集群参数
【master】#kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# 设置客户端认证参数
【master】#kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# 设置上下文参数
【master】#kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# 设置默认上下文
【master】#kubectl config use-context default --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
```


```bash
#查看创建的token
【master】# kubeadm token list --kubeconfig ~/.kube/config
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION               EXTRA GROUPS
ba9ucp.rk0vynzoo74d9723   23h       2019-11-14T16:28:02+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:k8snode01

【master】#查看token关联的secret
# kubectl get secrets -n kube-system|grep bootstrap-token
bootstrap-token-ba9ucp                           bootstrap.kubernetes.io/token         7      4m40s

【master】#分发bootstrap kubeconfig文件到node节点
# scp kubelet-bootstrap-k8snode01.kubeconfig root@ubt-k8s-node-01:/etc/kubernetes/kubelet-bootstrap.kubeconfig
```



3.3 创建和分发kubelet配置文件

```bash
【master】#创建和分发kubelet配置文件
# sed -e "s/##NODE_IP##/${node_ip}/" kubelet-config.yaml.template > kubelet-config-${node_ip}.yaml.template
# scp kubelet-config-${node_ip}.yaml.template root@${node_ip}:/etc/kubernetes/kubelet-config.yaml
```



3.4 创建和分发kubelet systemd unit文件

```bash
【master】# sed -e "s/##NODE_NAME##/${node_name}/" kubelet.service.template > kubelet-${node_name}.service

# scp kubelet-${node_name}.service root@${node_name}:/etc/systemd/system/kubelet.service
```



3.5 启动kubelet服务

```bash
# mkdir -p /data/k8s/k8s/kubelet/kubelet-plugins/volume/exec/
#systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet
```



## 四、分发kube-proxy二进制文件



4.1 安装依赖包

```bash
#yum install -y conntrack ipvsadm ntp ntpdate ipset jq iptables curl sysstat libseccomp && modprobe ip_vs
```



4.2 安装nginx

```bash
#cd /opt/k8s/work
#tar -xzvf nginx-1.15.3.tar.gz
#cd /opt/k8s/work/nginx-1.15.3
#mkdir nginx-prefix
#./configure --with-stream --without-http --prefix=$(pwd)/nginx-prefix --without-http_uwsgi_module --without-http_scgi_module --without-http_fastcgi_module
#make && make install

#验证
#./nginx-prefix/sbin/nginx -v
#输出
nginx version: nginx/1.15.3

#查看动态链接库
# ldd ./nginx-prefix/sbin/nginx
        linux-vdso.so.1 =>  (0x00007fff413f2000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f1fd4312000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f1fd40f6000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f1fd3d29000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f1fd4516000)
```



4.3 部署nginx

```bash
# mkdir -p /opt/k8s/kube-nginx/{conf,logs,sbin}
# cp -r /opt/k8s/work/nginx-1.15.3/nginx-prefix/sbin/nginx /opt/k8s/kube-nginx/sbin/kube-nginx
# chmod a+x /opt/k8s/kube-nginx/sbin/*
```



4.4 配置 nginx，开启 4 层透明转发功能

```bash
# cat /opt/k8s/kube-nginx/conf/kube-nginx.conf
worker_processes 1;

events {
    worker_connections  1024;
}

stream {
    upstream backend {
        hash $remote_addr consistent;
        server 10.10.1.174:6443        max_fails=3 fail_timeout=30s;
        server 10.10.1.175:6443        max_fails=3 fail_timeout=30s;
        server 10.10.1.176:6443        max_fails=3 fail_timeout=30s;
    }

    server {
        listen 127.0.0.1:8443;
        proxy_connect_timeout 1s;
        proxy_pass backend;
    }
}
```



4.5 配置服务

```bash
# cat /etc/systemd/system/kube-nginx.service
[Unit]
Description=kube-apiserver nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStartPre=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx -t
ExecStart=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx
ExecReload=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx -s reload
PrivateTmp=true
Restart=always
RestartSec=5
StartLimitInterval=0
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```



启动服务

```bash
# systemctl daemon-reload
# systemctl enable kube-nginx
# systemctl restart kube-nginx
```



4.6 配置kube-proxy

【master】分发kubeoconfig文件

```bash
# scp kube-proxy.kubeconfig ubt-k8s-node-01:/etc/kubernetes/
```



配置kube-proxy文件

```bash
# vim /etc/kubernetes/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  burst: 200
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
  qps: 100
bindAddress: 10.10.1.177
healthzBindAddress: 10.10.1.177:10256
metricsBindAddress: 10.10.1.177:10249
enableProfiling: true
clusterCIDR: 172.30.0.0/16
hostnameOverride: ubt-k8s-node-01
mode: "ipvs"
portRange: ""
kubeProxyIPTablesConfiguration:
  masqueradeAll: false
kubeProxyIPVSConfiguration:
  scheduler: rr
  excludeCIDRs: []
```



配置kube-proxy服务

```bash
# vim /etc/systemd/system/kube-proxy.service 
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/data/k8s/k8s/kube-proxy
ExecStart=/opt/k8s/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy-config.yaml \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```



```bash
# mkdir -p /data/k8s/k8s/kube-proxy
# modprobe ip_vs_rr
# systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy
```



查看监听端口

```bash
# netstat -lnpt|grep kube-prox
tcp        0      0 10.10.1.177:10249       0.0.0.0:*               LISTEN      17136/kube-proxy    
tcp        0      0 10.10.1.177:10256       0.0.0.0:*               LISTEN      17136/kube-proxy    
```



重启kubelet服务

```bash
#  systemctl restart kubelet
```



## 五、查看状态

5.1 查看节点状态

```bash
# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
k8s01       Ready    <none>   23d   v1.14.2
k8s02       Ready    <none>   23d   v1.14.2
k8s03       Ready    <none>   23d   v1.14.2
k8snode01   Ready    <none>   20m   v1.14.2#新节点加入了
```



5.2 查看CSR状态

```bash
# kubectl get csr
NAME        AGE     REQUESTOR                 CONDITION
csr-blfmw   6m26s   system:node:k8snode01     Pending
csr-h82bj   21m     system:bootstrap:ba9ucp   Approved,Issued
csr-sw724   2m51s   system:node:k8snode01     Pending
csr-x7r2d   21m     system:node:k8snode01     Pending
```



手动approve server cert csr

```bash
# kubectl certificate approve csr-x7r2d


# kubectl get csr
NAME        AGE     REQUESTOR                 CONDITION
csr-blfmw   10m     system:node:k8snode01     Approved,Issued
csr-h82bj   25m     system:bootstrap:ba9ucp   Approved,Issued
csr-sw724   6m53s   system:node:k8snode01     Approved,Issued
csr-x7r2d   25m     system:node:k8snode01     Approved,Issued
```



5.3 查看节点

![360截图18470124539074.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g8xpd5860aj31go0o7q5u.jpg)



5.4 创建pod测试

```yam
# cat nginx-prod.yml 
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-prod
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 8 # 8个副本
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



```bash
#执行
#kubectl create -f  nginx-prod.yml

#效果
```

![360截图18430708286124.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g8xpferg71j30t20d2ta5.jpg)

