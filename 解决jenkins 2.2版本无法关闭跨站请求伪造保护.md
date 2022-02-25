http://t.zoukankan.com/panchanggui-p-13557155.html



高版本jenkins不能界面禁用跨站请求伪造保护。

禁用跨站请求伪造保护操作如下：

修改jenkins的配置文件。

```bash
# vim /etc/sysconfig/jenkins
JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true 
改成
JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"

```



再重启

```bash
# service jenkins restart 
```





再在gitlab上指向hook test

返回正常

![企业微信截图_20220225162336.png](https://tva1.sinaimg.cn/large/007Xg1efgy1gzpu42dz3ij318z0osaks.jpg)

