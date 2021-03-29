参考地址：

https://mp.weixin.qq.com/s?__biz=MzAwNTM5Njk3Mw==&mid=2247499073&idx=1&sn=51306a358423adcc4c288f9c8d834982&chksm=9b1ffdc3ac6874d5a3354900769f849c21b45499973e9a42284b3e60dfcc539a2586fe6b8291&mpshare=1&scene=1&srcid=0326eibpcm4xcR9uVZnCcDiJ&sharer_sharetime=1616747393567&sharer_shareid=13b11b6cd09cc058dfc7408fed81afbb&version=3.1.2.2211&platform=win#rd

https://www.mdeditor.tw/pl/peMa/zh-tw

https://nc2era.com/de774324.html

https://my.oschina.net/u/4353890/blog/4705327



## 一、背景说明

​              在K8S中部署一套mysql主从集群，后台采用的ceph存储

​              mysql采用主从复制+读写分离方式，由单一的master和多个slave组成。

​              通过mysql+xtrabackup模式组成主从架构



![企业微信截图_20210329155519.png](http://ww1.sinaimg.cn/large/007Xg1efgy1gp0twlujcwj30gy0bnq48.jpg)







## 二、部署步骤

### 2.1、创建命名空间

```bash
# kubectl create ns blog
```





### 2.2、创建configmap

configmap可以将配置文件和镜像解耦

创建一个master.cnf文件配置内容为：log-bin，即开启bin-log日志，供主节点使用。
创建一个slave.cnf文件配置内容为：super-read-only，设为该节点只读，供备用节点使用

```bash
# cat mysql-cluster-configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  namespace: blog
  labels:
    app: mysql
data:
  master.cnf: |
    # Master配置
    [mysqld]
    log-bin=mysqllog
    skip-name-resolve
  slave.cnf: |
    # Slave配置
    [mysqld]
    super-read-only
    skip-name-resolve
    log-bin=mysql-bin
    replicate-ignore-db=mysql
```





### 2.3、创建mysql密码

```bash
# cat mysql-cluster-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: blog
  labels:
    app: mysql
type: Opaque
data:
  password: MTIzNDU2 # echo -n "123456" | base64
```







### 2.3、创建svc

创建一个服务名为mysql的headless类型的service。
创建一个服务名为mysql-read的service

```bash
# cat mysql-cluster-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: blog
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: blog
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```



### 2.4、创建主从集群

```bash
# cat mysql-cluster-app.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: blog
  labels:
    app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 2
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 从Pod的序号，生成server-id
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 由于server-id不能为0，因此给ID加100来避开它
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果Pod的序号为0，说明它是Master节点，从ConfigMap里把Master的配置文件拷贝到/mnt/conf.d目录下
          # 否则，拷贝ConfigMap里的Slave的配置文件
          if [[ ${ordinal} -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以数据已经存在则跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Master 节点（序号为 0）不需要这个操作
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal == 0 ]] && exit 0
          # 使用ncat指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
 #        - name: MYSQL_ALLOW_EMPTY_PASSWORD
 #          value: "1"
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          # 从备份信息文件里读取MASTER_LOG_FILE和MASTER_LOG_POS这2个字段的值，用来拼装集群初始化SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果xtrabackup_slave_info文件存在，说明这个备份数据来自于另一个Slave节点
            # 这种情况下，XtraBackup工具在备份的时候，就已经在这个文件里自动生成了“CHANGE MASTER TO”SQL语句
            # 所以，只需要把这个文件重命名为change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着xtrabackup_binlog_info了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只是存在xtrabackup_binlog_info文件，说明备份来自于Master节点，就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ $(cat xtrabackup_binlog_info) =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成SQL，写入change_master_to.sql.in文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          # 如果存在change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等MySQL容器启动之后才能进行下一步连接MySQL的操作
            echo "Waiting for mysqld to be ready（accepting connections）"
            until mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            # 将文件change_master_to.sql.in改个名字
            # 防止这个Container重启的时候，因为又找到了change_master_to.sql.in，从而重复执行一遍初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用change_master_to.sql.orig的内容，也就是前面拼装的SQL，组成一个完整的初始化和启动Slave的SQL语句
            mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} << EOF
          $(< change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          # 使用ncat监听3307端口。
          # 它的作用是，在收到传输请求的时候，直接执行xtrabackup --backup命令，备份MySQL的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql             
  volumeClaimTemplates:
  - metadata:
      name: data
      namespace: blog
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "ceph-sc"
      resources:
        requests:
          storage: 3Gi
```



```bash
# kubectl -n blog get pods
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   2/2     Running   0          7m58s
mysql-1   2/2     Running   0          7m39s
```





### 2.5、服务验证 

```bash
# kubectl -n blog exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show slave status \G'"
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-0.mysql
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysqllog.000003
          Read_Master_Log_Pos: 154
               Relay_Log_File: mysql-1-relay-bin.000002
                Relay_Log_Pos: 319
        Relay_Master_Log_File: mysqllog.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 528
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 100
                  Master_UUID: f76dded0-9065-11eb-ab10-8ed87d9fd957
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
```





在主库上新建库和插入表

```bash
kubectl -n blog exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'create database test'"
kubectl -n blog exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;create table counter(c int);'"
kubectl -n blog exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;insert into counter values(123)'"
```



从库验证 

```bash
# kubectl -n blog exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;select * from counter'"  
mysql: [Warning] Using a password on the command line interface can be insecure.
c
123
```





### 2.6、扩展从节点

```bash
# kubectl -n blog scale statefulset mysql --replicas=3
statefulset.apps/mysql scaled
# kubectl -n blog get pods            
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   2/2     Running   0          57m
mysql-1   2/2     Running   0          57m
mysql-2   2/2     Running   0          34m
```

验证

```bash
# kubectl -n blog exec mysql-2 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;select * from counter'"
mysql: [Warning] Using a password on the command line interface can be insecure.
c
123
```



### 2.7、测试svc

```bash
# kubectl -n blog get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
mysql        ClusterIP   None             <none>        3306/TCP   3h25m   app=mysql
mysql-read   ClusterIP   10.254.167.227   <none>        3306/TCP   3h25m   app=mysql


# kubectl -n blog run busybox -it --image=datica/busybox-dig --restart=Never --rm sh
/ # nslookup mysql
Server:    10.254.0.2
Address 1: 10.254.0.2 kube-dns.kube-system.svc.cluster.local

Name:      mysql
Address 1: 172.31.123.187 172-31-123-187.mysql-read.blog.svc.cluster.local
Address 2: 172.31.123.188 mysql-0.mysql.blog.svc.cluster.local
Address 3: 172.31.242.230 mysql-2.mysql.blog.svc.cluster.local
/ # nslookup mysql-read
Server:    10.254.0.2
Address 1: 10.254.0.2 kube-dns.kube-system.svc.cluster.local

Name:      mysql-read
Address 1: 10.254.167.227 mysql-read.blog.svc.cluster.local
```

