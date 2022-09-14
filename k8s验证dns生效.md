1、检查组件

```bash
# kubectl -n kube-system get pods
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-97769f7c7-xztdp   1/1     Running   0          38m
calico-node-9w6d6                         1/1     Running   0          38m
calico-node-wrdh2                         1/1     Running   0          38m
calico-node-xtbtn                         1/1     Running   0          38m
coredns-6cc56c94bd-twvq2                  1/1     Running   0          39m
```





2、DNS解析

```bash
# kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes
Server:         10.0.0.2
Address:        10.0.0.2#53

Name:   kubernetes.default.svc.cluster.local
Address: 10.0.0.1

```

