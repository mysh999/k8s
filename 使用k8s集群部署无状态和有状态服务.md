参考：

https://kubernetes.io/zh/docs/tasks/run-application/run-stateless-application-deployment/

https://www.jianshu.com/p/dd47a3cde390







## 一、什么是无状态和有状态服务

1.1、无状态服务

​          1）该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应结果是一致的

​		  2）多个实例可以共享相同的持久化数据，例如：nginx实例、tomcat实例等

​          3）相关的K8S资源有：ReplicaSet、ReplicationController、Deployment等，由于是无状态服务，所以这些控制器创建的POD序号都是随机值，在缩容的时候不会明确缩容某一个POD，而是随机的



1.2、有状态服务

​           1）需要数据存储功能的服务，或者多线程类型的服务、队列等，比如mysql数据库、kafka、zookeeper等

​           2）每个实例都有自己独立的持久化存储，在K8S中通过申明模板来定义，持久卷申明模板在创建POD前创建，绑定到POD中

​		   3）数据卷上存储的数据重要，在Statefulset缩容时删除该声明是灾难性的。需要释放特定的持久卷时，需要手动删除对应的持久卷声明

​		   4）相关的K8S资源有：statefulSet、由于是有状态的服务，所以每个POD都有特定的名称和网络标识，比如pod名由statefulSet名+有序的数字组成（0、1、2...）

​		   5）在进行缩容的时候，明确知道会缩容哪个POD，从数字最大的开始。并且Statefulset在有实例不健康的情况下是不允许做缩容操作

​		   6）缩容任何时候只会操作一个POD实例，所以有状态应用的缩容不会很迅速







## 二、使用Deployment运行一个无状态应用

创建一个Kubernetes Deployment对象来运行一个应用,

创建一个多副本应用首选方法是使用Deployment,反过来使用ReplicaSet



2.1、准备部署yaml文件

