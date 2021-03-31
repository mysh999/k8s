## 一、部署secret

```bash
# cat mysql-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: ns-mysh
type: Opaque
data:
  mysql-root-password: cGFzc3dvcmQ=     #对应密码password
```





## 二、部署mysql svc

```bash
# cat mysql-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  namespace: ns-mysh
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```



```bash
# kubectl -n ns-mysh get svc
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
wordpress-mysql   ClusterIP   None            <none>        3306/TCP    19s
```





## 三、部署mysql app

```bash
# cat mysql-app.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wordpress-mysql
  namespace: ns-mysh
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  serviceName: wordpress-mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
      namespace: ns-mysh
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "ceph-sc"
      resources:
        requests:
          storage: 2Gi
```





```bash
# kubectl -n ns-mysh get po
NAME                             READY   STATUS    RESTARTS   AGE
wordpress-mysql-0            1/1     Running   0          104s
```





## 四、部署wordpress svc

```bash
# cat wordpress-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: ns-mysh
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  clusterIP: None
```



```bash
# kubectl -n ns-mysh get svc
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE

wordpress         ClusterIP   None            <none>        80/TCP      9s
wordpress-mysql   ClusterIP   None            <none>        3306/TCP    118s
```





## 五、部署wordpress app

```bash
# cat wordpress-app.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: wordpress
  namespace: ns-mysh
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  serviceName: wordpress
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
  volumeClaimTemplates:
  - metadata:
      name: wordpress-storage
      namespace: ns-mysh
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "ceph-sc"
      resources:
        requests:
          storage: 2Gi
```



```bash
# kubectl -n ns-mysh get po
NAME                             READY   STATUS    RESTARTS   AGE
wordpress-0                  1/1     Running   0          61s
wordpress-mysql-0            1/1     Running   0          10m

```







## 六、部署wordpress ingress

```bash
# cat wordpress-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: ns-mysh
spec:
  rules:
  - host: wordpress.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: wordpress
          servicePort: 80
```



```bash
# kubectl -n ns-mysh get ingress
NAME                HOSTS              ADDRESS      PORTS   AGE
wordpress-ingress   wordpress.od.com   10.10.2.22   80      38s
```





## 七、客户端访问

配置hosts文件

```bash
10.10.2.22   wordpress.od.com
```

