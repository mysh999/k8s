## 一、概念说明

- Helm :

 Kubernetes 的包管理器，可以帮我们简化 kubernetes 的操作，一键部署应用，类似redhat的yum

Helm把Kubernetes资源(比如deployments、services或 ingress等) 打包到一个chart中，而chart被保存到chart仓库。通过chart仓库可用来存储和分享chart。Helm使发布可配置，支持发布应用配置的版本管理，简化了Kubernetes部署应用的版本控制、打包、发布、删除、更新等操作



- Chart: 

一系列 k8s 资源集合的命名，它包含一系列 k8s 资源配置文件的模板与参数，可供灵活配置

类似于docker中的image



- release: 

当一个 Chart 部署后生成一个 release

类似于docker中的container



- repo: 

即 chart 的仓库，其中有很多个 chart 可供选择，如官方 helm/charts



组成：

- helm客户端-HelmClient

1、制作、拉取、查找和验证 Chart
2、安装服务端Tiller
3、指示服务端Tiller做事，比如根据chart创建一个Release



- helm服务端 tiller-TillerServer

1、安装在Kubernetes集群内的一个应用， 用来执行客户端发来的命令，管理Release





## 二、安装

2.1 安装依赖

​      在所有node安装socat 软件

```bash
# yum -y install socat
```



2.2  HelmClient 的安装

有2种安装方式：二进制和脚本安装

这里采用二进制

```bash
# wget https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
# tar -zxvf helm-v2.14.1-linux-amd64.tar.gz 
# cp linux-amd64/helm /usr/local/bin/

# helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Error: could not find tiller
```



2.3 安装helm服务端tiller

​      替换 tiller 的镜像为为阿里云的镜像，Helm 的 stable 仓库源也更换为阿里云的 stable 仓库源

```bash
# helm init --upgrade \
> -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3 \
> --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 


Creating /root/.helm 
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```



2.4  确认服务端tiller

```bash
# kubectl get pods -n kube-system |grep tiller
tiller-deploy-599c9dff95-s677h             1/1     Running   0          113s

#确认客户端和服务端连接成功。如果只显示了客户端版本，说明没有连上服务端。 它会自动去K8s上kube-system命名空间下查找是否有Tiller的Pod在运行
# helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
```



2.5 chart 仓库

```bash
# helm repo list
NAME    URL                                                   
stable  https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local   http://127.0.0.1:8879/charts     
```



2.6、添加国内helm库

```bash
# helm repo add stable https://burdenbear.github.io/kube-charts-mirror/
# helm repo add stable2 https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
# helm repo add repo_name1 https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
# helm repo add stable3 https://burdenbear.github.io/kube-charts-mirror/
```



```bash
# helm repo list
NAME            URL                                                                      
stable          https://burdenbear.github.io/kube-charts-mirror/                         
local           http://127.0.0.1:8879/charts                                             
stable2         https://burdenbear.github.io/kube-charts-mirror/                         
repo_name1      https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
stable3         https://burdenbear.github.io/kube-charts-mirror/        
```



2.7、更新到最新的charts列表

```bash
# helm repo update
```



2.8、查看可安装的charts

Helm 将 Charts 包安装到 Kubernetes 集群中，一个安装实例就是一个新的 Release，要找到新的 Chart，我们可以通过搜索命令完成

