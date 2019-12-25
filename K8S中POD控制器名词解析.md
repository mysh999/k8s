## 一、POD控制器作用

kubelet是节点代理程序，在每个节点运行一个实例，当节点发生故障时，kubelet也将不可用，节点上的pod资源健康将无从保证。此场景的pod存活性由节点外的pod控制器保证，遭到意外删除的pod资源恢复也依赖pod控制器



pod控制器由master的kube-control-manager组件提供，常见控制器种类有Replicatiion Controller、ReplicaSet、Deployment、StatefulSet、Job、Cronjob





## 二、特性说明 

| 控制器名称              | 特性                                                         | 使用场景                                                     |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Replicatiion Controller | 确保每个POD副本在任一时刻均能满足目标数量                    | 属于上一代无状态POD应用控制器，建议使用新型控制器Deployment和ReplicaSet取代它 |
| ReplicaSet              | 与Replicatiion Controller唯一不同点在于支持的标签选择器不同，RC只支持等值选择器（env=dev或environment!=qa），RS额外支持基于集合的选择器 | 下一代复本控制器 (为无状态服务而设计)                        |
| Deployment              | 用于为POD和RS提供声明式更新，是建构在RS之上更高级的控制器    | 用于管理无状态的持久化应用，比如HTTP   (为无状态服务而设计)                                              典型的应用场景包括：定义Deployment来创建Pod和ReplicaSet、滚动升级和回滚应用、扩容和缩容、暂停和继续Deployment |
| StatefulSet             | 与Deployment不同点是为每个POD创建一个独有的持久性标识符，且确保各POD之间的顺序性 | 用于管理有状态的持久化应用，如DB                             |
| DemonSet                | 确保每个节点运行某POD的一个副本                              | 常用于运行集群存储守护进程，如glusterd和ceph，还有日志收集进程如fluentd和logstash，及监控进程，如prometheus的node exporter、collectd等 |
| Job                     | 管理运行完成后即可终止的应用                                 | 批处理作业任务                                               |



## 三、RC使用案例

3.1、使用RC配置3个nginx web服务副本 

```bash
# cat rc_nginx.yaml 
apiVersion: v1                       #必需
kind: ReplicationController          #必需
metadata:                            #必需
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx-rc
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```



查看状态

```bash
# kubectl describe replicationcontrollers/nginx-rc
Name:         nginx-rc
Namespace:    default
Selector:     app=nginx
Labels:       app=nginx
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  34s   replication-controller  Created pod: nginx-rc-cbhqx
  Normal  SuccessfulCreate  34s   replication-controller  Created pod: nginx-rc-5djc8
  Normal  SuccessfulCreate  34s   replication-controller  Created pod: nginx-rc-jnkpv
```



查看日志

```bash
# kubectl logs nginx-rc-5djc8  -n default -f  
```





## 四、RS使用案例

一个ReplicaSet保证pod副本为一个指定的数目在给定的任何时间内.然而,一个Deployment是一个更高级别的概念来去管理ReplicaSets 和提供描述性的pods升级以及很多其他有用的特性.因此,我们推荐使用Deployments来替代直接使用ReplicaSets



4.1、配置部署

```bash
# cat rs_frontend.yaml 
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80

```



4.2、查看状态

```bash
# kubectl describe rs/frontend
Name:         frontend
Namespace:    default
Selector:     tier=frontend,tier in (frontend)
Labels:       app=guestbook
              tier=frontend
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=guestbook
           tier=frontend
  Containers:
   php-redis:
    Image:      gcr.io/google_samples/gb-frontend:v3
    Port:       80/TCP
    Host Port:  0/TCP
    Requests:
      cpu:     100m
      memory:  100Mi
    Environment:
      GET_HOSTS_FROM:  dns
    Mounts:            <none>
  Volumes:             <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  6m21s  replicaset-controller  Created pod: frontend-5k7r2
  Normal  SuccessfulCreate  6m21s  replicaset-controller  Created pod: frontend-jdhdx
  Normal  SuccessfulCreate  6m21s  replicaset-controller  Created pod: frontend-4ld4w
```



## 五、deployment使用案例

