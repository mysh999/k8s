## 一、思路

- sprint boot jar包使用容器化改造
- 后台涉及的mysql、 zk、redis需要提前部署好
- 为了防止多节点镜像不一致，需要部署私有镜像，如果没有私有镜像，需要绑定节点
- mysql、zk、redis持久化数据使用nfs存储
- 持久化应用(mysql、zk、redis)使用StatefulSet方式部署
- 非持久化应用（spring boot）使用deployment方式部署
- 按照mysql->redis->zk->spring boot顺序部署





## 二、环境说明

- 1个master、2个node
- kubernetes版本1.15.3
- 单独一台nfs服务器



## 三、NFS服务器部署

3.1、准备路径

```bash
# mkdir -p /sharenfs        #mysql数据路径
# mkdir -p /shareinit       #mysql初始化脚本存放路径
# mkdir -p /shareredis      #redis数据路径
# mkdir -p /data/zk/data1   #zk集群数据路径
# mkdir -p /data/zk/data2   #zk集群数据路径
# mkdir -p /data/zk/data3   #zk集群数据路径
```



3.2、配置nfs

```bash
# vi  /etc/exports
/sharenfs *(rw,sync,no_subtree_check,no_root_squash)
/shareinit *(rw,sync,no_subtree_check,no_root_squash)
/shareredis *(rw,sync,no_subtree_check,no_root_squash)
/data/zk/data1 *(rw,sync,no_subtree_check,no_root_squash)
/data/zk/data2 *(rw,sync,no_subtree_check,no_root_squash)
/data/zk/data3 *(rw,sync,no_subtree_check,no_root_squash)
```





3.3、重启服务

```bash
# systemctl restart nfs
# systemctl restart rpc.service
```



3.4、客户端执行挂载

```bash
# mount -t nfs 192.168.255.100:/sharenfs /nfsclient
# mount -t nfs 192.168.255.100:/shareinit /dbinit
# mount -t nfs 192.168.255.100:/shareredis /redisclient
# mount -t nfs 192.168.255.100:/data/zk/data1 /data/zk/data1
# mount -t nfs 192.168.255.100:/data/zk/data2 /data/zk/data2
# mount -t nfs 192.168.255.100:/data/zk/data3 /data/zk/data3
```





## 四、PV和PVC准备

在master节点上执行



4.1、准备创建pv和pvc文件

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
    server: 192.168.255.100
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

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-dbinit
  labels:
    pv: pv-nfs-dbinit
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: "/shareinit"
    server: 192.168.255.100
    readOnly: false
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-dbinit
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      pv: pv-nfs-dbinit
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-redis
  labels:
    pv: pv-nfs-redis
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: "/shareredis"
    server: 192.168.255.100
    readOnly: false
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-redis
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      pv: pv-nfs-redis
```



4.2、准备zk的pv文件

```bash
# cat pv_zk.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-data1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.255.100
    path: /data/zk/data1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-data2
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.255.100
    path: /data/zk/data2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-data3
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.255.100
    path: /data/zk/data3
```



4.3、执行创建后的效果

```bash
# kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pv-nfs-dbinit   10Gi       RWX            Retain           Bound    default/pvc-nfs-dbinit                           25h
pv-nfs-mysql    10Gi       RWX            Retain           Bound    default/pvc-nfs-mysql                            25h
pv-nfs-redis    10Gi       RWX            Retain           Bound    default/pvc-nfs-redis                            23h
pv-nfs-zk       10Gi       RWX            Retain           Bound    default/pvc-nfs-zk                               19h
zk-data1        10Gi       RWO            Retain           Bound    default/datadir-zok-0                            6h9m
zk-data2        10Gi       RWO            Retain           Bound    default/datadir-zok-1                            6h9m
zk-data3        10Gi       RWO            Retain           Bound    default/datadir-zok-2                            6h9m
```



```bash
# kubectl get pvc
NAME             STATUS    VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
datadir-zok-0    Bound     zk-data1        10Gi       RWO                           6h7m
datadir-zok-1    Bound     zk-data2        10Gi       RWO                           6h5m
datadir-zok-2    Bound     zk-data3        10Gi       RWO                           6h3m
pvc-nfs-dbinit   Bound     pv-nfs-dbinit   10Gi       RWX                           25h
pvc-nfs-mysql    Bound     pv-nfs-mysql    10Gi       RWX                           25h
pvc-nfs-redis    Bound     pv-nfs-redis    10Gi       RWX                           23h
pvc-nfs-zk       Bound     pv-nfs-zk       10Gi       RWX                           19h
```







## 五、创建mysql



5.1、创建secret对象存储密码

密码使用base64 加密

```bash
# echo -n 'password' | base64
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