```bash
# helm search
NAME                                                    CHART VERSION   APP VERSION             DESCRIPTION                                                 
repo_name1/ack-advanced-horizontal-pod-autoscaler       0.1.0           1.0                     A Helm chart for Kubernetes                                 
repo_name1/ack-ahas-pilot                               1.6.1           1.6.1                   A cloud service that aims to improve the high availabilit...
repo_name1/ack-ahas-sentinel-pilot                      0.1.0           0.1.0                   Ahas sentinel Pilot - Webhook Admission Controller          
repo_name1/ack-ahas-springcloud-gateway                 0.1.1           0.1.1                   Spring Cloud Gateway with AHAS Sentinel integration Helm ...
repo_name1/ack-alibaba-cloud-metrics-adapter            1.2.0           0.1.1                   An implementation of the Kubernetes Custom Metrics API an...
repo_name1/ack-arms-pilot                               0.1.2           1.0.2                   ARMS Pilot - Webhook Admission Controller                   
repo_name1/ack-arms-prometheus                          0.1.2           1.0.3                   ARMS Prometheus Operator                                    
repo_name1/ack-consul                                   0.5.0           0.5.0                   Install and configure Consul on Kubernetes.                 
repo_name1/ack-etcd-backup-operator                     0.1.0           1.0                     This is an etcd backup operator for alibaba cloud kuberne...
repo_name1/ack-federation-v2                            0.2.3           0.0.10                  Kubernetes Federation V2 helm chart                         
repo_name1/ack-jenkins                                  2.205           1.0                     Open source continuous integration server. It supports mu...
repo_name1/ack-knative-eventing                         0.11.0          0.11.0                  Helm chart for Knative eventing                             
repo_name1/ack-knative-kafka-sources                    0.11.1          0.11.1                  Helm chart for Knative eventing kafka sources               
repo_name1/ack-knative-mnsoss-sources                   1.0.0           1.0.0                   Helm chart for Knative eventing-mnsoss-sources              
repo_name1/ack-knative-serving                          0.11.0          0.11.0                  Helm chart for Knative serving                              
repo_name1/ack-knative-sources                          0.11.1          0.11.1                  Helm chart for Knative eventing-sources                     
repo_name1/ack-kruise                                   0.1.0           0.1.0                   Helm chart for all kruise-manager components    
```





## 三、基础使用

3.1	查找仓库中可用的chart，比如查找mysql

```bash
# helm search mysql
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                                 
stable/mysql                    0.3.5                           Fast, reliable, scalable, and easy to use open-source rel...
stable/percona                  0.3.0                           free, fully compatible, enhanced, open source drop-in rep...
stable/percona-xtradb-cluster   0.0.2           5.7.19          free, fully compatible, enhanced, open source drop-in rep...
stable/gcloud-sqlproxy          0.2.3                           Google Cloud SQL Proxy                                      
stable/mariadb                  2.1.6           10.1.31         Fast, reliable, scalable, and easy to use open-source rel...

#查询包的具体情况
# helm inspect stable/mysql
```



3.2	修改tiller权限

​	默认安装的 tiller 权限很小，我们执行下面的脚本给它加最大权限，这样方便我们可以用 helm 部署应用到任意 namespace 下

```bash
[root@ubt-k8s-master-01 pod]# kubectl create serviceaccount --namespace=kube-system tiller
serviceaccount/tiller created
[root@ubt-k8s-master-01 pod]#  kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding.rbac.authorization.k8s.io/tiller-cluster-rule created
[root@ubt-k8s-master-01 pod]# kubectl patch deploy --namespace=kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
deployment.extensions/tiller-deploy patched
```



3.3  部署程序

```bash
# helm install stable3/mysql

# helm install --name mysql --set mysqlRootPassword=jicki  stable/mysql

# helm install --name redis-prod --set redis_image=quay.io/smile/redis:4.0.6r2 stable/redis-ha
```



3.4 查看部署的release 

```bash
# helm ls
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
solid-parrot    1               Wed Jan 15 17:52:54 2020        DEPLOYED        mysql-1.6.2     5.7.28          default   

# helm status redis-prod 
```



3.5、删除chart

```bash
# helm delete mysql
```



确认是否删除

```bash
# helm ls -a mysql
NAME    REVISION        UPDATED                         STATUS  CHART           APP VERSION     NAMESPACE
mysql   1               Fri Dec  6 16:19:24 2019        DELETED mysql-0.3.5                     default  
```



彻底删除release

```bash
# helm delete --purge mysql
```









## 四、安装nginx

4.1、添加helm库

```bash
# helm repo add bitnami https://charts.bitnami.com/bitnami

# helm search bitnami
```



4.2、安装nginx

