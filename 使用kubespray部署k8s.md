## 一、前提条件

- 3台VM
- CentOS7.7
- 关闭防火墙
- 关闭selinux



## 二、部署过程

未特别说明，均在master01上执行

### 2.1、下载包

```bash
# wget https://github.com/kubernetes-sigs/kubespray/archive/v2.13.1.tar.gz
# tar -zxvf v2.13.1.tar.gz
```



### 2.2、安装依赖

```bash
# cd kubespray-2.13.1/
# yum install -y epel-release python3-pip
# pip3 install -r requirements.txt
```

如果报

```bash
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-build-fhrtu4d4/cryptography/setup.py", line 14, in <module>
        from setuptools_rust import RustExtension
    ModuleNotFoundError: No module named 'setuptools_rust'
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-fhrtu4d4/cryptography/
```



解决办法：

```bash
# pip3 install --upgrade setuptools
# python3 -m pip install --upgrade pip
# pip3 install -r requirements.txt
```



### 2.3、更新 Ansible inventory file，IPS地址为3个服务器IP

```bash
# cp -rfp inventory/sample inventory/mycluster
# declare -a IPS=(192.168.188.71 192.168.188.72 192.168.188.73)
# CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```



查看自动生成的hosts.yaml，kubespray会根据提供的节点数量自动规划节点角色。这里部署2个master节点，同时3个节点也作为node，3个节点也用来部署etcd

```bash
# cat inventory/mycluster/hosts.yaml 
all:
  hosts:
    node1:
      ansible_host: 192.168.188.71
      ip: 192.168.188.71
      access_ip: 192.168.188.71
    node2:
      ansible_host: 192.168.188.72
      ip: 192.168.188.72
      access_ip: 192.168.188.72
    node3:
      ansible_host: 192.168.188.73
      ip: 192.168.188.73
      access_ip: 192.168.188.73
  children:
    kube-master:
      hosts:
        node1:
        node2:
    kube-node:
      hosts:
        node1:
        node2:
        node3:
    etcd:
      hosts:
        node1:
        node2:
        node3:
    k8s-cluster:
      children:
        kube-master:
        kube-node:
    calico-rr:
      hosts: {}
```



### 2.4、默认安装版本较低，指定kubernetes版本

```bash
# vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
kube_version: v1.16.7
```



### 2.5、配置ssh免密

```bash
# ssh-keygen
# ssh-copy-id 192.168.188.71
# ssh-copy-id 192.168.188.72
# ssh-copy-id 192.168.188.73
```



### 2.6、将google镜像改成国内源(可选)

在inventory/mycluster/group_vars/all/all.yml和inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml添加以下内容

```bash
kube_image_repo: "registry.cn-hangzhou.aliyuncs.com/google_containers"
calico_policy_image_repo: "calico/kube-controllers"
pod_infra_image_repo: "mirrorgooglecontainers/pause-{{ image_arch }}"
kubedns_image_repo: "google_containers/k8s-dns-kube-dns-{{ image_arch }}"
dnsmasq_nanny_image_repo: "google_containers/k8s-dns-dnsmasq-nanny-{{ image_arch }}"
dnsmasq_sidecar_image_repo: "google_containers/k8s-dns-sidecar-{{ image_arch }}"
dnsmasqautoscaler_image_repo: "mirrorgooglecontainers/cluster-proportional-autoscaler-{{ image_arch }}"
dnsautoscaler_image_repo: "mirrorgooglecontainers/cluster-proportional-autoscaler-{{ image_arch }}"
registry_proxy_image_repo: "google_containers/kube-registry-proxy"
dashboard_image_repo: "mirrorgooglecontainers/kubernetes-dashboard-{{ image_arch }}"
```





### 2.7、运行kubespray playbook安装集群

```bash
# ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

说明：如果由于网络因素导致无法从外网下载镜像，可以导入离线image再执行一遍命令





### 2.8、验证效果

```bash
# kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    master   85m   v1.16.7
node2   Ready    master   84m   v1.16.7
node3   Ready    <none>   83m   v1.16.7


