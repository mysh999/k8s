[TOC]





## 一、部署方式说明

官方提供3种部署方式，

- minikube

  minikube是一种工具，在本地快速运行一个单点的k8s，不能用于生产

- kubeadm

  kubeadm用于快速部署一套k8s集群，目前是beta版

- 二进制包

  从官方下载发行版的二进制包，手动部署每个组件，组成k8s集群，目前生产环境大都使用



本文档使用kubeadm方式部署



## 二、准备条件

- OS: Centos7

- 内存2G，2Core CPU

- 3个节点，1个作为master，2个作为node

- 禁止swap分区

- mac地址和系统id唯一

  ```bash
  #查看系统id
  #  cat /sys/class/dmi/id/product_uuid 
  4CF54D56-305D-A3B4-61BE-E6BB4876C746
  ```

- 使用ansible部署



## 三、基本配置

未特别说明，3个节点都需要配置



| 角色   | IP              |
| ------ | --------------- |
| master | 192.168.255.11  |
| node1  | 192.168.255.12  |
| node2  | 192.168.255.13s |





3.1 配置hosts文件

```bash
# cat /etc/ansible/hosts 
[master]
192.168.255.11
[node]
192.168.255.12
192.168.255.13
192.168.255.11

# cat /root/hosts.yml 
- hosts: node
  remote_user: root
  tasks:
     - name: Copy hosts file
       copy: src=/root/hosts dest=/etc/hosts
       tags:
       - only
       
# ansible-playbook hosts.yml 
```



3.2 配置主机名

```shell
# ansible 192.168.255.11 -m shell -a 'echo "k8s-master">/etc/hostname'
# ansible 192.168.255.12 -m shell -a 'echo "k8s-node1">/etc/hostname'  
# ansible 192.168.255.13 -m shell -a 'echo "k8s-node2">/etc/hostname'  
```



3.3 初始化配置

```bash
# cat /root/ntp.conf 
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
server ntp2.aliyun.com iburst
server ntp3.aliyun.com iburst
server ntp4.aliyun.com iburst
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys


# cat k8s_init.yml 
- hosts: node
  remote_user: root
  tasks:
  - name: disable selinux
    lineinfile:
      path: /etc/selinux/config
      regexp: '^SELINUX='
      line: SELINUX=disabled                    
        
  - name: disable firewall
    shell: |
      systemctl stop firewalld
      systemctl disable firewalld
        
  - name: set timezone to Asia/Shanghai 
    shell: timedatectl set-timezone Asia/Shanghai
 
  - name: sync hwclock 
    shell: hwclock -w

  - name: install ntp
    yum: 
      name: ntp
    become: yes
        
  - name: copy ntp configure
    template: src=/root/ntp.conf dest=/etc/ntp.conf

  - name: reload 
    shell: systemctl daemon-reload 

  - name: Start ntp service
    service:
      name: ntpd
      daemon_reload: yes
      state: started
      enabled: yes
        
  - name: config fstab
    lineinfile:
      path: /etc/fstab
      regexp: '^/dev/mapper/centos-swap swap '
      line: '#/dev/mapper/centos-swap swap                    swap    defaults        0 0'
          
  - name: disable swap
    command: swapoff -a
        
  - name: Install yum utils
    yum:
      name: yum-utils
      state: latest

  - name: install opvsadm
    yum:
      name: ipvsadm
      state: present
  - name: config ipvs kernel
    shell: |
        modprobe -- ip_vs
        modprobe -- ip_vs_rr
        modprobe -- ip_vs_wrr
        modprobe -- ip_vs_sh
        modprobe -- nf_conntrack_ipv4

  - name: config ip_forward
    lineinfile:
      path: /etc/sysctl.d/k8s.conf
      line: "{{ item }}"
      create: yes
      state: present
    with_items:
      - 'net.bridge.bridge-nf-call-ip6tables = 1'
      - 'net.bridge.bridge-nf-call-iptables = 1'
      - 'net.ipv4.ip_forward = 1'
      - 'vm.swappiness = 0'


  - name: install epel 
    yum:
        name: ['epel-release', 'conntrack', 'ipvsadm','ipset','jq','sysstat']
        state: present
        disable_gpg_check: yes

  - name: enable mode 
    shell: |
      modprobe br_netfilter 
      modprobe ip_vs

  - name: take effect
    shell: sysctl -p /etc/sysctl.d/k8s.conf

  - name: Install device-mapper-persistent-data
    yum:
       name: device-mapper-persistent-data
       state: latest

  - name: Install lvm2
    yum:
        name: lvm2
        state: latest

  - name: Add Docker repo
    get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docer-ce.repo
    become: yes

  - name: Enable Docker Edge repo
    ini_file:
        dest: /etc/yum.repos.d/docer-ce.repo
        section: 'docker-ce-edge'
        option: enabled
        value: 0
    become: yes

  - name: Enable Docker Test repo
    ini_file:
        dest: /etc/yum.repos.d/docer-ce.repo
        section: 'docker-ce-test'
        option: enabled
        value: 0
    become: yes

  - name: Install Docker
    package:
        name: docker-ce
        state: latest
    become: yes

  - name: Start Docker service
    service:
        name: docker
        state: started
        enabled: yes
    become: yes
        


  - name: add kube repo
    lineinfile:
        path: /etc/yum.repos.d/kubernetes.repo
        line: '{{ item }}'
        create: yes
        state: present
    with_items:
        - '[kubernetes]'
        - 'name=Kubernetes'
        - 'baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64'
        - 'enabled=1'
        - 'gpgcheck=1'
        - 'repo_gpgcheck=1'
        - 'gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg'
    
  - name: install kube
    yum: 
        name: ['kubeadm', 'kubelet', 'kubectl']
        state: present
        disable_gpg_check: yes
    
  - name: enable kubelet
    shell: systemctl enable kubelet
        
  - name: start kubelet
    shell: systemctl start kubelet
```



