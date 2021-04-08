参考文档：

https://www.cnblogs.com/tylerzhou/p/11026364.html

https://www.cnblogs.com/ssgeek/p/12768927.html





## 一、概念

- 污点（taints）是定义在节点上的键值型属性数据。用于让节点拒绝将POD调度运行其上，除非该POD对象具有容纳节点污点的容忍度（tolerations）。

- 容忍度（tolerations）是定义在POD对象上的键值型属性数据，用于配置可容忍的节点污点。

污点和容忍结合在一起可以确保POD不会调度到不适合的节点上。





## 二、和节点选择器、亲和性区别

节点选择器（nodeSelector）和节点亲和性（nodeAffinity）两种调度方式通过在Pod对象上添加标签选择器来完成对特定类型节点标签匹配。实现的是由POD选择节点的机制。

污点和容忍度是通过向节点添加污点信息来控制POD对象的调度结果，赋予节点控制何种POD对象能够调度其上的主控权。

节点亲和性使得POD被吸引到一类特定节点，污点相反，提供让节点排斥特定POD对象





## 三、污点和容忍度属性

污点定义在节点的node spec中

容忍度定义在POD的podspec中

都额外支持一个效果effect标记。effect用于定义对Pod对象的排斥等级，包含以下三种类型：



- NoSchedule

  不能容忍此污点的新POD对象不可调度至当前节点，节点上现存POD对象不受影响

  

- PreferNoSchedule

  不能容忍此污点的新POD对象尽量不要调度至当前节点，不过无其他节点可供调度时也允许接收相应的POD对象，节点上现存POD对象不受影响

  

- NoExecute 

  不能容忍此污点的新POD对象不可调度至当前节点，而且节点上现存的`Pod`对象因节点污点变动或`Pod`容忍度变动而不再满足匹配规则时，`Pod`对象将被驱逐。



使用`kubeadm`部署的`Kubernetes`集群，其`Master`节点将自动添加污点信息以阻止不能容忍此污点的`Pod`对象调度至此节点



## 四、管理污点

### 4.1、定义污点

```bash
# kubectl taint nodes k8s-node1 gpu=yes:NoSchedule
```



### 4.2、查看污点

```bash
#  kubectl describe node k8s-node1|grep -i taint 
Taints:             gpu=yes:NoSchedule
```



### 4.3、给pod添加容忍该污点

```bash
# cat nginx-app.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: ns-mysh
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.4
        ports:
        - containerPort: 80
      tolerations:     #新增容忍度
      - key: "gpu"
        operator: "Equal"
        value: "yes"
        effect: "NoSchedule"
```



### 4.4、查看pod

```bash
# kubectl -n ns-mysh get pods -o wide|grep -i nginx
nginx-app-9bc86dbd4-dbbpf                 0/1     ContainerCreating   0          69s     <none>           k8s-node1   <none>           <none>
nginx-app-9bc86dbd4-lhpmx                 0/1     ContainerCreating   0          69s     <none>           k8s-node1   <none>           <none>
```

创建的pod，在有污点的node1上运行





### 4.5、删除污点

```bash
# kubectl taint nodes k8s-node1 gpu=yes:NoSchedule-
```

在定义污点的命令添加-，就是删除