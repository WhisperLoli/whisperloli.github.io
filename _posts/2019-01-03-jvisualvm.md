---
layout: post
title:  "jvisualvm监控远程服务器"
date:   2019-01-03 21:05:21
tags: JVM
color: rgb(30, 144, 255)
cover: '/blog/Halloween/witch.webp'
subtitle: '连接方式jstatd'
---

步骤

1. ssh登录服务器

	```
ssh root@xxx.xxx.xxx.xxx
```
	
2. 在家目录下创建授权文件

	```shell
cd ~
vi jstatd.all.policy
#在文件中输入如下内容
grant codebase "file:${java.home}/../lib/tools.jar" {
	permission java.security.AllPermission;
};
```

3. 启动jstatd服务，默认连接端口号为1099

	```shell
${JAVA_HOME}/bin/jstatd -J-Djava.security.policy=jstatd.all.policy
```

本地连接不上远程解决方案

1. 服务器上开启了防火墙，需要开放1099端口，或者关闭防火墙，需要注意的是，仅仅开放1099端口可能还是连接不上，因为jstatd服务有两个端口，另一个端口也需要开放
![image](/blog/blog_jvm/jstatd_port.jpg)
如图所示，还有另一个端口37529，经测试，该端口是随机的，每次开启jstatd服务，该端口都会变化，检查端口是否开放，在本地执行如下命令
 
    ```shell
telnet ip port #如果端口未开放，则连接会被拒绝或者卡在尝试连接中
```

2. 如果端口均可以访问，执行如下命令启动服务

	```shell
${JAVA_HOME}/bin/jstatd -J-Djava.security.policy=jstatd.all.policy -J-Djava.rmi.server.hostname=域名(IP)
```