5.2、将mysql初始化脚本放到/shareinit 共享目录下

```bash
# ls
atris_afr.sql  cri.sql  cruzr.sql
```







5.3、创建mysql应用

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
        - name: mysql-init-storage
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: pvc-nfs-mysql 
      - name: mysql-init-storage
        persistentVolumeClaim:
          claimName: pvc-nfs-dbinit
```



5.4、创建mysql svc

```bash
# cat mysql_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
```



## 六、创建redis应用

6.1、创建redis应用

```bash
# cat redis.yaml 
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: redis
spec:
  template:
    metadata:
      labels:
        app: redis
        tier: backend
    spec:
      containers:
      - image: redis
        name: redis
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-persistent-storage
          mountPath: /redis-master-data
      volumes:
      - name: redis-persistent-storage
        persistentVolumeClaim:
          claimName: pvc-nfs-redis
```



6.2、创建redis svc

```bash
# cat redis_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: redis
```





## 七、创建zk

7.1、创建zk应用

```bash
# cat zk.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zok
spec:
  serviceName: zk-hs
  replicas: 3
  selector:
    matchLabels:
      app: zk
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
        image: leolee32/kubernetes-library:kubernetes-zookeeper1.0-3.4.10
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
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
          --servers=3 \
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
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```



7.2、创建zk svc

```bash
# cat zk_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
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
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
```



## 八、spring boot工程部署

8.1、修改spring boot jar包里的配置文件

```bash
# vi ./ubtechinc-ctc-micro-service-1.0.0/BOOT-INF/classes
logging:
  path: /logs/ubtechinc-ctc-micro-service
  level:
    com.ubtechinc: info

#mysql
druid:
   url: jdbc:mysql://mysql:3306/cruzr?useUnicode=true&characterEncoding=UTF-8
   username: root
   password: password
   driver-class-name: com.mysql.jdbc.Driver

dubbo:
   enable: true
   application-name: ctc-micro-service
   scan-package: com.ubtechinc.service
   protocol-name: dubbo
   protocol-accessLog: true
   protocol-port: 20884
   provider-timeout: 3000
   provider-retries: 3
   provider-delay: -1
   registry-protocol: zookeeper
   registry-address: zookeeper://zk-cs:2181


swagger:
   enable: true
   apiKeys: X-UBT-AppId,X-UBT-DeviceId
   globalParams: X-UBT-Sign

jedis:
   enable: false
   pool:
      configstr: jedis-center1:redis:6379
      password:
      timeout: 3000
      max-total: 300
      max-idle: 200
      max-wait-millis: 1000

```

再重新打包成一个jar包



8.2、将jar工程文件放到target目录下

```bash
# ls -lt ./target/
total 49476
-rw-r--r-- 1 root root 50663001 Jan  3 14:06 ubtechinc-ctc-micro-service-1.0.0.jar
```



8.3、生成Dockerfile文件

```bash
# cat Dockerfile 
FROM openjdk:8-jre-alpine
WORKDIR /tmp
COPY target/ubtechinc-ctc-micro-service-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```



8.4、执行 docker build

```bash
# docker build -t ctc-micro-service:latest .

