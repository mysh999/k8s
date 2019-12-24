## 一、环境说明

1、K8S集群版本：V1.15.3

2、EFK镜像从 docker.elastic.co站点下载

3、各节点内存要够大，不然启动ES的时候，会报

ES启动日志会报“OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.”



EFK 组合插件是k8s项目的日志解决方案，它包括三个组件：Elasticsearch, Fluentd, Kibana。相对于ELK这样的架构，k8s官方推行了EFK，Fluentd相对于Logstash更加轻量级。

Elasticsearch 是日志存储和日志搜索引擎，Fluentd 负责把k8s集群的日志发送给 Elasticsearch, Kibana 则是可视化界面查看和检索存储在 Elasticsearch 的数据。



GitHub官网：https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch

​      



## 二、下载镜像

提前下载好镜像

```bash
# docker pull docker.elastic.co/elasticsearch/elasticsearch:6.6.2
# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/fluentd-elasticsearch:v2.4.0
# docker pull docker.elastic.co/kibana/kibana-oss:6.6.1
```



## 三、准备EFK部署文件

3.1、修改 es-statefulset.yaml 

```bash

 # cat es-statefulset.yaml |grep -i image:
   - image: docker.elastic.co/elasticsearch/elasticsearch:6.6.2
  - image: alpine:3.6
```


3.2、修改fluentd-es-ds.yaml

```bash
# cat fluentd-es-ds.yaml |grep -i image:                   
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/fluentd-elasticsearch:v2.4.0
```



其他不用修改，修改后的文件已经上传到github上

下载地址：https://github.com/mysh1984/k8s/tree/master/EFK



3.3、执行

```bash
# kubectl apply -f .
service/elasticsearch-logging created
serviceaccount/elasticsearch-logging created
clusterrole.rbac.authorization.k8s.io/elasticsearch-logging created
clusterrolebinding.rbac.authorization.k8s.io/elasticsearch-logging created
statefulset.apps/elasticsearch-logging created
configmap/fluentd-es-config-v0.2.0 created
serviceaccount/fluentd-es created
clusterrole.rbac.authorization.k8s.io/fluentd-es created
clusterrolebinding.rbac.authorization.k8s.io/fluentd-es created
daemonset.apps/fluentd-es-v2.4.0 created
deployment.apps/kibana-logging created
service/kibana-logging created
```



3.4、查看状态

```bash
# kubectl get pods -n kube-system -o wide
```

![360截图174001198996129.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga830ofx2mj30vr07fgng.jpg)



```ba
#kubectl get pods -n kube-system -o wide|grep -E 'elasticsearch|fluentd|kibana'
```

![360截图18260728286443.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga8323ctvmj30u002hgml.jpg)



```bash
#  kubectl get service  -n kube-system|grep -E 'elasticsearch|kibana'
```

![360截图17400117043455.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga8332djxyj30md01aweq.jpg)



3.5、查看pod日志

```bash
# kubectl logs elasticsearch-logging-0 -n kube-system -f
```



## 四、访问kibana


4.1、通过 kube-apiserver 访问：

```bash
# kubectl cluster-info|grep -E 'Elasticsearch|Kibana'

Elasticsearch is running at https://192.168.230.100:6443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
Kibana is running at https://192.168.230.100:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
```



浏览器访问 URL： https://172.27.137.240:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy 



4.2、通过 kubectl proxy 访问：

创建代理

```bash
# nohup  kubectl proxy --address='192.168.230.100' --port=8086 --accept-hosts='^*$' &
Starting to serve on 192.168.230.100:8086
```




浏览器访问 URL：http://192.168.230.100:8086/api/v1/namespaces/kube-system/services/kibana-logging/proxy

![360截图17430808185264.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga838xm2gfj31730q3tcz.jpg)



4.3、创建索引

![360截图18750813483683.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga83a03jg4j31bo0j8acj.jpg)

![360截图17400111747198.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga83apj1scj31bs0hlmzf.jpg)



4.4、查看索引

![360截图1814122183109113.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ga83caii42j31c20q60wl.jpg)