## 一、目标

使用NFS实现k8s集群数据的持久化



## 二、概念

**Persistent Volume（持久卷）** 和 **Persistent Volume Claim（持久卷消费者）**。

Persistent Volume（PV）是集群之中的一块网络存储。跟 Node 一样，也是集群的资源。PV 跟 Volume (卷) 类似，不过会有独立于 Pod 的生命周期。这一 API 对象包含了存储的实现细节，例如 NFS、iSCSI 或者其他的云提供商的存储系统。



Persistent Volume Claim (PVC) 是用户的一个请求。跟 Pod 类似，Pod 消费 Node 的资源，PVC 消费 PV 的资源。Pod 能够申请特定的资源（CPU 和内存）；Claim 能够请求特定的尺寸和访问模式（例如可以加载一个读写，以及多个只读实例



## 三、NFS服务器准备

单独准备一台NFS服务器共享

```bash
#安装包
#yum -y install rpcbind nfs-utils

# vi /etc/exports
/usr/local/kubernetes/volumes *(rw,sync,no_subtree_check,no_root_squash)
/usr/local/kubernetes/v2 *(rw,sync,no_subtree_check,no_root_squash)
/usr/local/kubernetes/v3 *(rw,sync,no_subtree_check,no_root_squash)
/usr/local/kubernetes/v4 *(rw,sync,no_subtree_check,no_root_squash)

# systemctl restart nfs     
# systemctl restart rpcbind.service
```



## 四、K8S部署

4.1 每台客户端安装NFS

```bash
# yum -y install nfs-utils
# mount -t nfs 10.10.1.177:/usr/local/kubernetes/v2 /usr/local/kubernetes/v2
```





4.2 定义PV

创建一个名为 `nfs-pv.yml ` 的配置文件

```bash
# cat nfs-pv.yml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-mysql
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v2
  labels:
    name: nfs-pv-v2
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v2"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v3
  labels:
    name: nfs-pv-v3
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v3"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v4
  labels:
    name: nfs-pv-v4
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v4"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
    
    
   
#创建
#kubectl create -f nfs-pv.yml 

#查看
# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
nfs-pv-mysql   5Gi        RWX            Recycle          Bound    default/nfs-pvc-mysql-myshop                           8h
nfs-pv-v2      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v2                                     145m
nfs-pv-v3      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v3                                     145m
nfs-pv-v4      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v4                                     145m


```



4.3  定义PVC

```bash
# cat nfs-pvc.yml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-mysql-myshop
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v2
  labels:
    name: nfs-pv-v2
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v3
  labels:
    name: nfs-pv-v3
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 3Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v4
  labels:
    name: nfs-pv-v4
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 4Gi



#创建
#kubectl create -f nfs-pvc.yml 

#查看
#  kubectl get pvc
NAME                   STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-mysql-myshop   Bound    nfs-pv-mysql   5Gi        RWX                           8h
nfs-pvc-v2             Bound    nfs-pv-v2      5Gi        RWX                           143m
nfs-pvc-v3             Bound    nfs-pv-v3      5Gi        RWX                           143m
nfs-pvc-v4             Bound    nfs-pv-v4      5Gi        RWX                           143m
```





4.4 部署mysql