# docker images
REPOSITORY                                                                          TAG                              IMAGE ID            CREATED             SIZE
ctc-micro-service                                                                   latest                           e3514aff64a6        2 hours ago         136MB
```

8.5、指定节点标签

```bash
# 创建标签
kubectl label nodes k8s-master platform=beta      #节点打标签，如果有私有仓库可省略
# 删除标签
kubectl label nodes k8s-master platform-
# 查看标签
kubectl get nodes --show-labels
```

8.6、配置工程应用

```bash
# cat spring.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-jib
  template:
    metadata:
      labels:
        app: spring-boot-jib
    spec:
      nodeSelector:            #如果没有私有仓库，对image所在节点进行绑定
        platform: beta
      containers:
        - name: spring-boot-jib-pod
          image: ctc-micro-service:latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
```

<font style='color:red'>请注意，此处`nodeSelector`必须定义在`containers`之前</font>



8.7、查看运行状态

```bash
# kubectl describe pod kubectl get nodes --show-labels
```

8.8、查看业务日志

```bash
# kubectl logs -f --tail 100 kubectl get nodes --show-labels   # 查看最近100条日志
```

8.9、删除业务

```bash
# kubectl delete -f spring.yaml
```

8.10、数据库确认连接信息

```bash
mysql> show processlist;
+----+------+-------------------+-------+---------+------+-------+------------------+
| Id | User | Host              | db    | Command | Time | State | Info             |
+----+------+-------------------+-------+---------+------+-------+------------------+
|  5 | root | 10.244.0.31:34068 | cruzr | Sleep   | 7479 |       | NULL             |
|  6 | root | 10.244.0.31:34070 | cruzr | Sleep   | 7478 |       | NULL             |
|  7 | root | 10.244.0.31:34072 | cruzr | Sleep   | 7478 |       | NULL             |
|  8 | root | 10.244.0.31:34074 | cruzr | Sleep   | 7478 |       | NULL             |
|  9 | root | 10.244.0.31:34076 | cruzr | Sleep   | 7478 |       | NULL             |
| 10 | root | 10.244.0.31:34078 | cruzr | Sleep   | 7478 |       | NULL             |
| 11 | root | 10.244.0.31:34082 | cruzr | Sleep   | 7478 |       | NULL             |
| 12 | root | 10.244.0.31:34084 | cruzr | Sleep   | 7478 |       | NULL             |
| 13 | root | 10.244.0.31:34086 | cruzr | Sleep   | 7478 |       | NULL             |
| 14 | root | 10.244.0.31:34088 | cruzr | Sleep   | 7478 |       | NULL             |
| 15 | root | 10.244.0.31:34090 | cruzr | Sleep   | 7478 |       | NULL             |
| 16 | root | 10.244.0.31:34092 | cruzr | Sleep   | 7478 |       | NULL             |
| 17 | root | 10.244.0.31:34094 | cruzr | Sleep   | 7478 |       | NULL             |
| 18 | root | 10.244.0.31:34096 | cruzr | Sleep   | 7478 |       | NULL             |
| 19 | root | 10.244.0.31:34098 | cruzr | Sleep   | 7478 |       | NULL             |
| 20 | root | 10.244.0.31:34100 | cruzr | Sleep   | 7478 |       | NULL             |
| 21 | root | 10.244.0.31:34102 | cruzr | Sleep   | 7478 |       | NULL             |
| 22 | root | 10.244.0.31:34104 | cruzr | Sleep   | 7478 |       | NULL             |
| 23 | root | 10.244.0.31:34106 | cruzr | Sleep   | 7478 |       | NULL             |
| 24 | root | 10.244.0.31:34108 | cruzr | Sleep   | 7478 |       | NULL             |
| 27 | root | localhost         | NULL  | Query   |    0 | init  | show processlist |
+----+------+-------------------+-------+---------+------+-------+------------------+
```



```bash
# kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
linuxhub-web-deploy-5c6557c579-5j6xb   1/1     Running   2          7d2h    192.168.255.13   k8s-node2    <none>           <none>
linuxhub-web-deploy-5c6557c579-8ctl9   1/1     Running   587        7d2h    192.168.255.11   k8s-master   <none>           <none>
mysql-0                                1/1     Running   0          3h27m   10.244.2.41      k8s-node2    <none>           <none>
nginx-8f6959bd-87hmn                   1/1     Running   4          108d    10.244.0.28      k8s-master   <none>           <none>
nginx-8f6959bd-djbxj                   1/1     Running   5          108d    10.244.1.56      k8s-node1    <none>           <none>
nginx-test-7d875b886d-g9vrh            1/1     Running   4          108d    10.244.1.60      k8s-node1    <none>           <none>
nginx-test-7d875b886d-lmdrt            1/1     Running   4          108d    10.244.0.29      k8s-master   <none>           <none>
nginx-test-7d875b886d-v4k7x            1/1     Running   4          108d    10.244.1.53      k8s-node1    <none>           <none>
redis-bd64bf88-snp9k                   1/1     Running   0          8h      10.244.2.38      k8s-node2    <none>           <none>
redis-prod-569698fc79-d9ts6            1/1     Running   4          108d    10.244.1.58      k8s-node1    <none>           <none>
redis-prod-569698fc79-kdbv4            1/1     Running   4          108d    10.244.0.26      k8s-master   <none>           <none>
redis-prod-569698fc79-s6dfz            1/1     Running   4          108d    10.244.0.27      k8s-master   <none>           <none>
redis-qa-5675d75574-4tr97              1/1     Running   4          108d    10.244.1.54      k8s-node1    <none>           <none>
spring-deployment-6f44d48dd6-qn459     1/1     Running   0          125m    10.244.0.31      k8s-master   <none>           <none>
```

<font style='color:red'>spring-deployment-6f44d48dd6-qn459     1/1     Running   0          125m    10.244.0.31      k8s-master   <none>           <none></font>

10.244.0.31 IP和查询到的相符



8.11、创建svc

```bash
# cat spring_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: spring
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: spring
```



## 九、部署ingress

9.1、安装 Nginx Ingress Controller

```yaml
# cat mandatory.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
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
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

