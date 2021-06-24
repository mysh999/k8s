## 一、需求

在K8S中，分别对dev命名空间创建具有只读和读写权限的用户

| 用户名     | 权限 | 所在namespace |
| ---------- | ---- | ------------- |
| reader_dev | 只读 | dev           |
| admin_dev  | 管理 | dev           |
|            |      |               |





## 二、创建只读用户

### 2.1、创建用户私钥

```bash
# mkdir -p /opt/reader_dev
# cd /opt/reader_dev/
# openssl genrsa -out reader_dev.key 2048  
```



### 2.2、创建用户公钥

```bash
# openssl req -new -key reader_dev.key -out reader_dev.csr -subj "/CN=reader_dev/O=ubtdev"  
```



### 2.3、 创建用户证书

有效期365天

```bash
# openssl x509 -req -in reader_dev.csr -CA /etc/kubernetes/pki/ca.pem -CAkey /etc/kubernetes/pki/ca-key.pem  -CAcreateserial -out reader_dev.crt -days 365  
```



### 2.4、创建用户凭据

```bash
# kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://10.10.2.24:6443 --kubeconfig=reader_dev.config

# kubectl config set-credentials reader_dev --client-certificate=reader_dev.crt --client-key=reader_dev.key --embed-certs=true --kubeconfig=reader_dev.config

# kubectl config set-context kubernetes --cluster=kubernetes --namespace=dev --user=reader_dev --kubeconfig=reader_dev.config

# kubectl config use-context kubernetes --kubeconfig=reader_dev.config
```



### 2.5、创建role

```bash
# cat role.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reader-dev
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: reader-dev-roles
  namespace: dev
rules:
- apiGroups:
  - ""
  resources:
  - limitranges
  - limitranges/status
  - namespaces
  - namespaces/status
  - resourcequotas
  - resourcequotas/status
  - pods
  - pods/status
  - pods/log
  - ingresses
  - ingresses/status
  - endpoints
  - configmaps
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - roles
  - services
  - secrets
  - events
  verbs:
  - get
  - list
  - watch
---

```



### 2.6、创建绑定role

```bash
# cat bind_role.yaml 
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: reader-dev-binding
  namespace: dev
subjects:
- kind: User
  name: reader_dev
  apiGroup: ""
roleRef:
  kind: Role
  name: reader-dev-roles
  apiGroup: ""
```



### 2.7、验证

```bash
# export KUBECONFIG=/opt/reader_dev/reader_dev.config

# kubectl get nodes               
Error from server (Forbidden): nodes is forbidden: User "reader_dev" cannot list resource "nodes" in API group "" at the cluster scope

# kubectl -n dev get pods         
NAME                                 READY   STATUS    RESTARTS   AGE
dubbo-bc7b7c48b-khxgg                1/1     Running   2          118d
elasticsearch-0                      1/1     Running   2          175d
elasticsearch-1                      1/1     Running   1          48d
elasticsearch-2                      1/1     Running   2          175d
interaction-skill-67b45ddffd-lfm7x   1/1     Running   7          177d
jumpserver-699b56b9f5-h55vk          1/1     Running   7          240d
kubernetes-cicd-8d5c8f9b5-7vslr      1/1     Running   3          161d
mongo-0                              1/1     Running   3          181d
mysql-0                              1/1     Running   1          48d
redis-0                              1/1     Running   3          170d
rocketmq-0                           1/1     Running   1          52d
rocketmq-1                           1/1     Running   1          52d
rocketmq-2                           1/1     Running   1          48d
rocketmq-3                           1/1     Running   2          101d
rocketmq-console-767bb6c545-4cjzv    1/1     Running   1          48d
rocketmq-srv-0                       1/1     Running   1          52d
rocketmq-srv-1                       1/1     Running   1          52d
rocketmq-srv-2                       1/1     Running   1          48d
zk-0                                 1/1     Running   2          175d
zk-1                                 1/1     Running   1          48d
zk-2                                 1/1     Running   2          175d

# kubectl -n ns-mysh get pods    
Error from server (Forbidden): pods is forbidden: User "reader_dev" cannot list resource "pods" in API group "" in the namespace "ns-mysh"
```



## 三、创建读写用户

过程和步骤二类似，role多一些

附role文件

```bash
# cat role2.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin-dev
  namespace: dev
rules:
- apiGroups:
  - ""
  resources:
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  - secrets
  - services/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - impersonate
- apiGroups:
  - ""
  resources:
  - pods
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - replicationcontrollers
  - replicationcontrollers/scale
  - secrets
  - serviceaccounts
  - services
  - services/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - statefulsets/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  - networkpolicies
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - persistentvolumeclaims/status
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - services/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - controllerrevisions
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - replicasets
  - replicasets/scale
  - replicasets/status
  - statefulsets
  - statefulsets/scale
  - statefulsets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  - horizontalpodautoscalers/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - cronjobs/status
  - jobs
  - jobs/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - ingresses
  - ingresses/status
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicasets/status
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  - poddisruptionbudgets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  - ingresses/status
  - networkpolicies
  verbs:
  - get
  - list
  - watch
  
  
  
  
# cat bin2_role.yaml 
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin-dev-binding
  namespace: dev
subjects:
- kind: User
  name: admin_dev
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: admin-dev
  apiGroup: ""
  
```



创建pod

```bash
# kubectl -n dev run my-nginx --image=nginx:latest --replicas=1 --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/my-nginx created

# 跨命名空间创建不成功
# kubectl -n ns-mysh  run my-nginx2 --image=nginx:latest --replicas=1 --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
Error from server (Forbidden): deployments.apps is forbidden: User "admin_dev" cannot create resource "deployments" in API group "apps" in the namespace "ns-mysh"
```

