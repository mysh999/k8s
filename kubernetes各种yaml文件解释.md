参考来源：

```html
https://blog.csdn.net/Ay_Ly/article/details/89393281?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.channel_param

https://blog.csdn.net/qq_24794401/article/details/103827386?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param

https://www.cnblogs.com/saneri/p/9288239.html

https://blog.51cto.com/dongdong/2434128

```



## 一、概念说明

1.1、pod控制器分类：
1、ReplicationController
2、ReplicaSet
3、Deployment
4、StatefulSet
5、DaemonSet
6、Job,Cronjob
7、HPA



1.2、pod控制器：一般包括3部分
1、标签选择器
2、期望的副本数（DaemonSet控制器不需要）
3、pod模板



1.3、deploy控制器构建于rs控制器之上，新特性包括：
1、事件和状态查看
2、回滚
3、版本记录
4、暂停和启动
5、支持两种自动更新方案
      Recreate删除重建
      RollingUpdate回滚升级（默认方式）

创建deploy控制器时会自动创建rs控制器，二者具有相同的标签选择器



1.4、通常情况下，Deployment 被用来部署无状态服务，那么对于有状态服务的部署，使用 StatefulSet 进行有状态服务的部署



## 二、deployment.yaml文件

2.1、Deployment API 版本对照表