3.4  准备镜像

```bash
#### 版本信息
#K8S_VERSION=v1.11.2
#ETCD_VERSION=3.2.18
#DASHBOARD_VERSION=v1.8.3
#FLANNEL_VERSION=v0.10.0-amd64
#DNS_VERSION=1.14.8
#PAUSE_VERSION=3.1

### 基本组件
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:$PAUSE_VERSION

#### 网络
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64:$DNS_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64:$DNS_VERSION
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64:$DNS_VERSION
#docker pull quay.io/coreos/flannel:$FLANNEL_VERSION

#### 前端
#docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:$DASHBOARD_VERSION

### 修改tag
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION k8s.gcr.io/kube-apiserver-amd64:$K8S_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION k8s.gcr.io/kube-controller-manager-amd64:$K8S_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION k8s.gcr.io/kube-scheduler-amd64:$K8S_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION k8s.gcr.io/kube-proxy-amd64:$K8S_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION k8s.gcr.io/etcd-amd64:$ETCD_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:$PAUSE_VERSION k8s.gcr.io/pause-amd64:$PAUSE_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64:$DNS_VERSION k8s.gcr.io/k8s-dns-sidecar-amd64:$DNS_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64:$DNS_VERSION k8s.gcr.io/k8s-dns-kube-dns-amd64:$DNS_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64:$DNS_VERSION k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:$DNS_VERSION
#docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:$DASHBOARD_VERSION k8s.gcr.io/kubernetes-dashboard-amd64:$DASHBOARD_VERSION
```

![k8s_1.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g71ned38ngj31200i0q4j.jpg)

## 四、初始化master

4.1 在master上运行

```bash
[root@k8s-master ~]# kubeadm init --kubernetes-version=1.15.3 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.255.11  

[init] Using Kubernetes version: v1.15.3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.2. Latest validated version: 18.09
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'



[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.255.11]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.255.11 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.255.11 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.001595 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 36runb.g85f3g7bamgvknmn
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.255.11:6443 --token 36runb.g85f3g7bamgvknmn \
    --discovery-token-ca-cert-hash sha256:5d674a44c532cf1d966e69637ff34bdc62b3f8da747bc403ad9c2678090b60c2 
```



4.2  master按照提示执行

```bash
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]#  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



4.3 master和node节点都执行以下命令安装pod 网络插件

```bash
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```



4.4 node节点执行以下命令加入到集群中

```bash
# kubeadm join 192.168.255.11:6443 --token 36runb.g85f3g7bamgvknmn --discovery-token-ca-cert-hash sha256:5d674a44c532cf1d966e69637ff34bdc62b3f8da747bc403ad9c2678090b60c2
```



4.5 master部署nginx

```bash
# 部署一个 Nginx Deployment，包含两个Pod
# https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

