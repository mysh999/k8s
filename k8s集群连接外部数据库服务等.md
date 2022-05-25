## 一、需求

k8s集群中的pod，如何连接外部的mysql等服务



## 二、原理

最好的方式是采用Endpoint方式(能够看做是将k8s集群以外的服务抽象为内部服务)



## 三、创建步骤

3.1、在一台服务器192.168.0.213上创建mysql

端口使用3306



<span style='color:red'>service与endpoint名称必须相等</span>

3.2、创建mysql endpoint

```bash
# cat mysql-endpoint.yaml 
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-endpoint
  namespace: dev
subsets:
  - addresses:     
    - ip: 192.168.0.213    
    ports:
    - name: mysql
      port: 3306
      protocol: TCP 
```



3.3、创建mysql svc

```bash
# cat mysql-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql-endpoint
  namespace: dev
spec:
  type: ClusterIP
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306
    protocol: TCP
```



3.4、测试连接

```bash
# kubectl run mysql-client -n dev --image=mysql:5.7 -it --rm --restart=Never -- /bin/bash

# mysql -h mysql-endpoint -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 5.7.35-log MySQL Community Server (GPL)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
mysql> 
mysql> 
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mqtt               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.02 sec)
```

连接成功