---
```

执行

```
#kubectl create -f mandatory.yaml 
```

查看Nginx Ingress Controller

```bash
# kubectl get  pods -n ingress-nginx -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
nginx-ingress-controller-588f7dbb99-5ftqm   1/1     Running   0          81m   10.10.1.174   k8s01   <none>           <none>
```

9.2、部署ingress

```yaml
# cat ingress.yml 
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-web
  annotations:
    # 指定 Ingress Controller 的类型
    kubernetes.io/ingress.class: "nginx"
    # 指定我们的 rules 的 path 可以使用正则表达式
    nginx.ingress.kubernetes.io/use-regex: "true"
    # 连接超时时间，默认为 5s
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    # 后端服务器回转数据超时时间，默认为 60s
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    # 后端服务器响应超时时间，默认为 60s
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    # 客户端上传文件，最大大小，默认为 20m
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # URL 重写
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  # 路由规则
  rules:
  # 主机名，只能是域名，修改为你自己的
  - host: k8s.test.com
    http:
      paths:
      - path:
        backend:
          # 后台部署的 Service Name，与上面部署的 Tomcat 对应
          serviceName: spring
          # 后台部署的 Service Port，与上面部署的 Tomcat 对应
          servicePort: 8080
```

执行

```
#kubectl create -f ingress.yml 
```

查看ingress

```
# kubectl get ingress
NAME                 HOSTS                              ADDRESS         PORTS     AGE
nginx-web            k8s.test.com                                       80        90m
```

## 

9.3、客户端访问

修改客户端的hosts文件

客户端访问

```
http://k8s.test.com/
```



附源码地址：

https://github.com/mysh1984/k8s/tree/master/sprint boot src