Deployment为Pod和ReplicaSet提供了一个声明式定义(declarative)方法，用来替代以前的[ReplicationController](https://www.kubernetes.org.cn/replication-controller-kubernetes)来方便的管理应用



5.1、配置nginx应用

```bash
# cat deployment_nginx.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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



5.2、查看状态

```bash
# kubectl describe deployments nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Wed, 25 Dec 2019 11:37:27 +0800
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec...
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
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
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-5754944d6c (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  3m26s  deployment-controller  Scaled up replica set nginx-deployment-5754944d6c to 3
  
  
  
  # kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
mysql              1/1     1            1           6d18h
nginx              2/2     2            2           20d
nginx-deployment   10/10   10           10          13m
tomcat-app         2/2     2            2           6d16h
```



5.3、扩容

```bash
# kubectl scale deployment nginx-deployment --replicas 10
deployment.extensions/nginx-deployment scaled

#如果集群支持horizontal pod autoscaling，可以设置自动扩展
# kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
```



5.4、更新镜像

```bash
# kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
```



5.5、回滚

```bash
# kubectl rollout undo deployment/nginx-deployment
```



## 六、Statefulset部署

StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计）

这里以部署mysql为例



6.1、部署nfs服务器

NFS服务器准备

单独准备一台NFS服务器共享

```bash
#安装包
#yum -y install rpcbind nfs-utils

# vi /etc/exports

/sharenfs *(rw,sync,no_subtree_check,no_root_squash)

# systemctl restart nfs     
# systemctl enable nfs

# systemctl restart rpcbind.service
# systemctl enable rpcbind.service
```



6.2、每台客户端安装和挂载NFS

```bash
# yum -y install nfs-utils

# mkdir /nfsclient
# mount -t nfs 192.168.230.200:/sharenfs /nfsclient
```



6.3、配置PV和PVC

```bash
# cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-mysql
  labels:
    pv: pv-nfs-mysql
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: "/sharenfs"
    server: 192.168.230.200
    readOnly: false
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-mysql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      pv: pv-nfs-mysql
      
      
      
```



查看PV和PVC

```bash
# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pv-nfs-mysql   10Gi       RWX            Retain           Bound    default/pvc-nfs-mysql                           55s


# kubectl get pvc
NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs-mysql   Bound    pv-nfs-mysql   10Gi       RWX                           57s
```



6.4、创建secret对象存储密码

密码使用base64加密

```bash
# echo -n 'password' | base64
cGFzc3dvcmQ=
```



```bash
# cat secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: k8s-dev-mysql
type: Opaque
data:
  mysql-root-password: cGFzc3dvcmQ=

```





查看

```bash
# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-vd4b6   kubernetes.io/service-account-token   3      21d
k8s-dev-mysql         Opaque                                1      8s
```





6.5、创建stateful状态pod 

```bash
# cat mysql.yaml 
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        args: ["--character_set_server=utf8","--collation-server=utf8_unicode_ci"]
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: mysql-root-password
              name: k8s-dev-mysql
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: pvc-nfs-mysql 
```



查看：

```bash
# kubectl get pods
NAME                                READY   STATUS      RESTARTS   AGE
frontend-5k7r2                      1/1     Running     0          7h51m
frontend-kvq6j                      1/1     Running     0          75m
frontend-rmgtf                      1/1     Running     0          75m
mysql-0                             1/1     Running     0          6m50s
```



登陆mysql

```bash
# kubectl exec -it mysql-0 bash
root@mysql-0:/# mysql -uroot -ppassword
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.46 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```



6.6、创建svc

```bash
# cat mysql_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: k8s-dev-mysql
  labels:
    app: k8s-dev-mysql
spec:
  type: NodePort
  ports:
    - port: 3306
  selector:
    app: k8s-dev-mysql
```



查看

```bash
# kubectl get svc                  
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
k8s-dev-mysql   NodePort    10.109.40.115    <none>        3306:31962/TCP   4s
```





## 七、DaemonSet使用

DaemonSet保证在每个Node上都运行一个容器副本，常用来部署一些集群的日志、监控或者其他系统管理应用。典型的应用包括：

日志收集，比如fluentd，logstash等
系统监控，比如Prometheus Node Exporter，collectd，New Relic agent，Ganglia gmond等
系统程序，比如kube-proxy, kube-dns, glusterd, ceph等



7.1、使用Fluentd收集日志

```bash
# cat daemonset_fluentd.yaml 
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    metadata:
      labels:
        app: logging
        id: fluentd
      name: fluentd
    spec:
      containers:
      - name: fluentd-es
        image: gcr.io/google_containers/fluentd-elasticsearch:1.3
        env:
         - name: FLUENTD_ARGS
           value: -qq
        volumeMounts:
         - name: containers
           mountPath: /var/lib/docker/containers
         - name: varlog
           mountPath: /varlog
      volumes:
         - hostPath:
             path: /var/lib/docker/containers
           name: containers
         - hostPath:
             path: /var/log
           name: varlog
```



7.2、查看效果

每个node均有分布

```bash
# kubectl get pods -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
fluentd-5szjt                       0/1     ContainerCreating   0          3m13s   <none>         k8s-node1   <none>           <none>
fluentd-9m785                       0/1     ContainerCreating   0          3m13s   <none>         k8s-node2   <none>           <none>
```





## 八、jobs使用案例

8.1、使用

```bash
# cat job.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```



8.2、查看日志

```bash
# kubectl logs pi-llzrz
```