```bash
#vi deployment.yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
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



2.2、创建一个Deployment

```bash
# kubectl apply -f deployment.yaml 
```



2.3、展示Deployment相关信息

```bash
# kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Tue, 17 Dec 2019 11:56:51 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec":{"replica...
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-54f57cf6bf (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  44s   deployment-controller  Scaled up replica set nginx-deployment-54f57cf6bf to 2
```



2.4、列出deployment创建的pods

```bash
# kubectl get pods -l app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-9nbrp   1/1     Running   0          119s
nginx-deployment-54f57cf6bf-dl5qn   1/1     Running   0          119s
```



2.5、展示某一个pod信息

```bash
# kubectl describe pod nginx-deployment-54f57cf6bf-9nbrp
Name:         nginx-deployment-54f57cf6bf-9nbrp
Namespace:    default
Priority:     0
Node:         k8s-node-02/192.168.230.104
Start Time:   Tue, 17 Dec 2019 11:56:53 +0800
Labels:       app=nginx
              pod-template-hash=54f57cf6bf
Annotations:  cni.projectcalico.org/podIP: 172.33.44.208/32
Status:       Running
IP:           172.33.44.208
IPs:
  IP:           172.33.44.208
Controlled By:  ReplicaSet/nginx-deployment-54f57cf6bf
Containers:
  nginx:
    Container ID:   docker://cdfcd197c75dfc8466df1f042efc232d31097d2db2ce3c8008175d884d7aa3fb
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 17 Dec 2019 11:56:56 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-fxsvz (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-fxsvz:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-fxsvz
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From                  Message
  ----    ------     ----   ----                  -------
  Normal  Scheduled  3m13s  default-scheduler     Successfully assigned default/nginx-deployment-54f57cf6bf-9nbrp to k8s-node-02
  Normal  Pulled     3m10s  kubelet, k8s-node-02  Container image "nginx:1.7.9" already present on machine
  Normal  Created    3m10s  kubelet, k8s-node-02  Created container nginx
  Normal  Started    3m10s  kubelet, k8s-node-02  Started container nginx
```



2.6、更新deployment

可以通过更新一个新的YAML文件来更新deployment. 下面的YAML文件指定该deployment镜像更新为nginx 1.8.

```bash
# kubectl get pods -l app=nginx                         
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-9nbrp   1/1     Running   0          108m
nginx-deployment-54f57cf6bf-dl5qn   1/1     Running   0          108m

# kubectl apply -f deployment.yaml 
deployment.apps/nginx-deployment configured

# kubectl get pods -l app=nginx    
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deployment-9f46bb5-b7w7t   1/1     Running   0          10s
nginx-deployment-9f46bb5-fl7qn   1/1     Running   0          6s
```

查看该deployment创建的pods以新的名称同时删除旧的pods



2.7、通过增加副本数来弹缩应用

将`replicas`设置为4, 指定该Deployment应有4个pods:

```bash
# kubectl get pods -l app=nginx
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deployment-9f46bb5-b7w7t   1/1     Running   0          3m53s
nginx-deployment-9f46bb5-fl7qn   1/1     Running   0          3m49s

# kubectl apply -f deployment.yaml 
deployment.apps/nginx-deployment configured

# kubectl get pods -l app=nginx
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deployment-9f46bb5-b7w7t   1/1     Running   0          4m15s
nginx-deployment-9f46bb5-cb6n8   1/1     Running   0          14s
nginx-deployment-9f46bb5-fl7qn   1/1     Running   0          4m11s
nginx-deployment-9f46bb5-wfqq6   1/1     Running   0          14s
```



2.8、删除deployment

通过名称删除deployment

```bash
# kubectl delete deployment nginx-deployment
```





## 三、运行一个单实例有状态应用

使用PersistentVolume和Deployment运行一个单实例有状态mysql应用



3.1、定义PV和PVC

```bash
# cat mysql-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

执行创建：

```bash
# kubectl apply -f mysql-pv.yaml 
```



创建效果：

![20191217_1506_001.jpg](http://ww1.sinaimg.cn/large/007Xg1efgy1g9zqharin4j311q0jjjxe.jpg)



3.2、部署mysql deployment

```bash
# cat mysql-deployment.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

执行：

```bash
# kubectl apply -f mysql-deployment.yaml 
```



查看效果：

```bash
# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
mysql-c85f7f79c-tlk67   1/1     Running   0          2m27s   172.33.44.211    k8s-node-02   <none>           <none>
```



且在node02节点可以看到有/mnt/data/目录产生

```bash
[root@k8s-node-02 data]# ls -lt /mnt/data/
total 110604
-rw-rw---- 1 polkitd input 50331648 Dec 17 15:10 ib_logfile0
-rw-rw---- 1 polkitd input 12582912 Dec 17 15:10 ibdata1
drwx------ 2 polkitd input     4096 Dec 17 15:10 mysql
-rw-rw---- 1 polkitd input       56 Dec 17 15:10 auto.cnf
drwx------ 2 polkitd input     4096 Dec 17 15:09 performance_schema
-rw-rw---- 1 polkitd input 50331648 Dec 17 15:09 ib_logfile1
```



展示deployment相关信息

```bash
# kubectl describe deployment mysql
Name:               mysql
Namespace:          default
CreationTimestamp:  Tue, 17 Dec 2019 15:08:02 +0800
Labels:             <none>
Annotations:        deployment.kubernetes.io/revision: 1
                    kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"mysql","namespace":"default"},"spec":{"selector":{"matchL...
Selector:           app=mysql
Replicas:           1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:       Recreate
MinReadySeconds:    0
Pod Template:
  Labels:  app=mysql
  Containers:
   mysql:
    Image:      mysql:5.6
    Port:       3306/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_ROOT_PASSWORD:  password
    Mounts:
      /var/lib/mysql from mysql-persistent-storage (rw)
  Volumes:
   mysql-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mysql-pv-claim
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   mysql-c85f7f79c (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  7m2s  deployment-controller  Scaled up replica set mysql-c85f7f79c to 1
```



列出deployment创建的pods

```bash
# kubectl get pods -l app=mysql
NAME                    READY   STATUS    RESTARTS   AGE
mysql-c85f7f79c-tlk67   1/1     Running   0          9m2s
```



查看持久卷

```bash
# kubectl describe pv mysql-pv
Name:            mysql-pv-volume
Labels:          type=local
Annotations:     kubectl.kubernetes.io/last-applied-configuration:
                   {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"labels":{"type":"local"},"name":"mysql-pv-volume"},"spec":{"acc...
                 pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual
Status:          Bound
Claim:           default/mysql-pv-claim
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        20Gi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /mnt/data
    HostPathType:  
Events:            <none>
```



查看PVC：

```bash
#  kubectl describe pvc mysql-pv-claim
Name:          mysql-pv-claim
Namespace:     default
StorageClass:  manual
Status:        Bound
Volume:        mysql-pv-volume
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"mysql-pv-claim","namespace":"default"},"spec":{"acc...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      20Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    mysql-c85f7f79c-tlk67
Events:        <none>
```



3.3、访问mysql实例

```bash
# kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
mysql-c85f7f79c-tlk67                    1/1     Running   0          58m   172.33.44.211    k8s-node-02   <none>           <none>
mysql-c85f7f79c-tlk67-658b4cbcbd-t9q5v   1/1     Running   2          40m   172.33.154.210   k8s-node-01   <none>           <none>

# kubectl exec -it mysql-c85f7f79c-tlk67 bash
root@mysql-c85f7f79c-tlk67:/# mysql -uroot -ppassword
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.6.46 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>  #创建新库和表

# ll /mnt/data/
total 110604
-rw-rw---- 1 polkitd input       56 Dec 17 15:10 auto.cnf
-rw-rw---- 1 polkitd input 12582912 Dec 17 16:08 ibdata1
-rw-rw---- 1 polkitd input 50331648 Dec 17 16:08 ib_logfile0
-rw-rw---- 1 polkitd input 50331648 Dec 17 15:09 ib_logfile1
drwx------ 2 polkitd input     4096 Dec 17 15:10 mysql
drwx------ 2 polkitd input     4096 Dec 17 15:09 performance_schema
drwx------ 2 polkitd input       50 Dec 17 16:07 qa   --新库
```



3.4、模拟节点故障

将node02节点关机

会在node01上新建一个mysql pod，但是数据是永久化本地，所以是一个新环境

```bash
# kubectl get pods -o wide
NAME                                     READY   STATUS        RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
mysql-c85f7f79c-r784b                    1/1     Running       0          5m26s   172.33.154.211   k8s-node-01   <none>           <none>
mysql-c85f7f79c-tlk67                    1/1     Terminating   0          74m     172.33.44.211    k8s-node-02   <none>           <none>
mysql-c85f7f79c-tlk67-658b4cbcbd-t9q5v   1/1     Running       2          56m     172.33.154.210   k8s-node-01   <none>           <none>
```



```bash
[root@k8s-node-01 data]# ll /mnt/data/
total 110604
-rw-rw---- 1 polkitd input       56 Dec 17 16:17 auto.cnf
-rw-rw---- 1 polkitd input 12582912 Dec 17 16:17 ibdata1
-rw-rw---- 1 polkitd input 50331648 Dec 17 16:17 ib_logfile0
-rw-rw---- 1 polkitd input 50331648 Dec 17 16:16 ib_logfile1
drwx------ 2 polkitd input     4096 Dec 17 16:17 mysql
drwx------ 2 polkitd input     4096 Dec 17 16:17 performance_schema
```

![20191217_1624_001.jpg](http://ww1.sinaimg.cn/large/007Xg1efgy1g9zsqdtpq6j311w0jhtfs.jpg)