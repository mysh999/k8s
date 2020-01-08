## 一、目标

- 在K8S集群环境中，部署2台harbor私有仓库，这2台是主从关系，对项目镜像进行同步
- docker可以对私有仓库进行存取镜像
- k8s可以使用私有仓库



## 二、前提条件

1. 单独2台服务器作为harbor使用        192.168.255.100  harbor1        -> 192.168.255.200  harbor2
2. 该服务器要部署docker和docker-compose
3. K8S版本是1.15



## 三、安装harbor

3.1、解压缩离线文件

```bash
# tar -zxvf harbor-offline-installer-v1.10.0.tgz
```



3.2 、修改配置

```bash
# cat ./harbor/harbor.yml
hostname: 192.168.255.100                #harbor ip或者域名
http:                                    #只启用http协议
  port: 80
harbor_admin_password: Harbor12345       #默认密码
database:
  password: root123
  max_idle_conns: 50
  max_open_conns: 100
data_volume: /data                       #镜像存放路径
clair:
  updaters_interval: 12
jobservice:
  max_job_workers: 10
notification:
  webhook_job_max_retry: 10
chart:
  absolute_url: disabled
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
_version: 1.10.0
proxy:
  http_proxy:
  https_proxy:
  no_proxy:
  components:
    - core
    - jobservice
    - clair
```



3.3、执行安装

```bash
# ./install.sh   –-with-clair       #  –-with-clair启用镜像扫描功能

[Step 0]: checking if docker is installed ...

Note: docker version: 19.03.5

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.25.0

[Step 2]: loading Harbor images ...
Loaded image: goharbor/notary-signer-photon:v0.6.1-v1.10.0
Loaded image: goharbor/prepare:v1.10.0
Loaded image: goharbor/harbor-core:v1.10.0
Loaded image: goharbor/harbor-log:v1.10.0
Loaded image: goharbor/registry-photon:v2.7.1-patch-2819-2553-v1.10.0
Loaded image: goharbor/notary-server-photon:v0.6.1-v1.10.0
Loaded image: goharbor/clair-adapter-photon:v1.0.1-v1.10.0
Loaded image: goharbor/chartmuseum-photon:v0.9.0-v1.10.0
Loaded image: goharbor/harbor-db:v1.10.0
Loaded image: goharbor/harbor-registryctl:v1.10.0
Loaded image: goharbor/redis-photon:v1.10.0
Loaded image: goharbor/nginx-photon:v1.10.0
Loaded image: goharbor/harbor-portal:v1.10.0
Loaded image: goharbor/clair-photon:v2.1.1-v1.10.0
Loaded image: goharbor/harbor-jobservice:v1.10.0
Loaded image: goharbor/harbor-migrator:v1.10.0


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /opt/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /secret/keys/secretkey
Generated certificate, key file: /secret/core/private_key.pem, cert file: /secret/registry/root.crt
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir



[Step 5]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating registryctl   ... done
Creating harbor-db     ... done
Creating registry      ... done
Creating harbor-portal ... done
Creating redis         ... done
Creating harbor-core   ... done
Creating nginx             ... done
Creating harbor-jobservice ... done
✔ ----Harbor has been installed and started successfully.----
```



3.4、查看容器

```bash
# docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS                 PORTS                       NAMES
738cd55131f8        goharbor/nginx-photon:v1.10.0                             "nginx -g 'daemon of…"   3 hours ago         Up 3 hours (healthy)   0.0.0.0:80->8080/tcp        nginx
5015bff165ca        goharbor/harbor-jobservice:v1.10.0                        "/harbor/harbor_jobs…"   3 hours ago         Up 3 hours (healthy)                               harbor-jobservice
1894130ba4a9        goharbor/harbor-core:v1.10.0                              "/harbor/harbor_core"    3 hours ago         Up 3 hours (healthy)                               harbor-core
f2c6d8e2084a        goharbor/redis-photon:v1.10.0                             "redis-server /etc/r…"   3 hours ago         Up 3 hours (healthy)   6379/tcp                    redis
f4aeddc6c9de        goharbor/harbor-portal:v1.10.0                            "nginx -g 'daemon of…"   3 hours ago         Up 3 hours (healthy)   8080/tcp                    harbor-portal
0199357d453b        goharbor/registry-photon:v2.7.1-patch-2819-2553-v1.10.0   "/home/harbor/entryp…"   3 hours ago         Up 3 hours (healthy)   5000/tcp                    registry
e48c08aae164        goharbor/harbor-db:v1.10.0                                "/docker-entrypoint.…"   3 hours ago         Up 3 hours (healthy)   5432/tcp                    harbor-db
ac58c0c904b6        goharbor/harbor-registryctl:v1.10.0                       "/home/harbor/start.…"   3 hours ago         Up 3 hours (healthy)                               registryctl
40f451bb6dbe        goharbor/harbor-log:v1.10.0                               "/bin/sh -c /usr/loc…"   3 hours ago         Up 3 hours (healthy)   127.0.0.1:1514->10514/tcp   harbor-log
```