[root@k8s-master ~]#kubectl create deployment nginx --image=nginx:alpine
[root@k8s-master ~]#kubectl scale deployment nginx --replicas=2
[root@k8s-master ~]#kubectl expose deployment nginx --port=80 --type=NodePort

[root@k8s-master ~]# kubectl get pods -l app=nginx -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP           NODE        NOMINATED NODE   READINESS GATES
nginx-8f6959bd-djbxj   1/1     Running   0          101m   10.244.1.5   k8s-node1   <none>           <none>
nginx-8f6959bd-tmwrs   1/1     Running   0          101m   10.244.2.2   k8s-node2   <none>           <none>

[root@k8s-master ~]# kubectl get services nginx
NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.99.191.4   <none>        80:32080/TCP   99m


```



4.6 部署dashboard

```bash
 wget  https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

修改以下内容:

```bash
[root@k8s-master ~]# more kubernetes-dashboard.yaml 
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```



创建

```bash
[root@k8s-master ~]# kubectl apply -f kubernetes-dashboard.yaml                                                                                      
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```



查看

```bash
[root@k8s-master ~]# kubectl get pods -n kube-system -o wide   
NAME                                    READY   STATUS    RESTARTS   AGE    IP               NODE         NOMINATED NODE   READINESS GATES
calico-node-fzh54                       2/2     Running   0          61m    192.168.255.13   k8s-node2    <none>           <none>
calico-node-swcfh                       2/2     Running   0          61m    192.168.255.11   k8s-master   <none>           <none>
calico-node-zhcn6                       2/2     Running   0          61m    192.168.255.12   k8s-node1    <none>           <none>
coredns-5c98db65d4-pdkj6                1/1     Running   0          177m   10.244.1.4       k8s-node1    <none>           <none>
coredns-5c98db65d4-pql6l                1/1     Running   0          177m   10.244.1.3       k8s-node1    <none>           <none>
etcd-k8s-master                         1/1     Running   0          176m   192.168.255.11   k8s-master   <none>           <none>
kube-apiserver-k8s-master               1/1     Running   0          176m   192.168.255.11   k8s-master   <none>           <none>
kube-controller-manager-k8s-master      1/1     Running   0          176m   192.168.255.11   k8s-master   <none>           <none>
kube-flannel-ds-p5n9c                   1/1     Running   0          59m    192.168.255.13   k8s-node2    <none>           <none>
kube-flannel-ds-z6rsn                   1/1     Running   0          60m    192.168.255.12   k8s-node1    <none>           <none>
kube-flannel-ds-zlmz2                   1/1     Running   0          59m    192.168.255.11   k8s-master   <none>           <none>
kube-proxy-glxr6                        1/1     Running   0          177m   192.168.255.11   k8s-master   <none>           <none>
kube-proxy-hk2nb                        1/1     Running   1          172m   192.168.255.13   k8s-node2    <none>           <none>
kube-proxy-l6qpp                        1/1     Running   1          172m   192.168.255.12   k8s-node1    <none>           <none>
kube-scheduler-k8s-master               1/1     Running   0          176m   192.168.255.11   k8s-master   <none>           <none>
kubernetes-dashboard-7d75c474bb-4p8ql   1/1     Running   0          3s     10.244.2.7       k8s-node2    <none>           <none>

[root@k8s-master ~]#  kubectl get secret -n kube-system
NAME                                             TYPE                                  DATA   AGE
attachdetach-controller-token-69jz4              kubernetes.io/service-account-token   3      178m
bootstrap-signer-token-9t9sp                     kubernetes.io/service-account-token   3      178m
bootstrap-token-36runb                           bootstrap.kubernetes.io/token         7      178m
calico-node-token-b8cg8                          kubernetes.io/service-account-token   3      62m
certificate-controller-token-4l8vq               kubernetes.io/service-account-token   3      178m
clusterrole-aggregation-controller-token-h4l4v   kubernetes.io/service-account-token   3      178m
coredns-token-96r26                              kubernetes.io/service-account-token   3      178m
cronjob-controller-token-7tn9k                   kubernetes.io/service-account-token   3      178m
daemon-set-controller-token-x2gs4                kubernetes.io/service-account-token   3      178m
dashboard-admin-token-gnphd                      kubernetes.io/service-account-token   3      162m
default-token-4847b                              kubernetes.io/service-account-token   3      178m
deployment-controller-token-ksjs6                kubernetes.io/service-account-token   3      178m
disruption-controller-token-jrfmx                kubernetes.io/service-account-token   3      178m
endpoint-controller-token-r9gjl                  kubernetes.io/service-account-token   3      178m
expand-controller-token-q6vs6                    kubernetes.io/service-account-token   3      178m
flannel-token-qh67k                              kubernetes.io/service-account-token   3      175m
generic-garbage-collector-token-55nq4            kubernetes.io/service-account-token   3      178m
horizontal-pod-autoscaler-token-5n8vj            kubernetes.io/service-account-token   3      178m
job-controller-token-rblkw                       kubernetes.io/service-account-token   3      178m
kube-proxy-token-4kj7w                           kubernetes.io/service-account-token   3      178m
kubernetes-dashboard-certs                       Opaque                                0      45s
kubernetes-dashboard-key-holder                  Opaque                                2      6m37s
kubernetes-dashboard-token-swkvv                 kubernetes.io/service-account-token   3      45s
namespace-controller-token-wq8q2                 kubernetes.io/service-account-token   3      178m
node-controller-token-c9vvz                      kubernetes.io/service-account-token   3      178m
persistent-volume-binder-token-5lr74             kubernetes.io/service-account-token   3      178m
pod-garbage-collector-token-sjzl2                kubernetes.io/service-account-token   3      178m
pv-protection-controller-token-frrs6             kubernetes.io/service-account-token   3      178m
pvc-protection-controller-token-nc5sg            kubernetes.io/service-account-token   3      178m
replicaset-controller-token-pcp6x                kubernetes.io/service-account-token   3      178m
replication-controller-token-8gk7r               kubernetes.io/service-account-token   3      178m
resourcequota-controller-token-xsdn9             kubernetes.io/service-account-token   3      178m
service-account-controller-token-l7nsb           kubernetes.io/service-account-token   3      178m
service-controller-token-mvd4w                   kubernetes.io/service-account-token   3      178m
statefulset-controller-token-8f4rq               kubernetes.io/service-account-token   3      178m
token-cleaner-token-4bn6b                        kubernetes.io/service-account-token   3      178m
ttl-controller-token-kplvn                       kubernetes.io/service-account-token   3      178m
```



