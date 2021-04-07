## 一、目的

​    通过storgeclass创建动态PV，利用NFS来实现

   使用到 nfs-client 的自动配置程序，我们也叫它 Provisioner，这个程序使用已经配置好的 nfs 服务器，来自动创建持久卷





## 二、环境说明

| NFS Server | 192.168.188.61 |      |
| ---------- | -------------- | ---- |
| NFS客户端  | 192.168.188.61 |      |
|            | 192.168.188.62 |      |
|            | 192.168.188.63 |      |





## 三、NFS配置

### 3.1、服务端配置

```bash
# yum -y install nfs-utils rpcbind
# mkdir /data/nfsserver -p

# cat /etc/exports
/data/nfsserver *(rw,sync,no_root_squash)

# systemctl start rpcbind
# systemctl enable rpcbind
# systemctl status rpcbind

# systemctl start nfs
# systemctl enable nfs

```



### 3.2、客户端配置

```bash
# yum -y install nfs-utils rpcbind
# systemctl start rpcbind
# systemctl enable rpcbind
# systemctl start nfs
# systemctl enable nfs

# showmount -e 192.168.188.61
Export list for 192.168.188.61:
/data/nfsserver *
```





## 四、配置NFS存储

### 4.1、获取相关资源

```bash
# git clone https://github.com/kubernetes-incubator/external-storage.git
# cd external-storage/nfs-client/deploy/

```



### 4.2、创建NFS控制器

```bash
# 将deployment.yaml配置文件中的nfs服务器和路径修改为自己nfs服务器和路径
# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: ns-mysh
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.188.61
            - name: NFS_PATH
              value: /data/nfsserver
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.188.61
            path: /data/nfsserver
            
 # kubectl -n ns-mysh get pod
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-7db6484544-2zv7c   1/1     Running   2          73m
```





### 4.3、绑定权限

```bash
# cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: ns-mysh
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: ns-mysh
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: ns-mysh
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: ns-mysh
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: ns-mysh
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
  
  
  
  # kubectl -n ns-mysh get role
NAME                                    CREATED AT
leader-locking-nfs-client-provisioner   2021-04-07T19:18:18Z
[root@k8s-master deploy]# kubectl -n ns-mysh get sa
NAME                     SECRETS   AGE
default                  1         148m
nfs-client-provisioner   1         78m
```



### 4.4、创建sc

```bash
# cat class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
  
 
 # kubectl get sc
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  76m
```





## 五、POD使用NFS

### 5.1、调用SC动态提供PV创建PVC

```bash
# cat mysql-app.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app-mysql-dev
  namespace: ns-mysh
spec:
  selector:
    matchLabels:
      app: app-mysql-dev
  serviceName: app-mysql-svc
  replicas: 1
  template:
    metadata:
      labels:
        app: app-mysql-dev
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
              name: mysql-secret-dev
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
      namespace: ns-mysh
      annotations:
        volume.beta.kubernetes.io/storage-class: managed-nfs-storage
    spec:
      accessModes: 
        - ReadWriteMany
      resources:
        requests:
          storage: 1Gi
      
      
# kubectl -n ns-mysh get pvc
NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
mysql-persistent-storage-app-mysql-dev-0   Bound    pvc-df3e7017-afbe-4e8a-8ce0-a79fae39f6d5   1Gi        RWX            managed-nfs-storage   64s


# kubectl -n ns-mysh get pod
NAME                                      READY   STATUS    RESTARTS   AGE
app-mysql-dev-0                           1/1     Running   0          91s
nfs-client-provisioner-7db6484544-2zv7c   1/1     Running   2          103m
```

备注：如果创建PVC状态一直pending，

 通过kubectl describe命令查看错误提示信息，信息中有：waiting for a volume to be created, either by external provisioner “fuseim.pri/ifs” or manually created by system administrator
可能是1.20版本默认禁用使用selfLink



解决办法；

通过find / -name kube-apiserver.yaml 命令，找到kube-apiserver.yaml文件

```bash
#vi /etc/kubernetes/manifests/kube-apiserver.yaml
--- ----
spec:
  containers:
  - command:
    - kube-apiserver
    - --feature-gates=RemoveSelfLink=false     #新增
    - --advertise-address=192.168.188.61
```







