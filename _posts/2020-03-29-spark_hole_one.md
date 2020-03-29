---
layout: post
title:  "Spark踩坑记（一）"
date:   2020-03-29 21:01:29 +0800
tags:
- Spark
- Spark Streaming
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2019_05_06_07.webp'
subtitle: 'spark创建redis连接 Task not serializable'
---

在上周的项目中碰到了个坑吧
> spark streaming消费kafka数据存redis。kafka和redis都是自己公司内部封装过的，不能直接使用spark-redis连接内部的redis，没办法，只能老老实实的用自己公司提供的redis-client。每次流处理都要写redis，所以最好是只创建一个redis client，然后分发到所有excutor中。

> 自己公司内部的redis-client中的类没有实现序列化，spark分发时需要序列化对象再传输。
如果类没有实现序列化，就会报错org.apache.spark.SparkException: Task not serializable。我觉得这个简单啊，然后就上百度搜了一下。CSDN上很多人转载文章，都是一样的解决方案，给类中成员变量加注解@transient，成员对象不参与序列化。我去试了，根本不行，还有一些解决方案，还有些说用spark broadcast，但是广播也是需要序列化的，还有说把scala方法改成scala函数，我真的无语了。

> 我研究了挺久吧，最终得以解决。**<u>划重点，类中使用第三方成员变量未实现序列化且无法修改源码，使用伴生对象object创建该类的实例，类似单例模式，spark中使用object类中创建的对象，该方法无需序列化。scala中object类相当于java静态类，类中方法、属性皆为静态方法、属性。</u>**

> 还有一个就是在executor中直接调用driver端创建的SparkContent对象也会报错，信息为Caused by: java.io.NotSerializableException: org.apache.spark.SparkContext，SparkContext对象也是不支持序列化的，如果需要使用，可以遍历RDD的使用使用rdd.sparkContext方法获得

接下来提供一下我对流处理中redis-client创建的见解吧。大致可以分为一下几种方法吧。

####方法一

> 使用collect方法，将executor中的数据收集到driver端，再使用redis-client操作
> 
> 优点：redis-client可以不用序列化，且只创建一个连接
> 
> 缺点：所有数据都collect到driver端，会对driver造成压力，数据量过大还会直接OOM

####方法二

> 使用foreachPartition/foreach方法遍历流中的数据，在这里面new RedisClient操作，foreach遍历会每条数据都创建一个连接，性能更差，无需考虑
> 
> 优点：redis-client无需实现序列化
> 
> 缺点：会根据partition数量创建n个client，且均为短连接，partition数据太多的话会对redis集群造成压力


####方法三

> executor使用driver创建的一个连接
> 
> 优点：只有一个连接
> 
> 缺点：client需要实现序列化


####方法四

> 即上述我使用的方法，采用伴生对象object创建类实例
> 
> 优点：client无需序列化，应该只创建一个连接（未测试，应该是在driver端初始化该object，如果在每个executor JVM中创建的话，则创建的client数量会和executor数量一致，如果是的话，解决方案可以在driver端调用object中的方法强行初始化）
> 
> 缺点：貌似没啥缺点，是除开可以序列化之外最完美的方案

可能会有人问，为什么不使用广播，广播会比直接在executor中使用性能好。

> 我测试过了，使用广播的话，会抛出ConcurrentModificationException的异常，这个java异常是多线程并发改一个对象产生的异常。这个异常可能是我司的redis-client才有也不一定，可以做尝试。
