## 一、部署svc

```bash
# cat nginx-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: ns-mysh
spec:
  ports:
    - protocol: TCP 
      port: 80
      targetPort: 80
  selector:
    app: nginx
    
    
    
# kubectl -n ns-mysh get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
nginx-svc       ClusterIP   10.254.64.105   <none>        80/TCP      15m
```





## 二、部署pod

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
        
        
# kubectl -n ns-mysh get po
NAME                             READY   STATUS    RESTARTS   AGE
nginx-app-64c655b587-rm9lt       1/1     Running   0          16m
nginx-app-64c655b587-z8bnh       1/1     Running   0          16m
```



## 三、测试接口

```bash
# kubectl -n ns-mysh run busybox -it --image=radial/busyboxplus --restart=Never --rm sh  
/ # curl nginx-svc
  #可返回nginx界面
```





## 四、创建ingress

```bash
# cat nginx-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: ns-mysh
spec:
  rules:
  - host: nginx.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-svc
          servicePort: 80
          
          
          
 # kubectl -n ns-mysh get ingress
NAME            HOSTS          ADDRESS      PORTS   AGE
nginx-ingress   nginx.od.com   10.10.2.22   80      3m28s



# kubectl -n ns-mysh describe ingress nginx-ingress
Name:             nginx-ingress
Namespace:        ns-mysh
Address:          10.10.2.22
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host          Path  Backends
  ----          ----  --------
  nginx.od.com  
                /   nginx-svc:80 (172.31.123.168:80,172.31.242.240:80)
Annotations:
Events:
  Type    Reason  Age    From                      Message
  ----    ------  ----   ----                      -------
  Normal  CREATE  3m54s  nginx-ingress-controller  Ingress ns-mysh/nginx-ingress
  Normal  UPDATE  3m11s  nginx-ingress-controller  Ingress ns-mysh/nginx-ingress
```



## 五、访问ingress

客户端配置hosts文件

``` bash
10.10.2.22 nginx.od.com
```

再在浏览器中输入

http://nginx.od.com

可以打开nginx页面