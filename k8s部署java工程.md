## 一、架构说明

1、一套K8S集群

2、后端存储使用ceph，K8S使用动态PV

3、完整部署一套java spring boot'工程，包含mysql、redis、java工程等



```bash
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.2.22 kuber-22 demo.auth.cn
10.10.2.30 noder-30

# kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
kuber-22   Ready    <none>   195d   v1.17.6
noder-30   Ready    <none>   195d   v1.17.6
```





## 二、ceph存储部署

2.1、创建ceph集群过程略

2.2、创建pool

```bash
# ceph osd pool create pool_k8s 32
```



## 三、K8S集群基础配置

### 3.1、配置yum

```
[root@k8s-master yum.repos.d]# cat ceph.repo 
[ceph]

name=ceph

baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/x86_64/

gpgckeck=0

gpgkey=http://mirrors.163.com/ceph/keys/release.asc

[ceph-noarch]

name=Ceph noarch packages

baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/noarch/

gpgcheck=0

gpgkey=http://mirrors.163.com/ceph/keys/release.asc


[root@k8s-master yum.repos.d]# yum makecache
```



### 3.2、安装ceph-common

```
[root@k8s-master yum.repos.d]#  yum -y install ceph-common
```



### 3.3、从ceph节点拷贝文件到k8s master下

```
[root@k8s-master yum.repos.d]# scp 10.10.2.27:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
[root@k8s-master yum.repos.d]# scp 10.10.2.27:/etc/ceph/ceph.conf /etc/ceph/       
```



### 3.4、从k8s master上检查是否能看到ceph pool信息

```
[root@k8s-master yum.repos.d]# # rados lspools
device_health_metrics
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
pool_data01
pool_data01_metadata
kubernetes_default
pool_k8s
```



### 3.5、获取client.admin的keyring值，并用base64编码

```
[root@k8s-master ceph]#  ceph auth get-key client.admin | base64  
QVFCTE96TmZHejh6TnhBQXRpeUd3UFA4cExrN2U5MWFyZnFiOFE9PQ==
```



### 3.6、创建命名空间

```bash
# kubectl get ns     #查询namespace

# kubectl create ns ns-mysql  #创建namespace
```





### 3.7、配置ceph secret

```
# cat ceph-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: ns-mysql
type: "kubernetes.io/rbd"
data:
  key: QVFCTE96TmZHejh6TnhBQXRpeUd3UFA4cExrN2U5MWFyZnFiOFE9PQ== 



[root@k8s-master K8s-Ceph]# kubectl create -f ceph-secret.yaml 

# kubectl -n ns-mysql get secrets -o wide
NAME                  TYPE                                  DATA   AGE
ceph-secret           kubernetes.io/rbd                     1      33s
default-token-5s72d   kubernetes.io/service-account-token   3      28m
```





### 3.8、配置storage class

```bash
# cat ceph-class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: ceph-class
   namespace: ns-mysql
   annotations:
     storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.10.2.27:6789,10.10.2.28:6789,10.10.2.29:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: ns-mysql
  pool: pool_k8s
  userId: admin
  userSecretName: ceph-secret
  userSecretNamespace: ns-mysql
  fsType: xfs
  imageFormat: "2"
  imageFeatures: "layering"
  
  
  
# kubectl -n ns-mysql get sc
NAME                   PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ceph-class (default)   kubernetes.io/rbd   Delete          Immediate           false                  39s
```



### 3.9、根据sc动态创建PVC

```bash
# cat ceph-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-ceph-pvc
  namespace: ns-mysql
spec:
  storageClassName: ceph-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
      
      

# kubectl -n ns-mysql get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
persistentvolume/pvc-c1a880d8-35e9-402d-a5fd-72a4dcd38e67   10Gi       RWO            Delete           Bound    default/mysql-ceph-pvc   ceph-class              9h

NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-ceph-pvc   Bound    pvc-b383c289-aec1-4fe6-b4b2-ba27accd1828   10Gi       RWO            ceph-class     15s
```







## 四、配置mysql资源



### 4.1、创建mysql的 secret