4.7 创建管理员角色

```bash
[root@k8s-master ~]# cat /root/k8s-admin.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
  
[root@k8s-master ~] # kubectl apply -f k8s-admin.yaml
```



4.8 使用上述创建账号的token登录Kubernetes Dashboard

```bash
[root@k8s-master ~]#  kubectl get secret -n kube-system
NAME                                             TYPE                                  DATA   AGE
attachdetach-controller-token-69jz4              kubernetes.io/service-account-token   3      16h
bootstrap-signer-token-9t9sp                     kubernetes.io/service-account-token   3      16h
bootstrap-token-36runb                           bootstrap.kubernetes.io/token         7      16h
calico-node-token-b8cg8                          kubernetes.io/service-account-token   3      14h
certificate-controller-token-4l8vq               kubernetes.io/service-account-token   3      16h
clusterrole-aggregation-controller-token-h4l4v   kubernetes.io/service-account-token   3      16h
coredns-token-96r26                              kubernetes.io/service-account-token   3      16h
cronjob-controller-token-7tn9k                   kubernetes.io/service-account-token   3      16h
daemon-set-controller-token-x2gs4                kubernetes.io/service-account-token   3      16h
dashboard-admin-token-gnphd                      kubernetes.io/service-account-token   3      16h
default-token-4847b                              kubernetes.io/service-account-token   3      16h
deployment-controller-token-ksjs6                kubernetes.io/service-account-token   3      16h
disruption-controller-token-jrfmx                kubernetes.io/service-account-token   3      16h
endpoint-controller-token-r9gjl                  kubernetes.io/service-account-token   3      16h
expand-controller-token-q6vs6                    kubernetes.io/service-account-token   3      16h
flannel-token-qh67k                              kubernetes.io/service-account-token   3      16h
generic-garbage-collector-token-55nq4            kubernetes.io/service-account-token   3      16h
horizontal-pod-autoscaler-token-5n8vj            kubernetes.io/service-account-token   3      16h
job-controller-token-rblkw                       kubernetes.io/service-account-token   3      16h
kube-proxy-token-4kj7w                           kubernetes.io/service-account-token   3      16h
kubernetes-dashboard-certs                       Opaque                                0      13h
kubernetes-dashboard-key-holder                  Opaque                                2      13h
kubernetes-dashboard-token-swkvv                 kubernetes.io/service-account-token   3      13h
namespace-controller-token-wq8q2                 kubernetes.io/service-account-token   3      16h
node-controller-token-c9vvz                      kubernetes.io/service-account-token   3      16h
persistent-volume-binder-token-5lr74             kubernetes.io/service-account-token   3      16h
pod-garbage-collector-token-sjzl2                kubernetes.io/service-account-token   3      16h
pv-protection-controller-token-frrs6             kubernetes.io/service-account-token   3      16h
pvc-protection-controller-token-nc5sg            kubernetes.io/service-account-token   3      16h
replicaset-controller-token-pcp6x                kubernetes.io/service-account-token   3      16h
replication-controller-token-8gk7r               kubernetes.io/service-account-token   3      16h
resourcequota-controller-token-xsdn9             kubernetes.io/service-account-token   3      16h
service-account-controller-token-l7nsb           kubernetes.io/service-account-token   3      16h
service-controller-token-mvd4w                   kubernetes.io/service-account-token   3      16h
statefulset-controller-token-8f4rq               kubernetes.io/service-account-token   3      16h
token-cleaner-token-4bn6b                        kubernetes.io/service-account-token   3      16h
ttl-controller-token-kplvn                       kubernetes.io/service-account-token   3      16h


[root@k8s-master ~]#  kubectl describe secret dashboard-admin-token-gnphd -n kube-system
Name:         dashboard-admin-token-gnphd
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: b9f97b4a-b934-442c-8501-1e369c5a158f

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tZ25waGQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjlmOTdiNGEtYjkzNC00NDJjLTg1MDEtMWUzNjljNWExNThmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.cUUAOSRktnXydy1TgODi3sZGTE7DJz0uxmreJg6uPDltZxFMN-N4_iMDOPqYhG6nx6_9g-LnZSqenU1oZ-njCdWkj7dquJMBXUbTlHX-G5_rp7apXiix6aPCUKIcNz16dFJ9WuBRhOHu5bCf2f5isyxig_66BcJD0m9K3fQi5zx3EA03b53WQUQzJez-pmKAu7-H0XK_LA0QP2jf72oKMsWc-9JtqL4H9TXtm5tSuflItHxdBLfKB3heTAPIZyrHSizEkzNo1OYb2HP6jlrOdQW1jnolm2mWTMaYabmJmGK0MpEKO8YXmd5Mu4er6rtBzMsFrpJdBmov4dSKMG7C8Q
ca.crt:     1025 bytes
namespace:  11 bytes

[root@k8s-master ~]# kubectl get services nginx
NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.99.191.4   <none>        80:32080/TCP   99m
[root@k8s-master ~]# kubectl get pods -l app=nginx -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP           NODE        NOMINATED NODE   READINESS GATES
nginx-8f6959bd-djbxj   1/1     Running   0          101m   10.244.1.5   k8s-node1   <none>           <none>
nginx-8f6959bd-tmwrs   1/1     Running   0          101m   10.244.2.2   k8s-node2   <none>           <none>
```



## 五、登陆k8s

使用firefox浏览器打开

https://192.168.255.12:30001

使用上述的token令牌登陆 

![k8s_2.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72a7l4fbvj31hm0modgw.jpg)

主界面

![k8s_3.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72a8bz8t2j31h10sttao.jpg)





## 六、创建容器

点击创建按钮

![k8s_4.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72iphrx3pj31gz0j83zv.jpg)



![k8s_5.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72irowd3wj31h00nn406.jpg)



![k8s_6.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72isj8n5bj31gb0rw40p.jpg)

创建完毕

![k8s_7.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72iuucmlnj31gl0q7wgt.jpg)



## 七、集群测试

7.1 测试一：把node2关机

node2状态离线

![k8s_8.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72j6bg4noj31h00l4jta.jpg)



观察redis_prod三副本，2个在node2上的变成灰色，新建2个在master上

![k8s_9.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72j82hxcxj31gh0qe76y.jpg)



启动node2后，redis_prod恢复成3副本

![k8s_10.png](http://ww1.sinaimg.cn/large/007Xg1efgy1g72ji38a4hj31g80qb76n.jpg)