# kubectl -n kube-system get pod
NAME                                          READY   STATUS    RESTARTS   AGE
calico-kube-controllers-846df66b54-jfhjq      1/1     Running   1          3h25m
calico-node-dfkw9                             1/1     Running   2          3h26m
calico-node-zfntj                             1/1     Running   2          3h26m
calico-node-zzbw5                             1/1     Running   2          3h26m
coredns-76798d84dd-g7r5r                      1/1     Running   1          3h25m
coredns-76798d84dd-skrb9                      1/1     Running   1          3h25m
dns-autoscaler-56549847b5-x4mwg               1/1     Running   1          3h25m
kube-apiserver-node1                          1/1     Running   1          3h27m
kube-apiserver-node2                          1/1     Running   1          3h27m
kube-controller-manager-node1                 1/1     Running   1          3h27m
kube-controller-manager-node2                 1/1     Running   2          3h27m
kube-proxy-9xrvd                              1/1     Running   1          3h27m
kube-proxy-jxr64                              1/1     Running   1          3h27m
kube-proxy-ksbhm                              1/1     Running   1          3h27m
kube-scheduler-node1                          1/1     Running   1          3h27m
kube-scheduler-node2                          1/1     Running   2          3h27m
kubernetes-dashboard-5684cc8b68-h8lwf         1/1     Running   2          3h25m
kubernetes-metrics-scraper-747b4fd5cd-pv6rm   1/1     Running   1          3h25m
nginx-proxy-node3                             1/1     Running   1          3h27m
nodelocaldns-g86lb                            1/1     Running   1          3h25m
nodelocaldns-k76hh                            1/1     Running   1          3h25m
nodelocaldns-q47s2                            1/1     Running   1          3h25m

[root@node1 opt]# kubectl get services --all-namespaces
NAMESPACE     NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes                  ClusterIP   10.233.0.1      <none>        443/TCP                  3h41m
kube-system   coredns                     ClusterIP   10.233.0.3      <none>        53/UDP,53/TCP,9153/TCP   3h38m
kube-system   dashboard-metrics-scraper   ClusterIP   10.233.16.72    <none>        8000/TCP                 3h38m
kube-system   kubernetes-dashboard        ClusterIP   10.233.33.172   <none>        443/TCP                  3h38m
[root@node1 opt]# kubectl get pods --all-namespaces    
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
default       busybox                                       1/1     Running   1          86m
kube-system   calico-kube-controllers-846df66b54-jfhjq      1/1     Running   1          3h39m
kube-system   calico-node-dfkw9                             1/1     Running   2          3h39m
kube-system   calico-node-zfntj                             1/1     Running   2          3h39m
kube-system   calico-node-zzbw5                             1/1     Running   2          3h39m
kube-system   coredns-76798d84dd-g7r5r                      1/1     Running   1          3h38m
kube-system   coredns-76798d84dd-skrb9                      1/1     Running   1          3h38m
kube-system   dns-autoscaler-56549847b5-x4mwg               1/1     Running   1          3h38m
kube-system   kube-apiserver-node1                          1/1     Running   1          3h41m
kube-system   kube-apiserver-node2                          1/1     Running   1          3h41m
kube-system   kube-controller-manager-node1                 1/1     Running   1          3h41m
kube-system   kube-controller-manager-node2                 1/1     Running   2          3h41m
kube-system   kube-proxy-9xrvd                              1/1     Running   1          3h40m
kube-system   kube-proxy-jxr64                              1/1     Running   1          3h40m
kube-system   kube-proxy-ksbhm                              1/1     Running   1          3h40m
kube-system   kube-scheduler-node1                          1/1     Running   1          3h41m
kube-system   kube-scheduler-node2                          1/1     Running   2          3h41m
kube-system   kubernetes-dashboard-5684cc8b68-h8lwf         1/1     Running   2          3h38m
kube-system   kubernetes-metrics-scraper-747b4fd5cd-pv6rm   1/1     Running   1          3h38m
kube-system   nginx-proxy-node3                             1/1     Running   1          3h40m
kube-system   nodelocaldns-g86lb                            1/1     Running   1          3h38m
kube-system   nodelocaldns-k76hh                            1/1     Running   1          3h38m
kube-system   nodelocaldns-q47s2                            1/1     Running   1          3h38m
```



## 三、部署dashboard

### 

增加RBAC，创建文件

```bash
# cat admin-user.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
  
  
# cat admin-user-role.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system  
```



执行创建

```bash
#kubectl create -f  admin-user.yaml  &&kubectl create -f admin-user-role.yaml 
```



获取token看，用于登录dashboard页面

```bash
# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-hvdv8
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: c6587b81-92da-4050-a022-2dde68373f77

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjVjV2hvM0lhLWNBYTkzQTNGREtRNDdYZVRBNFVWOFluaG9xNTIxT0h1QTAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWh2ZHY4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjNjU4N2I4MS05MmRhLTQwNTAtYTAyMi0yZGRlNjgzNzNmNzciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.Y-x9tEdc0A65Y8JuAnrwQPjXGtEdvLwq3Ynfw4ovPC6hn0HgduL_YjfO18zQwoIT20Il8OHT2q0Xj0UYXkKxbazn0VTloRLabazbemh2TgL45nxFPLXhUU1MOBftuVLBot0zP_gxBj016-bLDKXKejTSIRjFJZPpnJAw4rc8o74Oesis3XsWSKRsa01vMdOzibJ941oOIeUYgpz1HXbUVnoO5EN_kLtWkrXUBVyHASJe8X0c_4jKzxgYdCd_ESQmk8BUElg7NcSNNB66qLPeXzm7JoAo9WktFxB7ko4mZ_tFCGALOjrKWInHEX961BSqB2jm5yeVb1GBQLJoo-rhyw
```



登陆

```html
https://192.168.188.71:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```