```bash
#创建secret对象存储密码，密码使用base64 加密
# echo -n 'password' | base64


# cat mysql-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: k8s-dev-mysql
  namespace: ns-mysql
type: Opaque
data:
  mysql-root-password: cGFzc3dvcmQ=
  
  
  
  
# kubectl -n ns-mysql get secrets
NAME                  TYPE                                  DATA   AGE
ceph-secret           kubernetes.io/rbd                     1      41m
default-token-5s72d   kubernetes.io/service-account-token   3      69m
k8s-dev-mysql         Opaque                                1      42s
```



### 4.2、创建mysql pod

```bash
# cat mysql_pod.yaml 
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mysql-app
  namespace: ns-mysql
spec:
  selector:
    matchLabels:
      app: mysql-app
  serviceName: mysql-svc
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql-app
    spec:
      containers:
      - name: mysql
        image: harbor.ubtrobot.com/database/mysql:5.6.41
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
          claimName: mysql-ceph-pvc
          
          
# kubectl -n ns-mysql get pod
NAME          READY   STATUS    RESTARTS   AGE
mysql-app-0   1/1     Running   0          42s
```



### 4.3、创建mysql svc

```bash
# cat mysql_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  namespace: ns-mysql
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app:  mysql-app
    
    
# kubectl -n ns-mysql get svc
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql-svc   ClusterIP   10.254.6.164   <none>        3306/TCP   11s
```



### 4.4、登录mysql命令行

```bash
# kubectl -n ns-mysql get pods
NAME          READY   STATUS    RESTARTS   AGE
mysql-app-0   1/1     Running   0          2m39s

# kubectl -n ns-mysql exec -it mysql-app-0 -- mysql -uroot -ppassword -hmysql-svc

mysql> 


# 执行mysql数据初始化sql
```





## 五、配置redis资源

说明：

redis是一个有状态应用，当把redis以pod的形式部署在k8s中时，每个pod里缓存的数据都是不一样的，而且pod的IP是会随时变化，如果使用普通的deployment和service来部署redis-cluster就会出现很多问题，因此需要改用StatefulSet + Headless Service来解决



Headless Service：简单的说，Headless Service就是没有指定Cluster IP的Service，相应的，在k8s的dns映射里，Headless Service的解析结果不是一个Cluster IP，而是它所关联的所有Pod的IP列表



StatefulSet：是k8s中专门用于解决有状态应用部署的一种资源，总的来说可以认为它是Deployment/RC的一个变种，它有以下几个特性：

StatefulSet管理的每个Pod都有唯一的文档/网络标识，并且按照数字规律生成，而不是像Deployment中那样名称和IP都是随机的（比如StatefulSet名字为redis，那么pod名就是redis-0, redis-1 ...）
StatefulSet中ReplicaSet的启停顺序是严格受控的，操作第N个pod一定要等前N-1个执行完才可以
StatefulSet中的Pod采用稳定的持久化储存，并且对应的PV不会随着Pod的删除而被销毁
另外需要说明的是，StatefulSet必须要配合Headless Service使用，它会在Headless Service提供的DNS映射上再加一层，最终形成精确到每个pod的域名映射，格式如下：
$(podname).$(headless service name)



流程如下：

创建PVC（此处放在创建POD里使用动态PVC创建）
创建ConfigMap
创建Headless Service
创建StatefulSet POD
初始化redis集群



### 5.1、创建ConfigMap

```bash
# cat redis-configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: ns-mysql
data:
  configdata: |
    bind  0.0.0.0
    protected-mode no
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    daemonize no
    supervised no
    pidfile /tmp/redis.pid
    loglevel notice
    logfile /tmp/redis.log
    databases 16
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /var/lib/redis
    slave-serve-stale-data yes
    slave-read-only yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-disable-tcp-nodelay no
    slave-priority 100
    appendonly no
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-load-truncated yes
    lua-time-limit 5000
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    latency-monitor-threshold 0
    notify-keyspace-events Ex
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-size -2
    list-compress-depth 0
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    hll-sparse-max-bytes 3000
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    hz 10
    aof-rewrite-incremental-fsync yes

    rename-command FLUSHALL ""
    rename-command FLUSHDB ""
    rename-command CONFIG ""
    rename-command EVAL ""
    
    
# kubectl -n ns-mysql get configmap
NAME           DATA   AGE
redis-config   1      89s 
```



### 5.2、创建pod

