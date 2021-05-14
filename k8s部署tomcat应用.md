## 一、案例说明

本案例是根据阿良老师的线上课程来部署的

已提前准备了mysql数据库环境，数据库如下：

| mysql service名称 | 用户名 | 密码     |
| ----------------- | ------ | -------- |
| mysql-svc         | root   | password |

同时也做了数据库初始化，sql语句如下：

```sql
# cat tables_ly_tomcat.sql 
/*
MySQL - 5.6.30-log : Database - test
*/
CREATE DATABASE IF NOT EXISTS `test`  DEFAULT CHARACTER SET utf8 ;
USE `test`;

CREATE TABLE `user` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL COMMENT '名字',
  `age` INT(3) NOT NULL COMMENT '年龄',
  `sex` CHAR(1) DEFAULT NULL COMMENT '性别',
  PRIMARY KEY (`id`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;

COMMIT;
```





## 二、部署过程

未特别说明，均在K8S master节点执行

2.1、安装包

```bash
# yum install java-1.8.0-openjdk maven git -y 
# git clone https://github.com/lizhenliang/tomcat-java-demo 
```



2.2、代码编译构建

```bash
# vi /etc/maven/settings.xml  #改成阿里镜像
在<mirrors></mirrors>标签中添加 mirror 子节点:

<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>阿里云公共仓库</name>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>


# mvn clean package -Dmaven.test.skip=true

# 解压缩
# unzip target/*.war -d target/ROOT
```



2.3、制作镜像

```bash
# cat Dockerfile 
FROM tomcat
COPY target/ROOT /usr/local/tomcat/webapps/ROOT


# docker build -t tomcat:v1 .

# docker tag tomcat:v1 java-demo:v1
```

将镜像导入到其他计算节点或者push到镜像仓库



2.4、创建configmap

```bash
# cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-demo-config
  namespace: ns-mysh
data:
    application.yml: |
        server:
          port: 8080
        spring:
          datasource:
            url: jdbc:mysql://mysql-svc:3306/test?characterEncoding=utf-8
            username: root
            password: password
            driver-class-name: com.mysql.jdbc.Driver
          freemarker:
            allow-request-override: false
            cache: true
            check-template-location: true
            charset: UTF-8
            content-type: text/html; charset=utf-8
            expose-request-attributes: false
            expose-session-attributes: false
            expose-spring-macro-helpers: false
            suffix: .ftl
            template-loader-path:
              - classpath:/templates/
              
          
  # kubectl -n ns-mysh get configmap|grep -i java
java-demo-config   1      67m
```



2.5、部署tomcat

```bash
# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-demo
  namespace: ns-mysh
spec:
  replicas: 1
  selector:
    matchLabels:
      project: www
      app: java-demo
  template:
    metadata:
      labels:
        project: www
        app: java-demo
    spec:
      containers:
      - image: java-demo:v1 
        name: java-demo
        volumeMounts:
        - name: config
          mountPath: "/usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application.yml"
          subPath: application.yml
        resources:
          requests:
            cpu: 0.5
            memory: 500Mi
          limits: 
            cpu: 1
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 50
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 50
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: java-demo-config 
          items:
          - key: "application.yml"
            path: "application.yml"
            
            
            
   # kubectl -n ns-mysh get pods|grep -i java
java-demo-75bb45fcb8-vgfxv                 1/1     Running   0          66m
```





2.6、创建svc

```bash
# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: java-demo-svc
  namespace: ns-mysh
spec:
  selector:
    project: www
    app: java-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080 
      
# kubectl -n ns-mysh get svc|grep -i java
java-demo-svc              ClusterIP   10.101.119.190   <none>        80/TCP                                                   64m


# curl 10.101.119.190   #可以打开
```





2.7、创建ingress

```bash
# cat java-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: java-ingress
  namespace: ns-mysh
spec:
  rules:
  - host: java.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: java-demo-svc
          servicePort: 8080
          
          
 # kubectl -n ns-mysh get ingress|grep -i java
java-ingress                   <none>   java.od.com                      80        66m
```

浏览器打开

```html
http://java.od.com
```