![企业微信截图_20200805160611.png](http://ww1.sinaimg.cn/large/007Xg1efgy1ghfzz1jqx8j30oa04x3ye.jpg)





2.2、Deployment yaml文件包含四个部分

- apiVersion: 表示版本


- kind: 表示资源


- metadata: 表示元信息


- spec: 资源规范字段



2.3、deployment.yaml文件示例



示例一：nginx_deployment.yaml

```yaml
apiVersion: extensions/v1beta1   #当前格式的版本
kind: Deployment                 #当前创建资源的类型， 当前类型是Deployment
metadata:                        #当前资源的元数据
  name: nginx-test               #当前资源的名字 是元数据必须的项
spec:                            #是当前Deployment的规格说明
  replicas:                      #指当前创建的副本数量 默认不填 默认值就为‘1’
  template:                      #定义pod的模板
    metadata:                    #当前pod的元数据
      labels:                    #至少顶一个labels标签，可任意创建一个 key:value
        app: web_server
    spec:                        #当前pod的规格说明
      containers:                #容器
      - name: nginx              #是容器的名字容器名字是必须填写的
        image: nginx:latest      #镜像 镜像的名字和版本
```







示例二：

```yaml
apiVersion: apps/v1  # 指定api版本，此值必须在kubectl api-versions中  
kind: Deployment  # 指定创建资源的角色/类型   
metadata:  # 资源的元数据/属性 
  name: demo  # 资源的名字，在同一个namespace中必须唯一
  namespace: default # 部署在哪个namespace中
  labels:  # 设定资源的标签
    app: demo
    version: stable
spec: # 资源规范字段
  replicas: 1 # 声明副本数目
  revisionHistoryLimit: 3 # 保留历史版本
  selector: # 选择器
    matchLabels: # 匹配标签
      app: demo
      version: stable
  strategy: # 策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 示在更新过程中能够进入不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
    type: RollingUpdate # 滚动更新策略
  template: # 模版
    metadata: # 资源的元数据/属性 
      annotations: # 自定义注解列表
        sidecar.istio.io/inject: "false" # 自定义注解名字
      labels: # 设定资源的标签
        app: demo
        version: stable
    spec: # 资源规范字段
      containers:
      - name: demo # 容器的名字   
        image: demo:v1 # 容器使用的镜像地址   
        imagePullPolicy: IfNotPresent # 每次Pod启动拉取镜像策略，三个选择 Always、Never、IfNotPresent
                                      # Always，每次都检查；Never，每次都不检查（不管本地是否有）；IfNotPresent，如果本地有就不检查，如果没有就拉取 
        resources: # 资源管理
          limits: # 最大使用
            cpu: 300m # CPU，1核心 = 1000m
            memory: 500Mi # 内存，1G = 1000Mi
          requests:  # 容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行
            cpu: 100m
            memory: 100Mi
        livenessProbe: # pod 内部健康检查的设置
          httpGet: # 通过httpget检查健康，返回200-399之间，则认为容器正常
            path: /healthCheck # URI地址
            port: 8080 # 端口
            scheme: HTTP # 协议
            # host: 127.0.0.1 # 主机地址
          initialDelaySeconds: 30 # 表明第一次检测在容器启动后多长时间后开始
          timeoutSeconds: 5 # 检测的超时时间
          periodSeconds: 30 # 检查间隔时间
          successThreshold: 1 # 成功门槛
          failureThreshold: 5 # 失败门槛，连接失败5次，pod杀掉，重启一个新的pod
        readinessProbe: # Pod 准备服务健康检查设置
          httpGet:
            path: /healthCheck
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 5
          #也可以用这种方法   
          #exec: 执行命令的方法进行监测，如果其退出码不为0，则认为容器正常   
          #  command:   
          #    - cat   
          #    - /tmp/health   
          #也可以用这种方法   
          #tcpSocket: # 通过tcpSocket检查健康  
          #  port: number 
        ports:
          - name: http # 名称
            containerPort: 8080 # 容器开发对外的端口 
            protocol: TCP # 协议
      imagePullSecrets: # 镜像仓库拉取密钥
        - name: harbor-certification
      affinity: # 亲和性调试
        nodeAffinity: # 节点亲和力
          requiredDuringSchedulingIgnoredDuringExecution: # pod 必须部署到满足条件的节点上
            nodeSelectorTerms: # 节点满足任何一个条件就可以
            - matchExpressions: # 有多个选项，则只有同时满足这些逻辑选项的节点才能运行 pod
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
```





## 三、pod.yaml文件

```yaml
# yaml格式的pod定义文件完整内容：
apiVersion: v1       #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork:false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```





## 四、StatefulSet说明

4.1、StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括：

1、稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
2、稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现
3、有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现
4、有序收缩，有序删除（即从N-1到0）
5、有序的滚动更新



StatefulSet由以下几个部分组成：

- Headless Service（无头服务）用于为Pod资源标识符生成可解析的DNS记录。
- volumeClaimTemplates （存储卷申请模板）基于静态或动态PV供给方式为Pod资源提供专有的固定存储。
- StatefulSet，用于管控Pod资源。



为什么要有headless

在deployment中，每一个pod是没有名称，是随机字符串，是无序的。
而statefulset中是要求有序的，每一个pod的名称必须是固定的。当节点挂了，重建之后的标识符是不变的，每一个节点的节点名称是不能改变的。pod名称是作为pod识别的唯一标识符，必须保证其标识符的稳定并且唯一。
为了实现标识符的稳定，这时候就需要一个headless service 解析直达到pod，还需要给pod配置一个唯一的名称。





4.2、创建StatefulSet需要准备的条件

创建顺序如下：
1、Volume
2、Persistent Volume
3、Persistent Volume Claim
4、Service
5、StatefulSet
Volume可以有很多种类型，比如nfs、glusterfs、ceph RBD等

一个完整的StatefulSet控制器由一个Headless Service、一个StatefulSet和一个volumeClaimTemplate组成



4.3、statefulset示例

- 为stateful对象创建创建headless service

  ```yaml
  # cat stateful-svc.yaml 
  apiVersion: v1
  kind: Service
  metadata:
    name: myapp-svc
    labels: 
      app: myapp-svc
  spec:
    ports:
    - name: web
      port: 80
    clusterIP: None
    selector:
  app: myapp-sfs
  ```

  

- 定义创建stateful格式对象

```yaml
# cat statefulset-pod.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  serviceName: myapp-svc
  replicas: 2
  selector:
    matchLabels:
      app: myapp-sfs
  template:
    metadata:
      labels:
        app: myapp-sfs
    spec:
      containers:
      - name: myapp
        image: nginx:1.12
        ports:
        - name: http
          containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: html
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "glusterfs"
      resources:
        requests:
```





## 五、Service yaml文件详解

```yaml
apiVersion: v1
kind: Service
matadata:                                #元数据
  name: string                           #service的名称
  namespace: string                      #命名空间  
  labels:                                #自定义标签属性列表
    - name: string
  annotations:                           #自定义注解属性列表  
    - name: string
spec:                                    #详细描述
  selector: []                           #label selector配置，将选择具有label标签的Pod作为管理 
                                         #范围
  type: string                           #service的类型，指定service的访问方式，默认为 
                                         #clusterIp
  clusterIP: string                      #虚拟服务地址      
  sessionAffinity: string                #是否支持session
  ports:                                 #service需要暴露的端口列表
  - name: string                         #端口名称
    protocol: string                     #端口协议，支持TCP和UDP，默认TCP
    port: int                            #服务监听的端口号
    targetPort: int                      #需要转发到后端Pod的端口号
    nodePort: int                        #当type = NodePort时，指定映射到物理机的端口号
  status:                                #当spce.type=LoadBalancer时，设置外部负载均衡器的地址
    loadBalancer:                        #外部负载均衡器    
      ingress:                           #外部负载均衡器 
        ip: string                       #外部负载均衡器的Ip地址值
        hostname: string                 #外部负载均衡器的主机名
```





## 六、ingress.yaml详解

6.1、说明

通常情况下，service和pod仅可在集群内部网络中通过IP地址访问。所有到达边界路由器的流量或被丢弃或被转发到其他地方
Ingress是授权入站连接到达集群服务的规则集合。

可以给Ingress配置提供外部可访问的URL、负载均衡、SSL、基于名称的虚拟主机等。用户通过POST Ingress资源到API server的方式来请求ingress。 
Ingress controller负责实现Ingress，通常使用负载平衡器，它还可以配置边界路由和其他前端，这有助于以HA方式处理流量。





6.2、示例

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:                                # Ingress spec 中包含配置一个loadbalancer或proxy server
  rules:                             # 的所有信息。最重要的是，它包含了一个匹配所有入站请求的规
  - http:                            # 则列表。目前ingress只支持http规则。
      paths:
      - path: /testpath              # 每条http规则包含以下信息：一个host配置项（比如 
                                     # for.bar.com，在这个例子中默认是*），path列表（比 
                                     # 如：/testpath），每个path都关联一个backend(比如 
                                     # test:80)。在loadbalancer将流量转发到backend之前，所有的 
                                     # 入站请求都要先匹配host和path。
       backend:                        
          serviceName: test          # backend是一个service:port的组合。Ingress的流量被转发到                        
          servicePort: 80            #  它所匹配的backend
```