```bash
# cat redis-pod.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-app
  namespace: ns-mysql
spec:
  serviceName: "redis-svc"
  replicas: 2
  selector:
    matchLabels:
      app: redis
      appCluster: redis-cluster
  template:
    metadata:
      labels: 
        app: redis
        appCluster: redis-cluster
    spec:
      containers:
      - name: redis
        image: harbor.ubtrobot.com/database/redis:5.0.7
        imagePullPolicy: IfNotPresent
        args:
          - redis-server
          - /redis/redis.config
        ports:
        - containerPort: 6379
          protocol: TCP
        resources:
          limits:
            cpu: 0.25
            memory: 0.5Gi
        volumeMounts:
        - name: redis-data
          mountPath: /var/lib/redis
        - name: config-volumes
          mountPath: /redis/redis.config
          subPath: configdata
      volumes:
        - name: config-volumes
          configMap:
            name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
      namespace: ns-mysql
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ceph-class
      resources:
        requests:
          storage: 2Gi
          
  # kubectl -n ns-mysql get pods
NAME          READY   STATUS    RESTARTS   AGE
mysql-app-0   1/1     Running   0          15d
redis-app-0   1/1     Running   0          52s
redis-app-1   1/1     Running   0          42s


# 修改pod里的redis data属主 
# kubectl -n ns-mysql exec -it redis-app-1 /bin/bash
# chown -R redis:redis /var/lib/redis/
# 插入测试数据
# for i in {2..22};do     redis-cli set $i $i; done  

# kubectl -n ns-mysql get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                             STORAGECLASS   REASON   AGE
persistentvolume/mysql-init-pv                              10Gi       RWX            Retain           Available                                                             20d
persistentvolume/pv-dev                                     50Gi       RWO            Recycle          Bound       default/pvc-dev                   rbd                     83d
persistentvolume/pv-ops                                     100Gi      RWO            Recycle          Bound       default/pvc-ops                   rbd                     83d
persistentvolume/pvc-36962df1-5e62-4190-80aa-6b3633b0faf0   2Gi        RWO            Delete           Bound       ns-mysql/redis-data-redis-app-1   ceph-class              14m
persistentvolume/pvc-968a011e-f791-455f-b447-20a59cf3f5cc   1500Mi     RWO            Delete           Bound       default/datadir-zk-0              ceph-class              23h
persistentvolume/pvc-b364f3b8-302a-4f5a-a70a-b29f2d615359   1500Mi     RWO            Delete           Bound       default/datadir-zk-1              ceph-class              23h
persistentvolume/pvc-b383c289-aec1-4fe6-b4b2-ba27accd1828   10Gi       RWO            Delete           Bound       ns-mysql/mysql-ceph-pvc           ceph-class              15d
persistentvolume/pvc-c1a880d8-35e9-402d-a5fd-72a4dcd38e67   10Gi       RWO            Delete           Failed      default/mysql-ceph-pvc            ceph-class              22d
persistentvolume/pvc-ff391a93-b335-408a-b577-ce5d34aecf0f   2Gi        RWO            Delete           Bound       ns-mysql/redis-data-redis-app-0   ceph-class              15m

NAME                                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-ceph-pvc           Bound    pvc-b383c289-aec1-4fe6-b4b2-ba27accd1828   10Gi       RWO            ceph-class     15d
persistentvolumeclaim/redis-data-redis-app-0   Bound    pvc-ff391a93-b335-408a-b577-ce5d34aecf0f   2Gi        RWO            ceph-class     15m
persistentvolumeclaim/redis-data-redis-app-1   Bound    pvc-36962df1-5e62-4190-80aa-6b3633b0faf0   2Gi        RWO            ceph-class     14m




```







### 5.3、创建headless service（可选）

  Headless service是StatefulSet实现稳定网络标识的基础，需要提前创建，使用在有状态服务中，后台连接集群，如果是单机架构可以不用部署。此处其实是单机架构，为了加深记录，因此也做了headless

```bash
# cat redis-headless-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: redis-headless-service
  namespace: ns-mysql
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster
    
 
 
# kubectl -n ns-mysql get svc
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql-svc                ClusterIP   10.254.6.164   <none>        3306/TCP   15d
redis-headless-service   ClusterIP   None           <none>        6379/TCP   15s
```



### 5.4、创建redis svc