```bash
# cat nfs-pv.yml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-mysql
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v2
  labels:
    name: nfs-pv-v2
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v2"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v3
  labels:
    name: nfs-pv-v3
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v3"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-v4
  labels:
    name: nfs-pv-v4
spec:
  # 设置容量
  capacity:
    storage: 5Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/v4"
    # NFS 服务端地址
    server: 10.10.1.177
    readOnly: false
[root@ubt-k8s-master-01 work]# cat nfs-pv.yml ^C
[root@ubt-k8s-master-01 work]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
nfs-pv-mysql   5Gi        RWX            Recycle          Bound    default/nfs-pvc-mysql-myshop                           8h
nfs-pv-v2      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v2                                     145m
nfs-pv-v3      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v3                                     145m
nfs-pv-v4      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v4                                     145m
[root@ubt-k8s-master-01 work]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
nfs-pv-mysql   5Gi        RWX            Recycle          Bound    default/nfs-pvc-mysql-myshop                           8h
nfs-pv-v2      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v2                                     145m
nfs-pv-v3      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v3                                     145m
nfs-pv-v4      5Gi        RWX            Recycle          Bound    default/nfs-pvc-v4                                     145m
[root@ubt-k8s-master-01 work]# cat nfs-pvc.yml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-mysql-myshop
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v2
  labels:
    name: nfs-pv-v2
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v3
  labels:
    name: nfs-pv-v3
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 3Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-v4
  labels:
    name: nfs-pv-v4
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 4Gi
[root@ubt-k8s-master-01 work]#  kubectl get pvc
NAME                   STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-mysql-myshop   Bound    nfs-pv-mysql   5Gi        RWX                           8h
nfs-pvc-v2             Bound    nfs-pv-v2      5Gi        RWX                           143m
nfs-pvc-v3             Bound    nfs-pv-v3      5Gi        RWX                           143m
nfs-pvc-v4             Bound    nfs-pv-v4      5Gi        RWX                           143m

[root@ubt-k8s-master-01 work]# cat mysql.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql-myshop
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: mysql-myshop
    spec:
      containers:
        - name: mysql-myshop
          image: mysql:5.7
          # 只有镜像不存在时，才会进行镜像拉取
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
          # 同 Docker 配置中的 environment
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
          # 容器中的挂载目录
          volumeMounts:
            - name: nfs-vol-myshop
              mountPath: /var/lib/mysql
      volumes:
        # 挂载到数据卷
        - name: nfs-vol-myshop
          persistentVolumeClaim:
            claimName: nfs-pvc-mysql-myshop
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-myshop
spec:
  ports:
    - port: 3306
      targetPort: 3306
  type: LoadBalancer
  selector:
    name: mysql-myshop

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: mysql-v2
    spec:
      containers:
        - name: mysql-v2
          image: mysql:5.7
          # 只有镜像不存在时，才会进行镜像拉取
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
          # 同 Docker 配置中的 environment
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
          # 容器中的挂载目录
          volumeMounts:
            - name: nfs-vol-v2
              mountPath: /var/lib/mysql
      volumes:
        # 挂载到数据卷
        - name: nfs-vol-v2
          persistentVolumeClaim:
            claimName: nfs-pvc-v2
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-v2
spec:
  ports:
    - port: 3306
      targetPort: 3306
  type: LoadBalancer
  selector:
    name: mysql-v2
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql-v3
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: mysql-v3
    spec:
      containers:
        - name: mysql-v3
          image: mysql:5.7
          # 只有镜像不存在时，才会进行镜像拉取
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
          # 同 Docker 配置中的 environment
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
          # 容器中的挂载目录
          volumeMounts:
            - name: nfs-vol-v3
              mountPath: /var/lib/mysql
      volumes:
        # 挂载到数据卷
        - name: nfs-vol-v3
          persistentVolumeClaim:
            claimName: nfs-pvc-v3
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-v3
spec:
  ports:
    - port: 3306
      targetPort: 3306
  type: LoadBalancer
  selector:
    name: mysql-v3
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql-v4
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: mysql-v4
    spec:
      containers:
        - name: mysql-v4
          image: mysql:5.7
          # 只有镜像不存在时，才会进行镜像拉取
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
          # 同 Docker 配置中的 environment
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
          # 容器中的挂载目录
          volumeMounts:
            - name: nfs-vol-v4
              mountPath: /var/lib/mysql
      volumes:
        # 挂载到数据卷
        - name: nfs-vol-v4
          persistentVolumeClaim:
            claimName: nfs-pvc-v4
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-v4
spec:
  ports:
    - port: 3306
      targetPort: 3306
  type: LoadBalancer
  selector:
    name: mysql-v4
    
    
    
 #创建
 #kubectl create -f mysql.yaml
 
 #查看
# kubectl get pod --all-namespaces -o wide 
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE     IP                NODE    NOMINATED NODE   READINESS GATES
default         dnsutils-ds-4zhs8                           1/1     Running   489        20d     192.168.73.69     k8s01   <none>           <none>
default         dnsutils-ds-b4xgn                           1/1     Running   489        20d     192.168.235.129   k8s03   <none>           <none>
default         dnsutils-ds-f6rg4                           1/1     Running   488        20d     192.168.236.142   k8s02   <none>           <none>
default         my-nginx-5dd67b97fb-5qx6b                   1/1     Running   8          37h     192.168.73.80     k8s01   <none>           <none>
default         my-nginx-5dd67b97fb-v8g9q                   1/1     Running   9          20d     192.168.73.81     k8s01   <none>           <none>
default         mysql-myshop-cf468c857-7qkf5                1/1     Running   0          5h1m    192.168.235.140   k8s03   <none>           <none>
default         mysql-v2-77bbd47d95-jznqr                   1/1     Running   0          143m    192.168.236.145   k8s02   <none>           <none>
default         mysql-v3-6766948f74-pdn4g                   1/1     Running   0          143m    192.168.236.146   k8s02   <none>           <none>
default         mysql-v4-6c8587689c-hjqlh                   1/1     Running   0          143m    192.168.236.144   k8s02   <none>           <none>
```



