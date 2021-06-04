如果有yaml文件，可以通过yaml文件删除pod。这里讲的是没有yaml文件的情况下



查看pod

```bash
# kubectl -n ns-mysh get pods|grep -i java
java-demo-5cfdfcf75c-sww64                  0/1     Running            17         51m
java-demo-668db4bbfb-hbr2k                  0/1     CrashLoopBackOff   15         45m
```





删除pod，不一会又出来了

```bash
#  kubectl -n ns-mysh delete pod java-demo-5cfdfcf75c-sww64 
pod "java-demo-5cfdfcf75c-sww64" deleted
# kubectl -n ns-mysh delete pod java-demo-668db4bbfb-hbr2k
pod "java-demo-668db4bbfb-hbr2k" deleted

# kubectl -n ns-mysh get pods|grep -i java                 
java-demo-5cfdfcf75c-rlfr5                  0/1     Running   0          22s
java-demo-668db4bbfb-544cj                  0/1     Running   0          10s
```



查看deployment

```bash
# kubectl get deployment -n ns-mysh
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE

java-demo                  0/1     1            0           52m
```



查看deployment

```bash
# kubectl -n ns-mysh delete deployment java-demo 
deployment.apps "java-demo" deleted
```



再查看pod已经没有了