```bash
# cat redis-service.yaml 
kind: Service
apiVersion: v1
metadata:
  name: redis-service
  namespace: ns-mysql
  labels:
    app: redis
spec:
  selector:
    app: redis
  clusterIP: None
  ports:
    - port: 6379
      targetPort: 6379
      
      
# cat redis-client-svc.yaml 
kind: Service
apiVersion: v1
metadata:
  name: redis-client
  namespace: ns-mysql
  labels:
    app: redis
spec:
  selector:
    app: redis
  type: NodePort
  ports:
    - port: 6379
      targetPort: 6379
      
  
  
  # kubectl -n ns-mysql get svc
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql-svc                ClusterIP   10.254.6.164     <none>        3306/TCP         15d
redis-client             NodePort    10.254.107.185   <none>        6379:30272/TCP   8s
redis-headless-service   ClusterIP   None             <none>        6379/TCP         10m
redis-service            ClusterIP   None             <none>        6379/TCP         3m6s


[root@kuber-22 redis]# kubectl  -n ns-mysql describe svc redis-service
Name:              redis-service
Namespace:         ns-mysql
Labels:            app=redis
Annotations:       <none>
Selector:          app=redis
Type:              ClusterIP
IP:                None
Port:              <unset>  6379/TCP
TargetPort:        6379/TCP
Endpoints:         172.31.123.134:6379,172.31.242.194:6379
Session Affinity:  None
Events:            <none>
[root@kuber-22 redis]# kubectl  -n ns-mysql describe svc redis-client
Name:                     redis-client
Namespace:                ns-mysql
Labels:                   app=redis
Annotations:              <none>
Selector:                 app=redis
Type:                     NodePort
IP:                       10.254.107.185
Port:                     <unset>  6379/TCP
TargetPort:               6379/TCP
NodePort:                 <unset>  30272/TCP
Endpoints:                172.31.123.134:6379,172.31.242.194:6379
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
[root@kuber-22 redis]# kubectl  -n ns-mysql describe svc redis-headless-service
Name:              redis-headless-service
Namespace:         ns-mysql
Labels:            app=redis
Annotations:       <none>
Selector:          app=redis,appCluster=redis-cluster
Type:              ClusterIP
IP:                None
Port:              redis-port  6379/TCP
TargetPort:        6379/TCP
Endpoints:         172.31.123.134:6379,172.31.242.194:6379
Session Affinity:  None
Events:            <none>
```







## 六、配置zookeeper资源

### 6.1、创建 zk pod，关联动态PVC

```bash
# cat zk-pod-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  namespace: ns-mysql
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  namespace: ns-mysql
  labels:
    app: zk
spec:
  type: NodePort
  ports:
  - port: 2181
    targetPort: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
  namespace: ns-mysql
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
  namespace: ns-mysql
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 2
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "leolee32/kubernetes-library:kubernetes-zookeeper1.0-3.4.10"
        resources:
          requests:
            memory: "0.25Gi"
            cpu: "0.25"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=2 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
  volumeClaimTemplates:
  - metadata:
      name: datadir
      namespace: ns-mysql
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1500Mi
      storageClassName: ceph-class
      
      
      
      
#  kubectl -n ns-mysql get  pods
NAME          READY   STATUS    RESTARTS   AGE
mysql-app-0   1/1     Running   0          15d
redis-app-0   1/1     Running   0          168m
redis-app-1   1/1     Running   0          168m
zk-0          1/1     Running   0          4m55s
zk-1          1/1     Running   0          4m55s


# kubectl -n ns-mysql get PodDisruptionBudget
NAME     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
zk-pdb   N/A             1                 1                     7m34s


# kubectl -n ns-mysql get svc
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
mysql-svc                ClusterIP   10.254.6.164     <none>        3306/TCP            15d
redis-client             NodePort    10.254.107.185   <none>        6379:30272/TCP      159m
redis-headless-service   ClusterIP   None             <none>        6379/TCP            169m
redis-service            ClusterIP   None             <none>        6379/TCP            162m
zk-cs                    NodePort    10.254.83.230    <none>        2181:32364/TCP      25s
zk-hs                    ClusterIP   None             <none>        2888/TCP,3888/TCP   25s

```



### 6.2、测试连接

