---
layout: post
title:  "Spark踩坑记（二）"
date:   2020-04-13 20:25:43 +0800
tags:
- Spark
- Sbt
- Maven
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2019_05_06_08.webp'
subtitle: '第三方包与spark集群guava包冲突'
---

> 前不久提交了个任务到集群中，死活不通过，查看日志，发现guava包冲突。项目中用了个第三方包，该第三方包使用了guava包，版本是19.0。而spark集群内部也会使用guava包，集群用的是CDH的，spark版本2.3，但是我最后查到guava版本竟然是11.0.2，如何查看集群jar版本，通过spark history server随意进入一个application下，然后点击上方Environment，在该页面下有Classpath Entries，如下图

![guava版本](/blog/blog_spark_hole_two/spark_guava_version.jpg)

> 出现该冲突的原因是JVM加载jar，如果有多个版本的jar，默认会先使用spark集群内部的，JVM一旦加载后，就不会进行第二次加载，如果运用了高版本的jar的一些特性，也就会无法运行程序。
> 
---

###解决方案
> 方案一：升级集群内部jar版本，且不说该方式不好评估升级jar后对集群带来的负面影响，该方式有些治标不治本吧，以后如果其他jar更新，集群每次都要更新jar版本，成本也太大了，不推荐使用
> 
> 方案二：使用shade方式替换包，推荐该方式
> 
> 	1）maven管理
> 
> 修改pom.xml

```java
<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <relocations>
                                <relocation>
                                    <pattern>com.google.common</pattern>
                                    <shadedPattern>com.xxx.shaded.com.google.common</shadedPattern>
                                </relocation>
                            </relocations>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/maven/**</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```
> 2) sbt管理
> 
> 修改项目目录的project/plugins.sbt，如果没有该文件则创建该文件，添加assembly插件addSbtPlugin(\"com.eed3si9n\" % \"sbt-assembly\" % \"0.14.10\")
>  
> 修改build.sbt

```scala
assemblyShadeRules in assembly := Seq(
  ShadeRule.rename("com.google.common" -> "com.xxx.shaded.com.google.common").inAll
)
```

> assembly插件打的是个fat jar，需要注意的是maven方式打得fat jar是没办法完成包名替换的，还是会存在guava包冲突

---
> 本人使用的是maven方式，打完包后需要验证是否替换包名成功
> 
> 下载个jd-gui工具验证，[官方下载地址](http://java-decompiler.github.io/)，下载完安装成功后打开，拖动jar包进入应用程序窗口内会自动打开jar包，包名未完成替换jar如下图
![未替换图片](/blog/blog_spark_hole_two/jar_origin.jpg)

> 替换成功后的截图如下，可见com.google.common.base变成了com.feiniu.shaded.com.google.common包
![替换成功图片](/blog/blog_spark_hole_two/jar_replace_success.jpg)

---
> 总结下，无论是自己项目中直接引入guava包，还是引用的第三方包用了guava包都可以完成替换，类推下，所以其他包与spark集群包冲突都可以通过shade化来解决