```bash
# helm install --name mywebserver bitnami/nginx

# kubectl describe deployment mywebserver
Name:                   mywebserver-nginx
Namespace:              default
CreationTimestamp:      Thu, 16 Jan 2020 20:13:24 +0800
Labels:                 app.kubernetes.io/instance=mywebserver
                        app.kubernetes.io/managed-by=Tiller
                        app.kubernetes.io/name=nginx
                        helm.sh/chart=nginx-5.1.3
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app.kubernetes.io/instance=mywebserver,app.kubernetes.io/name=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app.kubernetes.io/instance=mywebserver
           app.kubernetes.io/managed-by=Tiller
           app.kubernetes.io/name=nginx
           helm.sh/chart=nginx-5.1.3
  Containers:
   nginx:
    Image:        docker.io/bitnami/nginx:1.16.1-debian-9-r139
    Port:         8080/TCP
    Host Port:    0/TCP
    Liveness:     http-get http://:http/ delay=30s timeout=5s period=10s #success=1 #failure=6
    Readiness:    http-get http://:http/ delay=5s timeout=3s period=5s #success=1 #failure=3
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   mywebserver-nginx-6dfcf6fcb8 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  23m   deployment-controller  Scaled up replica set mywebserver-nginx-6dfcf6fcb8 to 1
  
 

```



4.3、查看状态 

```bash
 
 # kubectl get pods -l app.kubernetes.io/name=nginx
NAME                                 READY   STATUS    RESTARTS   AGE
mywebserver-nginx-6dfcf6fcb8-rgkpl   1/1     Running   0          24m

# kubectl get service mywebserver-nginx -o wide
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
mywebserver-nginx   LoadBalancer   10.254.70.215   <pending>     80:30395/TCP,443:30603/TCP   24m   app.kubernetes.io/instance=mywebserver,app.kubernetes.io/name=nginx
```



4.4、查看效果

```bash
# curl 10.254.70.215                              
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```







## 五、安装redis（包括持久化存储）

5.1、安装storage

```bash
#使用了 WaitForFirstConsumer 延迟绑定pv，直到 pod 被调度
# cat storage.yml 
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage-redis
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

# kubectl create -f storage.yml 

# kubectl get storageclass
NAME                  PROVISIONER                    AGE
local-storage-redis   kubernetes.io/no-provisioner   68m
```



5.2、安装PV绑定

```bash
#提前在k8snode01节点上创建目录
# mkdir -p  /app/kelu/local-storage/redis

#定义 nodeAffinity，Kubernetes Scheduler 需要使用 PV 的 nodeAffinity 描述信息来保证 Pod 能够调度到有对应 local volume 的机器上
# cat pv.yml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-redis
spec:
  capacity:
    storage: 8Gi
  # volumeMode field requires BlockVolume Alpha feature gate to be enabled.
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage-redis
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /app/kelu/local-storage/redis
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8snode01
          
          
          
# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                          STORAGECLASS          REASON   AGE
local-pv-redis   8Gi        RWO            Retain           Available                                  local-storage-redis            63m
```



5.3、配置values

去到helm维护的 Redis 目录： https://github.com/helm/charts/tree/master/stable/redis ，下载 values.yaml 文件

搜索 persistence ，填充 storage class的内容。

![360截图16751026385349.jpg](http://ww1.sinaimg.cn/large/007Xg1efgy1gazduwoodfj30jb09cdgg.jpg)



5.4、安装

```bash
# helm search redis
# helm install --name redis -f values.yaml stable/redis --namespace kelu
安装到 kelu 这个namespace下
```





## 六、部署mysql（持久化数据）

6.1、在node01节点上创建mysql目录

```bash
# mkdir -p /opt/mysql
```



6.2、准备pv

```bash
# cat mysql_pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity: 
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /opt/mysql                   #对应node01节点目录
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8snode01
          
          
# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                          STORAGECLASS          REASON   AGE
mysql-pv         2Gi        RWO,ROX        Retain           Bound       default/mysql-pvc                                             35m
```



6.3、准备pvc

```bash
# cat mysql_pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
      
      
# kubectl get pvc
NAME                                  STATUS    VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc                             Bound     mysql-pv       2Gi        RWO,ROX                       31m
```



6.4、安装mysql

```bash
# helm install --name mysql --set mysqlRootPassword=rootpassword,mysqlUser=mysql,mysqlPassword=my-password,mysqlDatabase=mydatabase,persistence.existingClaim=mysql-pvc stable/mysql
```



6.5、查看pod

```bash
# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
mysql-mysql-6cbf467756-srkds                       1/1     Running   0          13m
```



6.6 、按照安装的输出提示登陆验证

```bash
# MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

# echo $MYSQL_ROOT_PASSWORD
rootpassword

# kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

root@ubuntu:/# apt-get update && apt-get install mysql-client -y

root@ubuntu:/ $ mysql -h mysql-mysql -p

```