```bash
# 查看日志 
# kubectl -n ns-mysql logs -f zk-0


# 验证集群是否正常
# for i in 0 1 ; do kubectl -n ns-mysql exec zk-$i zkServer.sh status; done    
ZooKeeper JMX enabled by default
Using config: /usr/bin/../etc/zookeeper/zoo.cfg
Mode: follower
ZooKeeper JMX enabled by default
Using config: /usr/bin/../etc/zookeeper/zoo.cfg
Mode: leader

# 一个leader，一个follower，说明已将zk集群在k8s中创建成功



# 查看主机名
# for i in 0 1 ; do kubectl -n ns-mysql exec zk-$i -- hostname; done                  
zk-0
zk-1


# 查看 myid
# for i in 0 1; do echo "myid zk-$i";kubectl -n ns-mysql  exec zk-$i -- cat /var/lib/zookeeper/data/myid; done   
myid zk-0
1
myid zk-1
2


# 查看完整域名
# for i in 0 1 ; do kubectl -n ns-mysql exec zk-$i -- hostname -f; done                                          
zk-0.zk-hs.ns-mysql.svc.cluster.local
zk-1.zk-hs.ns-mysql.svc.cluster.local



# 查看zk配置
# for i in 0 1 ; do kubectl -n ns-mysql exec zk-$i -- cat /opt/zookeeper/conf/zoo.cfg; done              
#This file was autogenerated DO NOT EDIT
clientPort=2181
dataDir=/var/lib/zookeeper/data
dataLogDir=/var/lib/zookeeper/data/log
tickTime=2000
initLimit=10
syncLimit=5
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
autopurge.snapRetainCount=3
autopurge.purgeInteval=12
server.1=zk-0.zk-hs.ns-mysql.svc.cluster.local:2888:3888
server.2=zk-1.zk-hs.ns-mysql.svc.cluster.local:2888:3888
#This file was autogenerated DO NOT EDIT
clientPort=2181
dataDir=/var/lib/zookeeper/data
dataLogDir=/var/lib/zookeeper/data/log
tickTime=2000
initLimit=10
syncLimit=5
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
autopurge.snapRetainCount=3
autopurge.purgeInteval=12
server.1=zk-0.zk-hs.ns-mysql.svc.cluster.local:2888:3888
server.2=zk-1.zk-hs.ns-mysql.svc.cluster.local:2888:3888
```



### 6.3、测试zookeeper集群整体性

```bash
# 登陆节点1记录数据
# kubectl -n ns-mysql exec -it zk-0 bash
root@zk-0:/# cd /opt/zookeeper/bin
root@zk-0:/opt/zookeeper/bin# zkCli.sh 
Connecting to localhost:2181
2020-11-11 12:50:38,635 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
...   ...

[zk: localhost:2181(CONNECTED) 0] create /hello world
Created /hello
[zk: localhost:2181(CONNECTED) 1] get /hello
world
cZxid = 0x100000002
ctime = Wed Nov 11 12:50:50 UTC 2020
mZxid = 0x100000002
mtime = Wed Nov 11 12:50:50 UTC 2020
pZxid = 0x100000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0


# 在其他节点查看数据是否同步
# kubectl -n ns-mysql exec -it zk-1 bash
root@zk-1:/# #
root@zk-1:/# cd /opt/zookeeper/bin/
root@zk-1:/opt/zookeeper/bin# zkCli.sh
Connecting to localhost:2181
... ...

WATCHER::

WatchedEvent state:SyncConnected type:None path:null

[zk: localhost:2181(CONNECTED) 0] get /hello
world
cZxid = 0x100000002
ctime = Wed Nov 11 12:50:50 UTC 2020
mZxid = 0x100000002
mtime = Wed Nov 11 12:50:50 UTC 2020
pZxid = 0x100000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```









## 七、java spring boot工程改造



### 7.1、修改对应spring boot jar包配置

