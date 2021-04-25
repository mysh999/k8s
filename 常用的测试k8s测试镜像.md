- 第一种镜像： mydlqclub/dnsutils:1.3

```bash
# cat ndsutils.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: ns-mysh
spec:
  containers:
  - name: dnsutils
    image: mydlqclub/dnsutils:1.3
    imagePullPolicy: IfNotPresent
    command: ["sleep","3600"]
```

测试过程：

```bash
# kubectl -n ns-mysh exec -it dnsutils -- sh
/ # nslookup mysql-svc     #同命名空间对svc解析
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   mysql-svc.ns-mysh.svc.cluster.local
Address: 10.103.252.234

/ # nslookup nginx-web-svc.ns-dev    #跨命名空间对svc解析
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx-web-svc.ns-dev.svc.cluster.local
Address: 10.108.58.77
```







- 第二种镜像   busybox:1.28.4

  ```bash
  # cat busybox.yaml 
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: buxybox
    namespace: ns-mysh
    labels:
      app: busybox
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: busybox
    template:
      metadata:
        labels:
          app: busybox
      spec:
        containers:
        - name: busybox 
          image: busybox:1.28.4
          command:
            - sleep
            - "3600"
          ports:
          - containerPort: 80
  ```

  

测试过程类似







- 第三种镜像：radial/busyboxplus

  ```bash
  # cat busyboxplus.yaml 
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: buxyboxplus
    namespace: ns-mysh
    labels:
      app: busyboxplus
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: busyboxplus
    template:
      metadata:
        labels:
          app: busyboxplus
      spec:
        containers:
        - name: busyboxplus 
          image: radial/busyboxplus
          command:
            - sleep
            - "3600"
          ports:
          - containerPort: 80
  ```

  带curl命令，其他类似