4.5 后记

```bash
#查看运行端口
#  kubectl get service
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
dnsutils-ds    NodePort       10.254.12.62     <none>        80:30732/TCP     20d
kubernetes     ClusterIP      10.254.0.1       <none>        443/TCP          25d
my-nginx       ClusterIP      10.254.29.238    <none>        80/TCP           20d
mysql-myshop   LoadBalancer   10.254.219.155   <pending>     3306:32374/TCP   5h47m
mysql-v2       LoadBalancer   10.254.175.77    <pending>     3306:30023/TCP   144m
mysql-v3       LoadBalancer   10.254.11.69     <pending>     3306:31834/TCP   144m
mysql-v4       LoadBalancer   10.254.208.213   <pending>     3306:32608/TCP   144m

#查看日志
# kubectl logs mysql-myshop-cf468c857-k7z7k


#执行命令
# kubectl --namespace=default exec mysql-myshop-cf468c857-k7z7k -- ls -al /var/lib/mysql
total 188484
drwxrwxrwx 5 mysql root       328 Nov 12 09:43 .
drwxr-xr-x 1 root  root        97 Oct 17 04:49 ..
-rw-r----- 1 mysql mysql       56 Nov 12 09:34 auto.cnf
-rw------- 1 mysql mysql     1680 Nov 12 09:34 ca-key.pem
-rw-r--r-- 1 mysql mysql     1112 Nov 12 09:34 ca.pem
-rw-r--r-- 1 mysql mysql     1112 Nov 12 09:34 client-cert.pem
-rw------- 1 mysql mysql     1676 Nov 12 09:34 client-key.pem
-rw-r----- 1 mysql mysql      692 Nov 12 09:43 ib_buffer_pool
-rw-r----- 1 mysql mysql 50331648 Nov 12 09:43 ib_logfile0
-rw-r----- 1 mysql mysql 50331648 Nov 12 09:34 ib_logfile1
-rw-r----- 1 mysql mysql 79691776 Nov 12 09:43 ibdata1
-rw-r----- 1 mysql mysql 12582912 Nov 12 09:44 ibtmp1
drwxr-x--- 2 mysql mysql     4096 Nov 12 09:34 mysql
drwxr-x--- 2 mysql mysql     8192 Nov 12 09:34 performance_schema
-rw------- 1 mysql mysql     1676 Nov 12 09:34 private_key.pem
-rw-r--r-- 1 mysql mysql      452 Nov 12 09:34 public_key.pem
-rw-r--r-- 1 mysql mysql     1112 Nov 12 09:34 server-cert.pem
-rw------- 1 mysql mysql     1676 Nov 12 09:34 server-key.pem
drwxr-x--- 2 mysql mysql     8192 Nov 12 09:34 sys

#登陆mysql
# kubectl exec -it mysql-myshop-cf468c857-k7z7k  sh

测试
#对mysql做加库加表操作

```



## 五、切换测试

5.1  对mysql pod所在主机进行下电操作



5.2  mysql pod切换到其他主机上

```bash
# kubectl get pod -o wide
NAME                               READY   STATUS        RESTARTS   AGE     IP                NODE    NOMINATED NODE   READINESS GATES
dnsutils-ds-4zhs8                  1/1     Running       485        20d     192.168.73.69     k8s01   <none>           <none>
dnsutils-ds-b4xgn                  1/1     Running       485        20d     192.168.235.129   k8s03   <none>           <none>
dnsutils-ds-f6rg4                  1/1     Running       484        20d     192.168.236.131   k8s02   <none>           <none>
my-nginx-5dd67b97fb-5qx6b          1/1     Running       8          33h     192.168.73.80     k8s01   <none>           <none>
my-nginx-5dd67b97fb-v8g9q          1/1     Running       9          20d     192.168.73.81     k8s01   <none>           <none>
mysql-myshop-cf468c857-7qkf5       1/1     Running       0          58m     192.168.235.140   k8s03   <none>           <none>
```

查询mysql数据还在

```bash
# kubectl -exec it mysql-myshop-cf468c857-7qkf5 sh
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| abc                |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