```bash
# vi application-local.yaml
## AOP logger configuration configure the log info   async : true   sysn : false or default
configure:
  log:
    async: false

spring:
  swagger:
    open: false
  ## REDIS-MASTER
  jedis:
    password: 
    timeout: 600000
    pool:
      configstr: jedis-center1:redis-service:6379
      max-active: 10
      max-idle: 20
      min-idle: 10
      max-wait: 100000
      max-total: 50
  ## REDIS-SLAVE
  jedis-two:
    open: false
    password: 
    timeout: 600000
    pool:
      configstr: jedis-center1:redis-service:6379
      max-active: 10
      max-idle: 20
      min-idle: 10
      max-wait: 100000
      max-total: 50
      
      
      
  # vi dubbo.properties
  ##Dubbo Configuration
dubbo.application-name=user-common-authorization
dubbo.scan-package= com.ubtechinc.service

dubbo.protocol-name=dubbo
dubbo.protocol-accessLog=true
dubbo.protocol-port=20882
dubbo.provider-timeout=3000
dubbo.provider-retries=1
dubbo.provider-delay=-1

dubbo.registry-protocol=zookeeper
dubbo.registry-address=zookeeper://zk-cs:2181



# vi jedis.properties
##Redis Configuration
jedis.pool.configstr=jedis-center1:redis-service:6379
jedis.pool.password=

jedis.pool.timeout=3000
jedis.pool.max-total=300
jedis.pool.max-idle=200
jedis.pool.max-wait-millis=1000


# vi druid.properties
# Druid jdbc Connection Configuration
druid.url=jdbc:mysql://mysql-svc:3306/ubxdb?useUnicode=true&characterEncoding=UTF-8
druid.username=root
druid.password=password
druid.driver-class-name=com.mysql.jdbc.Driver
druid.initial-size=10
druid.min-idle=10
druid.max-active=25
druid.test-while-idle=true
```



### 7.2、制作docker image

```bash
# cat Dockerfile 
FROM openjdk:8-jre-alpine
WORKDIR /tmp
COPY target/ubtechinc-authorization-server-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]

# docker build -t authorization:latest .
```



### 7.3、创建pod

```bash
# cat spring.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment-authorization
  namespace: ns-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-authorization
  template:
    metadata:
      labels:
        app: spring-boot-authorization
    spec:
      containers:
        - name: spring-boot-authorization-pod
          image: authorization:latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
  
  
              
  # kubectl -n ns-mysql get pod   
NAME                                               READY   STATUS    RESTARTS   AGE
mysql-app-0                                        1/1     Running   0          15d
redis-app-0                                        1/1     Running   0          17h
redis-app-1                                        1/1     Running   0          17h
spring-deployment-authorization-568fb4c6cb-hfttc   1/1     Running   0          37s
zk-0                                               1/1     Running   0          14h
zk-1                                               1/1     Running   0          14h


# kubectl -n ns-mysql logs -f spring-deployment-authorization-568fb4c6cb-hfttc
```



### 7.4、创建svc

```bash
# cat svc.yaml 
apiVersion: v1
kind: Service
metadata:
 labels:
 name: springboot-service
 namespace: ns-mysql
spec:
 type: NodePort
 selector:
   app: spring-boot-authorization
 ports:
   - port: 8080
     targetPort: 8080
     nodePort: 32200
[root@kuber-22 docker]# kubectl -n ns-mysql get svc
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
mysql-svc                ClusterIP   10.254.6.164     <none>        3306/TCP            69d
redis-client             NodePort    10.254.107.185   <none>        6379:30272/TCP      54d
redis-headless-service   ClusterIP   None             <none>        6379/TCP            54d
redis-service            ClusterIP   None             <none>        6379/TCP            54d
springboot-service       NodePort    10.254.55.38     <none>        8080:32200/TCP      4m40s
zk-cs                    NodePort    10.254.2.128     <none>        2181:30676/TCP      54d
zk-hs                    ClusterIP   None             <none>        2888/TCP,3888/TCP   54d


# kubectl -n ns-mysql describe svc springboot-service
Name:                     springboot-service
Namespace:                ns-mysql
Labels:                   <none>
Annotations:              <none>
Selector:                 app=spring-boot-authorization
Type:                     NodePort
IP:                       10.254.55.38
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32200/TCP
Endpoints:                172.31.242.197:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```



### 7.5、测试

```bash
# kubectl exec -it centos /bin/bash
# 可以打开swagger界面
[root@centos /]# curl http://springboot-service.ns-mysql:8080/v1/client-auth-service/swagger-ui.html


#使用endpoints ip也可以访问
[root@centos /]# curl http://172.31.123.133:8080/v1/client-auth-service/swagger-ui.html

# 在K8S主机上也能通过nodeport ip访问
[root@kuber-22 docker] #  curl http://10.254.55.38:8080/v1/client-auth-service/swagger-ui.html
```



