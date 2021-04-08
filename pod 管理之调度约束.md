# Pod 管理之调度约束
[toc]

## 调度过程

`kubernetes`是容器编排引擎,基于`kube-scheduler`实现容器的完全自动化调度.

整体流程如下：
![](https://img-blog.csdnimg.cn/20200626150015541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE3MjU2MDM=,size_16,color_FFFFFF,t_70)

`kube-scheduler`部分流程如下:

![](https://img-blog.csdnimg.cn/20200626150041213.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE3MjU2MDM=,size_16,color_FFFFFF,t_70)

调度周期分为:

- `调度周期Scheduling Cycle`
    - `过滤filter`: 按照指定的调度策略将满足`Pod`节点运行条件的`node`筛选出来.
    - `权重weight`: 
- `绑定周期Binding Cycle`: `kubelet`通过`watch`到新的`Pod`被调度到本节点上,执行`bind`操作


![](https://img-blog.csdnimg.cn/20200626150253266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE3MjU2MDM=,size_16,color_FFFFFF,t_70)

- Queue sort：用于对scheduelr优先级队列进行排序，需要实现"less(pod1, pod2)"接口，且该插件只会生效一个
- Pre-filter：用于检查集群和pod需要满足的条件，或者对pod进行预选 预处理，需要实现"PreFilter"接口
- Filter：对应scheduler预选算法，用于根据预选策略对节点进行过滤
- Pre-Score：对应"Pre-filter"，主要用于优选 预处理，比如：更新cache，产生logs/metrics等
- Scoring：对应scheduler优选算法，分为"score"(Map)和"normalize scoring"(Reduce)两个阶段
- score：并发执行node打分；同一个node在打分的时候，顺序执行插件对该node进行score
- normalize scoring：并发执行所有插件的normalize scoring；每个插件对所有节点score进行reduce，最终将分数限制在[MinNodeScore, MaxNodeScore]有效范围
- Reserve(aka Assume)：scheduling cycle的最后一步，用于将node相关资源预留(assume)给pod，更新scheduler cache；若binding cycle执行失败，则会执行对应的Un-reserve插件，清理掉与pod相关的assume资源，并进入scheduling queue等待重新调度
- Permit：binding cycle的第一个步骤，判断是否允许pod与node执行bind，有如下三种行为：
- approve：允许，进入Pre-bind流程
- deny：不允许，执行Un-reserve插件，并进入scheduling queue等待重新调度
- wait (with a timeout)：pod将一直持续处于Permit阶段，直到approve，进入Pre-bind；如果超时，则会被deny，等待重新被调度
- Pre-bind：执行bind操作之前的准备工作，例如volume相关的操作
- Bind：用于执行pod与node之间的绑定操作，只有在所有pre-bind plugins相关操作都完成的情况下才会被执行；另外，如果一个bind插件选择处理pod，那么其它bind插件都会被忽略
- Post-bind：binding

`cycle`最后一个步骤，用于在bind操作执行成功后清理相关资源


## 初级调度
### 基于节点名称

通过配置`spec.nodeName`方式指定Pod的运行节点.

1. 首先通过`kubectl describe nodes `查看预部署节点的`Labels`中的`kubernetes.io/hostname`的节点名称
2. 在Pod的配置文件中设置`spec.nodeName`值为节点名称

如上.

示例配置:

```yaml
[root@ubtcloud-1 yaml]# cat nginx_by_hostname.yml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx-by-hostname
  labels:
    app: nginx
spec:
  nodeName: ubtcloud-6
  containers:
    - name: nginx
      image: harbor.ubtrobot.com/nginx/nginx:1.15.1.0
```

结果验证:

```bash
[root@ubtcloud-1 yaml]# kubectl get pods -o wide
NAME                    READY   STATUS      RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
nginx-by-hostname       1/1     Running     0          37s     192.168.43.22   ubtcloud-6   <none>           <none>
```

### 基于节点标签

通过配置`spec.nodeSelector`方式指定Pod的运行节点.

1. 首先通过`kubectl label nodes <nodename> <label>`方式给预部署节点打算特定标签
2. 在Pod的配置文件中设置`spec.nodeSelector`值为节点特定标签


<font style='color:red'>注意`nodeSelector`的标签格式是: `<label_key>: <label_value>`和`kubectl describe nodes`中的`<label_key>=<label_value>`不一样哦</font>

示例配置:

```yaml
[root@ubtcloud-1 yaml]# cat nginx_by_nodelabel.yml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx-by-nodelabel
  labels:
    app: nginx
spec:
  nodeSelector:
    environment: devopt
  containers:
    - name: nginx
      image: harbor.ubtrobot.com/nginx/nginx:1.15.1.0
```

结果验证:
```bash
[root@ubtcloud-1 yaml]# kubectl get pods -o wide
NAME                    READY   STATUS      RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
nginx-by-nodelabel      1/1     Running     0          10s     192.168.43.23   ubtcloud-6   <none>           <none>
```

## 高级调度

### 节点亲和性

使用帮助: `kubectl explain deploy.spec.template.spec.affinity.nodeAffinity`

节点亲和性`nodeAffinity`和`nodeSelector`功能类似都是**基于`node`已拥有的`label`作为约束**,但节点亲和性`nodeAffinity`支持更多的匹配方式: `In`,`NotIn`,`Exists`,`DoesNotExist`,`Gt`,`Lt`.

In: label的值在某个列表中
NotIn：label的值不在某个列表中
Exists：某个label存在
DoesNotExist：某个label不存在
Gt：label的值大于某个值（字符串比较）
Lt：label的值小于某个值（字符串比较）

规则为: **如果节点满足一个或者多个规则,则新的Pod将会(优先)调度到该节点**

节点亲和性`nodeAffinity`提供4种指标,分别指定了调度器对`Pod`调度期间(`DuringScheduling`)和运行期间(`DuringExecution`)亲和性执行依据:

|指标|描述|
|:-|:-|
|`requiredDuringSchedulingIgnoredDuringExecution`|硬亲和性,必须满足不然无法调度.如果节点的标签在运行时发生变化导致亲和性策略不能满足,不会影响到正在运行的`Pod`.|
|`preferredDuringSchedulingIgnoredDuringExecution`|软亲和性,优先条件能满足最好不满足也不影响调度.如果节点的标签在运行时发生变化导致亲和性策略不能满足,不会影响到正在运行的`Pod`.|
|`requiredDuringSchedulingRequiredDuringExecution`|超硬亲和性(暂不支持),必须满足不然无法调度.如果节点的标签在运行时发生变化导致亲和性策略不能满足,则将`Pod`重新调度到满足条件的节点.|
|`preferredDuringSchedulingRequiredDuringExecution`|硬软亲和性(暂不支持),优先条件能满足最好不满足也不影响调度.如果节点的标签在运行时发生变化导致亲和性策略不能满足,则将`Pod`重新调度到满足条件的节点.|

- `requiredDuringSchedulingIgnoredDuringExecution`: 硬亲和性,必须满足不然无法调度.

```yaml
requiredDuringSchedulingIgnoreDuringExecution： 
## 硬亲和性；节点必须满足设定条件，Pod才能被调度到这个节点。
  nodeSelectorTerms： 
  ## 节点选择器列表
  - matchExpressions：
  ## 按节点标签列出的节点选择器要求列表
    - key：     
    ## 键
      operator：
      ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
      values：  
      ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
    matchFields：
    ## 按节点字段列出的节点选择器要求列表
    - key：     
    ## 键
      operator：
      ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
      values：  
      ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
```

- `preferredDuringSchedulingIgnoredDuringExecution`: 软亲和性,优先条件能满足最好不满足也不影响调度.

```yaml
preferredDuringSchedulingIgnoreDuringExecution：
## 软亲和性；不管节点上能否满足设定条件，Pod都可以被调度，只是Pod优先被调度到符合条件多的节点上。
- preference：
## 与相应权重相关联的节点选择器项。
    matchExpressions：## 按节点标签列出的节点选择器要求列表
    - key：     
    ## 键
      operator：
      ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
      values：  
      ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
    matchFields：
    ## 按节点字段列出的节点选择器要求列表
    - key：     
    ## 键
      operator：
      ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
      values：  
      ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
  weight：      
  ## 权重，0~100的数值
```


示例:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.ubtechinc-authorization-server: 1.0.0
  name: ubtechinc-authorization-server
  namespace: dev
spec:
  # 副本数
  replicas: 2
  # 版本变更记录--record记录
  revisionHistoryLimit: 10
  # Pod标签选择
  selector:
    matchLabels:
      app: ubtechinc-authorization-server
  # 调度失败超时时间
  progressDeadlineSeconds: 120
  # Pod ready状态等待时间
  minReadySeconds: 0
  # Pod模板
  template:
    metadata:
      labels:
        app: ubtechinc-authorization-server
    spec:
      # 重启策略
      restartPolicy: Always
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - noder-031
                - noder-032
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: worker-node
                operator: In
                values:
                - 'true'
```

上述资源清单中,
1. `Pod`必须调度到拥有`kubernetes.io/hostname`标签的值为`noder-031`或者`noder-032`上面,不满足将无法调度.
2. 在第一步过滤出来的节点中优先调度到拥有`worker-node`标签的值为`true`的节点上.如果没有也没关系

> 如果`nodeAffinity`和`nodeSelector`同时存在,则二者必须都要满足才行.


### Pod亲和性和反亲和性

`Pod`亲和性`podAffinity`和反亲和性`podAntiAffinity`都是**基于已在节点上运行的`Pod`的`label`作为约束**,而不是基于节点`label`.

与节点亲和性的区别有:

1. `Pod`拥有命名空间属性,所有必须通过`namespace`指定命名空间,如为空则为`all`.
2. 增加`topologyKey`(拓扑)参数,用于选择拥有特定`label`的`Node`的范围,通常用来划定一个机柜标签,也可以是机房标签的`node`
.

规则为: **Pod`亲和性`podAffinity`: 如果节点上已经运行的所有`Pod`满足一个或者多个规则,则新的`Pod`将会(优先)调度到该节点.反亲和性`podAntiAffinity`反之不会(尽量避免)调度到该节点**

`Pod`亲和性`podAffinity`和反亲和性`podAntiAffinity`均提供2种指标,分别指定了调度器对`Pod`调度期间(`DuringScheduling`)和运行期间(`DuringExecution`)亲和性执行依据:


- `requiredDuringSchedulingIgnoredDuringExecution`: 硬亲和性,必须满足不然无法调度.

```yaml
requiredDuringSchedulingIgnoreDuringExecution： 
## 硬亲和性；必须满足设定条件。
- labelSelector： 
## 根据标签选定一组pod作为亲和对象。
    matchExpressions：
    ## 标签选择器要求列表
    - key：     
    ## 键
      operator：
      ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
      values：  
      ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
    matchLabels：
    ## 标签选择器要求列表
  namespaces:   
  ## 指明选定的pod亲和对象是哪组名称空间下的，不指定则为第一个pod所运行的命名空间下。
  topologyKey:  
  ## 位置拓扑键，用来判定拥有哪些标签的节点是同一位置。
```

- `preferredDuringSchedulingIgnoredDuringExecution`: 软亲和性,优先条件能满足最好不满足也不影响调度.

```yaml
preferredDuringSchedulingIgnoreDuringExecution：
## 软亲和性；不管节点上能否满足设定条件，Pod都可以被调度，只是Pod优先被调度到符合条件多的节点上。
  podAffinityTerm： 
  ## 定义一组pods（即与相对于此pod应位于（关联）或不在同一地点（反亲和力），在同一地点被定义为运行在一个节点上，其键<topologykey>的标签值与一组pods运行的任何节点
    labelSelector： 
    ## 根据标签选定一组pod作为亲和对象。
      matchExpressions：
      ## 标签选择器要求列表
      - key：     
      ## 键
        operator：
        ## 表示键与一组值的关系。有效的运算符有：In、NotIn、Exist、DoesNotExsit。GT和LT
        values：  
        ## 值；若operator为In或NotIn则值必须为非空；若operator为Exists或DoesNotExist则值必须为空；若operator为Gt或Lt则值必须有一个元素。
      matchLabels：
      ## 标签选择器要求列表
    namespaces：  
    ## 指明选定的pod亲和对象是哪组名称空间下的，不指定则为第一个pod所运行的命名空间下。
    topologyKey： 
    ## 位置拓扑键，用来判定拥有哪些标签的节点是同一位置。
  weight：        
  ## 权重，0~100的数值
```

示例:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.ubtechinc-authorization-server: 1.0.0
  name: ubtechinc-authorization-server
  namespace: dev
spec:
  # 副本数
  replicas: 2
  # 版本变更记录--record记录
  revisionHistoryLimit: 10
  # Pod标签选择
  selector:
    matchLabels:
      app: ubtechinc-authorization-server
  # 调度失败超时时间
  progressDeadlineSeconds: 120
  # Pod ready状态等待时间
  minReadySeconds: 0
  # Pod模板
  template:
    metadata:
      labels:
        app: ubtechinc-authorization-server
    spec:
      # 重启策略
      restartPolicy: Always
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            labelSelector:
            - matchExpressions:
              - key: team
                operator: In
                values:
                - asr
            namespaces: asr-team
            topologyKey: guiyang.kubernetes.io/zone
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                - key: worker-type
                  operator: In
                  values:
                  - gpu
              namespaces: asr-team
              topologyKey: guiyang.kubernetes.io/zone
```

上述资源清单中,

1. `Pod`必须调度到`node`标签为`guiyang.kubernetes.io/zone`,且节点存在`Pod`归属`namespace`为`asr-team`拥有标签`team:asr`.
2. `Pod`尽量避免调度到`node`标签为`guiyang.kubernetes.io/zone`,且节点存在`Pod`归属`namespace`为`asr-team`拥有标签`worker-type:gpu`.

原则上`topologyKey`可以是节点的合法标签,但是有一些约束:
- 对于亲和性以及`RequiredDuringScheduling`的反亲和性,`topologyKey`需要指定
- 对于`RequiredDuringScheduling`的反亲和性,`LimitPodHardAntiAffinityTopology`的准入控制限制`topologyKey`为`kubernetes.io/hostname`,可以通过修改或者`disable`解除该约束
- 对于`PreferredDuringScheduling`的反亲和性,空的`topologyKey`表示`kubernetes.io/hostname`,`failure-domain.beta.kubernetes.io/zone` ,`failure-domain.beta.kubernetes.io/region`的组合．
- `topologyKey`在遵循其他约束的基础上可以设置成其他的key.
 

### 容忍与污点

使用`kubectl taint`命令可以给某个`Node` 节点设置污点,`Node`被设置上污点之后就和`Pod` 之间存在了一种相斥的关系,可以让`Node`拒绝`Pod`的调度执行,甚至将`Node`已经存在的`Pod`驱逐出去.每个污点的组成如下：`key=value:effect`

每个污点有一个`key`和`value`作为污点的标签,其中`value`可以为空,`effect`描述污点的作用.当前`taint effect`支持如下三个选项:

- `NoSchedule`: 表示`k8s`将不会将`Pod`调度到具有该污点的`Node`上.不会影响到已经运行的`Pod`
- `PreferNoSchedule`: 表示`k8s`将尽量避免将`Pod`调度到具有该污点的`Node`上.不会影响到已经运行的`Pod`
- `NoExecute`: 表示`k8s`将不会将`Pod`调度到具有该污点的`Node` 上,同时会将`Node`上已经存在的`Pod`驱逐出去.
