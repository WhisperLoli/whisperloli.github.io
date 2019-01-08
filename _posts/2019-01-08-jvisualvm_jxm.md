---
layout: post
title:  "jvisualvm监控远程服务器"
date:   2019-01-08 19:24:56
tags:
- JVM
- Scala
color: rgb(255,90,90)
cover: '/blog/Halloween/witch_moon.jpg'
subtitle: '连接方式JMX'
---
添加JVM参数，在这里未使用身份验证，生产环境建议开启用户名密码验证，端口号为18999，可自行更改，同jstatd方式一样，远程访问需要开放指定的端口

```shell
-Djava.rmi.server.hostname=ip(域名) -Dcom.sun.management.jmxremote.port=18999 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false
```

![image](/blog/blog_jvm/success.jpg)

注意：Scala程序做成systemd服务启动，添加JVM参数需要在参数前加-J，如下

![image](/blog/blog_jvm/scala_style.jpg)