### 7.6、部署ingress controler

获取ingress_controler.yaml文件

nginx-ingress-controller即ingress controller，内部也就是一个nginx，k8s中的应用通过创建Ingress类型的对象，修改nginx的配置文件，从而根据Host路由到不同的service

```bash
#vi ingress_controler.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx

---
# Source: ingress-nginx/templates/controller-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
---
# Source: ingress-nginx/templates/controller-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # 反代配置
  proxy-body-size: "100m"
  # 格式化日志
  log-format-upstream: '{ "@timestamp": "$time_iso8601", "@fields": {"remote_addr": "$remote_addr","remote_user": "$remote_user","body_bytes_sent": "$body_bytes_sent", "request_time": "$request_time", "status": "$status", "request": "$request", "request_method": "$request_method", "http_referrer": "$http_referer", "http_x_forwarded_for": "$http_x_forwarded_for", "http_user_agent": "$http_user_agent" } }'
---
# Source: ingress-nginx/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - update
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
---
# Source: ingress-nginx/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/controller-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - configmaps
      - pods
      - secrets
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - update
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io   # k8s 1.14+
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - configmaps
    resourceNames:
      - ingress-controller-leader-nginx
    verbs:
      - get
      - update
  - apiGroups:
      - ''
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ''
    resources:
      - endpoints
    verbs:
      - create
      - get
      - update
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
---
# Source: ingress-nginx/templates/controller-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/controller-service-webhook.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  type: ClusterIP
  ports:
    - name: https-webhook
      port: 443
      targetPort: webhook
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
      nodePort: 30080
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
kind: Deployment
# kind: DaemonSet
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  revisionHistoryLimit: 10
  replicas: 3
  minReadySeconds: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: harbor.ubtrobot.com/ingress/controller:v0.34.1
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
          args:
            - /nginx-ingress-controller
            - --election-id=ingress-controller-leader
            - --ingress-class=nginx
            - --configmap=ingress-nginx/ingress-nginx-controller
            - --validating-webhook=:8443
            - --validating-webhook-certificate=/usr/local/certificates/cert
            - --validating-webhook-key=/usr/local/certificates/key
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            runAsUser: 101
            allowPrivilegeEscalation: true
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 3
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
              # 新增
              hostPort: 80
            - name: https
              containerPort: 443
              protocol: TCP
              # 新增
              hostPort: 443
            - name: webhook
              containerPort: 8443
              protocol: TCP
          volumeMounts:
            - name: webhook-cert
              mountPath: /usr/local/certificates/
              readOnly: true
            # 新增
            - name: tmp
              mountPath: /tmp
          resources:
            requests:
              cpu: 100m
              memory: 90Mi
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
        - name: webhook-cert
          secret:
            secretName: ingress-nginx-admission
      # 新增
        - name: tmp
          emptyDir: {}
      nodeSelector:
        ingress-controller-schedule: "true"
---
# Source: ingress-nginx/templates/admission-webhooks/validating-webhook.yaml
# before changing this value, check the required kubernetes version
# https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#prerequisites
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  name: ingress-nginx-admission
  namespace: ingress-nginx
webhooks:
  - name: validate.nginx.ingress.kubernetes.io
    rules:
      - apiGroups:
          - extensions
          - networking.k8s.io
        apiVersions:
          - v1beta1
        operations:
          - CREATE
          - UPDATE
        resources:
          - ingresses
    failurePolicy: Fail
    sideEffects: None
    admissionReviewVersions:
      - v1
      - v1beta1
    clientConfig:
      service:
        namespace: ingress-nginx
        name: ingress-nginx-controller-admission
        path: /extensions/v1beta1/ingresses
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
rules:
  - apiGroups:
      - admissionregistration.k8s.io
    resources:
      - validatingwebhookconfigurations
    verbs:
      - get
      - update
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-create
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
spec:
  template:
    metadata:
      name: ingress-nginx-admission-create
      labels:
        helm.sh/chart: ingress-nginx-2.11.1
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 0.34.1
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: create
          image: harbor.ubtrobot.com/ingress/kube-webhook-certgen:v1.2.2
          imagePullPolicy: IfNotPresent
          args:
            - create
            - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.ingress-nginx.svc
            - --namespace=ingress-nginx
            - --secret-name=ingress-nginx-admission
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-patch
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
spec:
  template:
    metadata:
      name: ingress-nginx-admission-patch
      labels:
        helm.sh/chart: ingress-nginx-2.11.1
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 0.34.1
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: patch
          image: harbor.ubtrobot.com/ingress/kube-webhook-certgen:v1.2.2
          imagePullPolicy: IfNotPresent
          args:
            - patch
            - --webhook-name=ingress-nginx-admission
            - --namespace=ingress-nginx
            - --patch-mutating=false
            - --secret-name=ingress-nginx-admission
            - --patch-failure-policy=Fail
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - secrets
    verbs:
      - get
      - create
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-2.11.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.34.1
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  namespace: ingress-nginx
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  limits:
  - min:
      memory: 900Mi
      cpu: 200m
    type: Container

```



