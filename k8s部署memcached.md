## 一、创建service

```bash
# cat memcached-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: memcached-svc
  namespace: ns-mysh
spec:
  ports:
    - protocol: TCP 
      port: 11211      #对外暴露的端口
      targetPort: 11211     #pod内的端口
  selector:
    app: mem       #对应pod中的标签
```





## 二、创建pod

```bash
# cat memcached-app.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memcached-app
  namespace: ns-mysh
spec:
  selector:
    matchLabels:
      app: mem
  replicas: 2
  template:
    metadata:
      labels:
        app: mem    #上下对应
    spec:
      containers:
      - name: memcached
        image: memcached:1.4.37
        ports:
        - containerPort: 11211
```



运行结果：

```bash
# kubectl -n ns-mysh get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
memcached-svc   ClusterIP   10.254.150.60   <none>        11211/TCP   13m

# kubectl -n ns-mysh get pod
NAME                             READY   STATUS    RESTARTS   AGE
memcached-app-77c88bc644-n448p   1/1     Running   0          13m
memcached-app-77c88bc644-x6nrn   1/1     Running   0          13m
```







## 三、测试

```bash
# kubectl -n ns-mysh run busybox -it --image=datica/busybox-dig --restart=Never --rm sh
/ # telnet memcached-svc 11211   #说明联通
stats
STAT pid 1
STAT uptime 541
```