如果出现forbidden无法访问提示，提示内容如下：

```bash
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
  },
  "status": "Failure",
  "message": "services \"https:kubernetes-dashboard:\" is forbidden: User \"system:anonymous\" cannot get resource \"services/proxy\" in API group \"\" in the namespace \"kube-system\"",
  "reason": "Forbidden",
  "details": {
    "name": "https:kubernetes-dashboard:",
    "kind": "services"
  },
  "code": 403
}
```



因为kubernetes基于安全性的考虑，浏览器必须要一个根证书，防止中间人攻击

解决办法:

方法一：

创建浏览器证书

```bash
#生成crt
grep 'client-certificate-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

#生成key文件
grep 'client-key-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

# 生成p12证书文件（证书的生成和导入需要一个密码）
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

将该证书kubecfg.p12导入浏览器即可正常访问



方法二：

让system:anonymous用户可以访问

```bash
# cat al.yaml 
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-anonymous
rules:
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["https:kubernetes-dashboard:"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- nonResourceURLs: ["/ui", "/ui/*", "/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-anonymous
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard-anonymous
subjects:
- kind: User
  name: system:anonymous
  
  # kubectl create -f al.yaml
```







## 四、部署第三方dashboard(kuboard)

执行以下命令

```bash
# kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
# kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.6/metrics-server.yaml


# kubectl get pods --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
default       busybox                                       1/1     Running   3          3h48m
kube-system   calico-kube-controllers-846df66b54-jfhjq      1/1     Running   1          6h1m
kube-system   calico-node-dfkw9                             1/1     Running   2          6h2m
kube-system   calico-node-zfntj                             1/1     Running   2          6h2m
kube-system   calico-node-zzbw5                             1/1     Running   2          6h2m
kube-system   coredns-76798d84dd-g7r5r                      1/1     Running   1          6h
kube-system   coredns-76798d84dd-skrb9                      1/1     Running   1          6h
kube-system   dns-autoscaler-56549847b5-x4mwg               1/1     Running   1          6h
kube-system   kube-apiserver-node1                          1/1     Running   1          6h3m
kube-system   kube-apiserver-node2                          1/1     Running   1          6h3m
kube-system   kube-controller-manager-node1                 1/1     Running   1          6h3m
kube-system   kube-controller-manager-node2                 1/1     Running   2          6h3m
kube-system   kube-proxy-9xrvd                              1/1     Running   1          6h2m
kube-system   kube-proxy-jxr64                              1/1     Running   1          6h2m
kube-system   kube-proxy-ksbhm                              1/1     Running   1          6h2m
kube-system   kube-scheduler-node1                          1/1     Running   1          6h3m
kube-system   kube-scheduler-node2                          1/1     Running   2          6h3m
kube-system   kubernetes-dashboard-5684cc8b68-h8lwf         1/1     Running   2          6h
kube-system   kubernetes-metrics-scraper-747b4fd5cd-pv6rm   1/1     Running   1          6h
kube-system   kuboard-59bdf4d5fb-zxbph                      1/1     Running   0          12m           #新的
kube-system   metrics-server-56b49c5f5b-bn64q               1/1     Running   0          11m     #新的
kube-system   nginx-proxy-node3                             1/1     Running   1          6h2m
kube-system   nodelocaldns-g86lb                            1/1     Running   1          6h
kube-system   nodelocaldns-k76hh                            1/1     Running   1          6h
kube-system   nodelocaldns-q47s2                            1/1     Running   1          6h
```



登陆

```htm
http://任意一个Worker节点的IP地址:32567/
```



