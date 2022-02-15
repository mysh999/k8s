参考来源：

https://blog.csdn.net/shm19990131/article/details/107087817



## 一、什么是helm

在没有使用 helm 之前，在 kubernetes 部署应用，需要依次部署 deployment、SVC 等步骤较为繁琐。

况且，随着很多项目微服务化，复杂的应用在容器中部署以及管理显得较为复杂，helm通过打包得方式，支持发布得版本管理和控制，很大程度上简化了 kubernetes 应用得部署以及管理。

Helm 本质就是让 K8S 得应用管理可方便配置，能动态生成。通过动态生成 K8S 资源清单文件。然后带哦用 kubectl 自动执行 K8S 资源部署



Helm 是官方提供的类似于 YUM 得包管理工具，是部署环境的流程封装。
Helm 有两个重要得概念：chart 和 release

 chart ：是创建一个应用的信息集合，包括各种 kubernetes 对象得配置模板、参数定义、依赖关系、文档说明等。chart 是应用部署得自包含逻辑单元。可以将 chart 想象成 apt\yum 中得软件安装包。
 release ：是 chart 得运行实例，代表了一个正在运行得应用。当 chart 被安装到 kubernetes 集群，就会生成一个 release。chart 能够多次安装到同一个集群，但是只会有一个 release。





## 二、helm组件

![企业微信截图_20220215175107.png](https://tva1.sinaimg.cn/large/007Xg1efgy1gzecg0hpvxj30hu0810tw.jpg)



1、Helm Client 负责 chart 和 release 创建和管理， 通过 gRPC 得方式与 Tiller 进行交互。
2、Tiller 服务端运行在 kubernetes 集群中，它会处理 Helm 得请求，并发送给 KubeAPI。
3、KubeAPI 将数据、资源得生成写入到 etcd ，被 kubelet 接受并创建。





## 三、安装helm

helm 由客户端 helm 命令行工具和服务端 tiller 组成

helm 的安装十分简单。下载 helm 命令行工具到 master 节点 node1的 /usr/local/bin 下。

https://github.com/helm/helm/releases/tag/v3.2.4  

```bash
# tar -zxvf  helm-v3.2.4-linux-amd64.tar.gz
# mv linux-amd64/helm /usr/local/bin
# helm version
version.BuildInfo{Version:"v3.2.4", GitCommit:"0ad800ef43d3b826f31a5ad8dfbb4fe05d143688", GitTreeState:"clean", GoVersion:"go1.13.12"}

```





## 四、常用chart源

准备好 helm 后，需要添加 helm 源数据仓库。有以下几个常见的源库

```bash
# helm repo add  aliyuncs https://apphub.aliyuncs.com
```



查看当前库

```bash
# helm repo list
NAME            URL                        
aliyuncs        https://apphub.aliyuncs.com
```





查看chart库有哪些可安装程序

```bash
# helm search repo aliyuncs|more
NAME                                    CHART VERSION   APP VERSION                     DESCRIPT
ION                                       
aliyuncs/admin-mongo                    0.1.0           1                               MongoDB
理工具(web gui)                          
aliyuncs/aerospike                      0.3.2           v4.5.0.5                        A Helm c
hart for Aerospike in Kubernetes          
aliyuncs/airflow                        4.3.3           1.10.9                          Apache A
irflow is a platform to programmaticall...
aliyuncs/ambassador                     5.3.0           0.86.1                          A Helm c
hart for Datawire Ambassador              
aliyuncs/apache                         7.3.5           2.4.41                          Chart fo
r Apache HTTP Server                      
aliyuncs/apm-server                     2.1.5           7.0.0                           The serv
er receives data from the Elastic APM a...
aliyuncs/arkid                          0.1.0           1.0                             ArkID is
 an enterprise SSO solutions, it suppor...
aliyuncs/atlantis                       3.11.0          v0.11.1                         A Helm c
hart for Atlantis https://www.runatlant...
```





查找想要安装的软件

```bash
# helm search repo nginx
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION                                       
aliyuncs/nginx                          5.1.5           1.16.1                  Chart for the nginx server                        
aliyuncs/nginx-ingress                  1.30.3          0.28.0                  An nginx Ingress controller that uses ConfigMap...
aliyuncs/nginx-ingress-controller       5.3.4           0.29.0                  Chart for the nginx Ingress controller            
aliyuncs/nginx-lego                     0.3.1                                   Chart for nginx-ingress-controller and kube-lego  
aliyuncs/nginx-php                      1.0.0           nginx-1.10.3_php-7.0    Chart for the nginx php server                   
```





## 五、安装helm应用

```bash
#  helm install nginx  aliyuncs/nginx
NAME: nginx
LAST DEPLOYED: Tue Feb 15 17:41:12 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NGINX URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w nginx'

  export SERVICE_IP=$(kubectl get svc --namespace default nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "NGINX URL: http://$SERVICE_IP/"

# kubectl get svc
NAME            TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                  AGE
emqx-headless   ClusterIP      None         <none>        1883/TCP,8883/TCP,8081/TCP,8083/TCP,8084/TCP,18083/TCP   3d23h
kubernetes      ClusterIP      10.0.0.1     <none>        443/TCP                                                  30d
nginx           LoadBalancer   10.0.0.113   <pending>     80:31685/TCP,443:30686/TCP                               20s

# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
emqx-statefulset-0       1/1     Running   0          3d23h
emqx-statefulset-1       1/1     Running   0          3d23h
emqx-statefulset-2       1/1     Running   0          3d23h
nginx-84cc856498-2mcfs   1/1     Running   0          28s

                          32s
[root@k8s-master1 ~]# curl -l 10.0.0.113:31685
```



跟踪发布的状态

```bash
#  helm status nginx
NAME: nginx
LAST DEPLOYED: Tue Feb 15 17:41:12 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NGINX URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w nginx'

  export SERVICE_IP=$(kubectl get svc --namespace default nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "NGINX URL: http://$SERVICE_IP/"
```





查看helm生成的应用

```bash
# helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS         CHART            APP VERSION
nginx   default         1               2022-02-15 17:41:12.122475589 +0800 CST deployed       nginx-5.1.5      1.16.1     
```



卸载应用

```bash
# helm uninstall nginx
release "nginx" uninstalled
```

