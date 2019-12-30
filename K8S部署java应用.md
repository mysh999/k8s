参考：

https://www.linuxhub.cn/2019/04/18/deploy-k8s-java.html



## 一、准备java应用

```bash
# 拉取代码
# git clone https://github.com/linuxhub/SpringBootDemo.git
# cd SpringBootDemo/
# git checkout v0.1

# 编译代码
# mvn clean package

# 打包镜像
# docker build --rm --tag "linuxhub_www:v1.0" .

# 运行镜像
# docker run -p 8088:8088/tcp linuxhub_www:v1.0

# 修改tag
# docker tag linuxhub_www:v1.0 linuxhub/linuxhub_www:v1.0
```



## 二、部署java应用到kubernetes集群

2.1、创建deployment

```bash
# cat web-deploy.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: linuxhub-web-deploy
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: linuxhub-web
    spec:
      hostNetwork: true
      containers:
      - name: linuxhub-web
        image: linuxhub/linuxhub_www:v1.0
        ports:
        - containerPort: 8088
```



查看deployment状态

```bash
# kubectl get deployment
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
linuxhub-web-deploy   2/2     2            2           2d23h

# kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
linuxhub-web-deploy-5c6557c579-5j6xb   1/1     Running   1          2d23h   192.168.255.13   k8s-node2    <none>           <none>
linuxhub-web-deploy-5c6557c579-8ctl9   1/1     Running   586        2d23h   192.168.255.11   k8s-master   <none>           <none>


# kubectl get pod --show-labels
NAME                                   READY   STATUS    RESTARTS   AGE     LABELS
linuxhub-web-deploy-5c6557c579-5j6xb   1/1     Running   1          2d23h   app=linuxhub-web,pod-template-hash=5c6557c579
linuxhub-web-deploy-5c6557c579-8ctl9   1/1     Running   586        2d23h   app=linuxhub-web,pod-template-hash=5c6557c579
```



2.2、创建service

```bash
# cat web-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: linuxhub-web-svc
  labels:
    app: linuxhub-web
spec:
  type: ClusterIP
  selector:
    app: linuxhub-web
  ports:
  - name: http
    port: 8088
    targetPort: 8088
```



查看service:

```bash
# kubectl get service -o wide
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP          104d    <none>
linuxhub-web-svc   ClusterIP      10.105.156.131   <none>        8088/TCP         2d23h   app=linuxhub-web


# kubectl describe  service linuxhub-web-svc
Name:              linuxhub-web-svc
Namespace:         default
Labels:            app=linuxhub-web
Annotations:       <none>
Selector:          app=linuxhub-web
Type:              ClusterIP
IP:                10.105.156.131
Port:              http  8088/TCP
TargetPort:        8088/TCP
Endpoints:         192.168.255.11:8088,192.168.255.13:8088
Session Affinity:  None
Events:            <none>
```





2.3、创建Ingress

```bash
# cat web-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: linuxhub-web-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: demo.linuxhub.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: linuxhub-web-svc
          servicePort: 8088
```



查看ingress

```bash
# kubectl get ingress
NAME                   HOSTS              ADDRESS   PORTS   AGE
linuxhub-web-ingress   demo.linuxhub.cn             80      2d23h
```



三、访问

3.1、配置客户端hosts

```bash
192.168.255.11 demo.linuxhub.cn
```

3.2、访问

http://192.168.255.11:8088/index或者http://192.168.255.11:8088/hello

![360截图16770803626292.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gaerfkc8wgj30ez0443ye.jpg)