3.5、登陆界面

输入

http://192.168.255.100

按照提示输入用户名admin 密码Harbor12345

![360截图1822122696140117.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gao8wjdhm7j31h90k3t9y.jpg)





## 四、harbor配置

4.1、创建用户

点击“用户管理”-“创建用户”

![360截图176106089699131.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gao93r1dkqj30g10dc749.jpg)



4.2、创建项目

点击“项目管理”-“创建项目”

![360截图17001018507398.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gao95r1t23j30fu09jq2w.jpg)



4.3、给项目分配用户

点击项目名称-成员-用户，分配角色权限

![360截图1685081510010597.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gao995tkdaj30g109zglm.jpg)



## 五、docker客户端配置使用

5.1、客户端配置http免密

```bash
#  添加 --insecure-registry=192.168.255.100
#  vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry=192.168.255.100
```



5.2、重启docker服务

```bash
# systemctl daemon-reload
# systemctl restart docker
```



5.3、从官方pull镜像

```bash
# docker pull centos:latest
```



5.4、登陆私有仓库

```bash
# docker login 192.168.255.100 
Username: mysh
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```



5.5、修改镜像tag

```bash
# docker tag centos:latest 192.168.255.100/dev/centos:latest
```



5.6、push镜像到私有仓库

```bash
# docker push 192.168.255.100/dev/centos:latest

The push refers to repository [192.168.255.100/dev/centos]
9e607bb861a7: Pushed 
latest: digest: sha256:6ab380c5a5acf71c1b6660d645d2cd79cc8ce91b38e0352cbf9561e050427baf size: 529
```



5.7、查看效果

![360截图16530707777495.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gao9zfna82j318l0g2my3.jpg)



5.8、将之前从官方库下载的镜像和tag的镜像删除



5.9、从私有仓库中下载镜像

```bash
# docker pull 192.168.255.100/dev/centos:latest
```



5.10、镜像做新tag

```bash
# docker tag 192.168.255.100/dev/centos:latest centos:7
```



## 六、k8s使用私有仓库

6.1、按照5.1和5.2章节配置docker免密



6.2、创建私有仓库的secret

```bash
# kubectl create secret docker-registry registry-secret --namespace=default \
--docker-server=http://192.168.255.100 --docker-username=mysh \
--docker-password=XXXXXX
```



6.3、配置默认规则,将密钥设置到默认账号中

```bash
# kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-secret"}]}'
```



6.4、查看默认账号配置

```bash
# kubectl get serviceaccounts default -o yaml
apiVersion: v1
imagePullSecrets:
- name: registry-secret
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-09-16T09:29:08Z"
  name: default
  namespace: default
  resourceVersion: "1108873"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 0918c06b-1abe-44e5-8a7f-b10467e99688
secrets:
- name: default-token-bjhtv
```







6.5、部署配置

```bash
# cat nginx-dev.yaml 
apiVersion: v1                       
kind: ReplicationController           
metadata:                             
  name: nginx-dev
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx-dev
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 192.168.255.100/dev/nginx:latest
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: registry-secret             #和上面的默认账号配置registry-secret一致
```





6.6、部署效果

```bash
# kubectl create -f nginx-dev.yaml  

# kubectl get pods -o wide                   
NAME                                   READY   STATUS         RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
nginx-dev-98klw                        1/1     Running        0          18m     10.244.1.118     k8s-node1    <none>           <none>
nginx-dev-tbk9l                        1/1     Running        0          18m     10.244.0.68      k8s-master   <none>           <none>
nginx-dev-wlb97                        1/1     Running        0          18m     10.244.2.84      k8s-node2    <none>           <none>
```





## 七、harbor同步功能部署

7.1、按照第三章节部署第2台harbor服务器，其中3.2章节IP改成对应IP



7.2、在主仓库上新建仓库目标

​        “仓库管理”-“新建目标”

​        ![360截图16800416396058.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gap06y34rmj30gk0i3t8w.jpg)



7.3、同步管理

​		“同步管理”-“新建规则”



​     ![360截图170602218671114.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gap1ik9xfxj30iy0n0aag.jpg)



7.3、手动执行同步

![360截图17860604317466.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gap0aqdn7oj318e0iwjsf.jpg)



![360截图18141222524446.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gap1jei0ygj317o0hpgmm.jpg)



查看目标仓库

![360截图186302175911060.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gap1k1g79nj31hc0i8t9y.jpg)