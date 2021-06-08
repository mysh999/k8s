## 一、需求说明

根据阿良老师的在线课程，通过harbor+jenkins+gitlab部署一套java spring的cicd





## 二、环境说明

| 环境说明       | IP                           | 用户名 |             |
| -------------- | ---------------------------- | ------ | ----------- |
| harbor镜像仓库 | http://10.10.2.34/           | admin  | Harbor12345 |
| gitlab地址     | http://10.10.2.34:9980/      | root   | Harbor12345 |
| jenkins        | k8s http://10.10.2.24:30008/ | admin  | Harbor12345 |
| mysql          | k8s mysql-svc                | root   | password    |

以上环境部署过程略，重点介绍cicd步骤

在mysql中提前初始化sql



## 三、配置步骤

### 3.1、配置configmap

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
              
              
              
 # kubectl -n ns-mysh get configmap
NAME               DATA   AGE

java-demo-config   1      5s
```





### 3.2、配置git

准备java源码目录

```bash
# tree -L 2
.
├── db
│   └── tables_ly_tomcat.sql
├── LICENSE
├── pom.xml
├── README.md
└── src
    └── main

3 directories, 4 files
```



配置git

```bash
# git init
# git remote add origin http://10.10.2.34:9980/root/java-demo.git
# git add .
# git commit -m "Initial commit"
# git push -u origin master
```

确认gitlab上已经有源码文件

![企业微信截图_20210607204203.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gr9zkfe1dbj611e0nkwfk02.jpg)



### 3.3、在jenkins上配置全局凭证

![企业微信截图_20210607204320.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gr9zln9j8uj61ga0py77z02.jpg)

![企业微信截图_20210607204357.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gr9zman5nsj61gy0budgy02.jpg)





### 3.4、在jenkins上配置k8s凭证

![企业微信截图_20210607204527.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gr9znunvb1j61h40pb42n02.jpg)





### 3.5、在gitlab上准备deploy.yaml文件

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-demo
  namespace: ns-mysh
spec:
  replicas: REPLICAS
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
      imagePullSecrets:
      - name: SECRET_NAME
      containers:
      - image: IMAGE_NAME 
        imagePullPolicy: Always
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
---
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
---
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
```





### 3.6、在jenkins上配置流水线

![企业微信截图_20210608102013.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gran7nz82dj60z20m440m02.jpg)



pipeline脚本

```yaml
// 公共
def registry = "10.10.2.34"
// 项目
def project = "demo"
def app_name = "java-demo"
def image_name = "${registry}/${project}/${app_name}:${BUILD_NUMBER}"
def git_address = "http://10.10.2.34:9980/root/java-demo.git"
// 认证
def secret_name = "registry-pull-secret"
def harbor_auth = "44cbeaf3-d7ba-4dbe-a5ea-bf945a2b75ea"
def git_auth = "11f1df7d-4888-44d0-b4eb-54d19ec81ef3"
def k8s_auth = "d67fe54a-997a-4057-b498-3664a2d51467"

pipeline {
  agent {
    kubernetes {
        label "jenkins-slave"
        yaml """
kind: Pod
metadata:
  name: jenkins-slave
spec:
  containers:
  - name: jnlp
    image: "${registry}/library/jenkins-slave-jdk:1.8"
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-sock
        mountPath: /var/run/docker.sock
      - name: maven-cache
        mountPath: /root/.m2
  volumes:
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: maven-cache
      hostPath:
        path: /tmp/m2
"""
        }
      
      }
    parameters {    
        gitParameter branch: '', branchFilter: '.*', defaultValue: 'master', description: '选择发布的分支', name: 'Branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'PT_BRANCH'
        choice (choices: ['1', '3', '5', '7'], description: '副本数', name: 'ReplicaCount')
        choice (choices: ['dev','test','prod','ns-mysh'], description: '命名空间', name: 'Namespace')
    }
    stages {
        stage('拉取代码'){
            steps {
                checkout([$class: 'GitSCM', 
                branches: [[name: "${params.Branch}"]], 
                doGenerateSubmoduleConfigurations: false, 
                extensions: [], submoduleCfg: [], 
                userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]
                ])
            }
        }

        stage('代码编译'){
           steps {
             sh """
                mvn clean package -Dmaven.test.skip=true
                yum -y install unzip
                unzip target/*.war -d target/ROOT
                """ 
           }
        }

        stage('构建镜像'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                  echo '
                    FROM harbor.aliyun.com/java/tomcat:8.5.59
                    RUN rm -rf /usr/local/tomcat/webapps/*
                    COPY target/ROOT /usr/local/tomcat/webapps/ROOT
                  ' > Dockerfile
                  docker build -t ${image_name} .
                  docker login -u ${username} -p '${password}' ${registry}
                  docker push ${image_name}
                """
                }
           } 
        }

        stage('部署到K8S平台'){
          steps {
              configFileProvider([configFile(fileId: "${k8s_auth}", targetLocation: "admin.kubeconfig")]){
                sh """
                  sed -i 's#IMAGE_NAME#${image_name}#' deploy.yaml
                  sed -i 's#SECRET_NAME#${secret_name}#' deploy.yaml
                  sed -i 's#REPLICAS#${ReplicaCount}#' deploy.yaml
                  curl -o /usr/bin/kubectl http://mirrors.aliyun.com/kubernetes/releases/1.17.6/kubectl 
                  chmod u+x /usr/bin/kubectl
                  kubectl apply -f deploy.yaml -n ${Namespace} --kubeconfig=admin.kubeconfig
                """
              }
          }
        }


    }
}


```



执行build

![企业微信截图_20210608102157.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gran9dz6r4j61020gywfw02.jpg)





![企业微信截图_20210608102229.png](http://ww1.sinaimg.cn/large/007Xg1efgy1grana108qyj60jt0gxjs802.jpg)



查看console输出

![企业微信截图_20210608102642.png](http://ww1.sinaimg.cn/large/007Xg1efgy1granegc9vqj60y70qdq4p02.jpg)





### 3.7、查看执行结果

```bash
# kubectl -n ns-mysh get pods|grep -i java
java-demo-7697899f79-kngtr                  1/1     Running   0          92s

# kubectl -n ns-mysh get ingress|grep -i java
java-ingress                   java.od.com            10.10.2.24,10.10.2.25,10.10.2.26   80      2m12s
```



### 3.8、验证

浏览器中输入

```html
http://java.od.com/
```



![企业微信截图_20210608102911.png](http://ww1.sinaimg.cn/large/007Xg1efgy1grangxft45j61460ix4qq02.jpg)



点击添加美女按钮可以添加，说明和后台数据库建立连接