```bash
# 执行
# kubectl create -f ingress_controler.yaml 

# kubectl get pod -A
NAMESPACE       NAME                                               READY   STATUS      RESTARTS   AGE
devopt          nginx-only-limit-57fbc8997c-vxzq6                  1/1     Running     3          136d
ingress-nginx   ingress-nginx-admission-create-qmfbj               0/1     Completed   0          16s
ingress-nginx   ingress-nginx-admission-patch-rl45g                0/1     Completed   1          16s
ingress-nginx   ingress-nginx-controller-57b44b584-cnm2b           0/1     Pending     0          26s
ingress-nginx   ingress-nginx-controller-57b44b584-k8gx9           0/1     Running     0          26s
ingress-nginx   ingress-nginx-controller-57b44b584-sv8qt           0/1     Pending     0          26s
kube-system     calico-kube-controllers-6c4d9c4776-mpxlp           1/1     Running     3          36d
kube-system     calico-node-57k62                                  1/1     Running     2          173d
kube-system     calico-node-5n6np                                  1/1     Running     3          173d
kube-system     coredns-75b95966b7-d85zl                           1/1     Running     3          174d
kube-system     coredns-75b95966b7-vhg7k                           1/1     Running     3          174d
kube-system     dns-autoscaler-754574499b-h26jj                    1/1     Running     3          36d
kube-system     kuboard-5ffbc8466d-fbf9v                           1/1     Running     3          36d
kube-system     metrics-server-56b49c5f5b-dkx7h                    1/1     Running     3          36d
kube-system     nginx-proxy-kuber-22                               1/1     Running     55         195d
kube-system     nginx-proxy-noder-30                               1/1     Running     10         195d
kube-system     tiller-deploy-6b89cdb79b-6f54x                     1/1     Running     3          36d
ns-mysql        mysql-app-0                                        1/1     Running     2          36d
ns-mysql        redis-app-0                                        1/1     Running     2          36d
ns-mysql        redis-app-1                                        1/1     Running     3          54d
ns-mysql        spring-deployment-authorization-568fb4c6cb-jjsph   1/1     Running     8          36d
ns-mysql        zk-0                                               1/1     Running     2          36d
ns-mysql        zk-1                                               1/1     Running     3          54d
weave           weave-scope-agent-htvfn                            1/1     Running     3          195d
weave           weave-scope-agent-m29lt                            1/1     Running     5          195d
weave           weave-scope-app-7887fbd9ff-lb87c                   1/1     Running     5          195d
weave           weave-scope-cluster-agent-6d68855547-zrm75         1/1     Running     4          195d
```



## 7.7、创建ingress

```bash
# cat ingress.yaml 
apiVersion: extensions/v1beta1 
kind: Ingress
metadata: 
  name: auth-ingress 
  namespace: ns-mysql
spec: 
  rules: 
  - host: demo.auth.cn
    http: 
      paths: 
      - path: / 
        backend: 
          serviceName: springboot-service 
          servicePort: 8080
[root@kuber-22 docker]# kubectl create -f ingress.yaml 
ingress.extensions/auth-ingress created
# kubectl -n ns-mysql get ingress
NAME           HOSTS          ADDRESS      PORTS   AGE
auth-ingress   demo.auth.cn   10.10.2.22   80      15s

```





### 7.8、访问ingress

编辑hosts文件

```bash
#hosts文件新增
10.10.2.22 demo.auth.cn
```



访问连接

```html
http://demo.auth.cn/v1/client-auth-service/swagger-ui.html
```

可以正常访问