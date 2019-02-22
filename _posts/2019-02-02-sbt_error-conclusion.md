---
layout: post
title:  "引入sbt错误"
date:   2019-02-02 22:59:37 +0800
tags: Sbt
color: rgb(255,90,90)
cover: '../blog/Halloween/happy.webp'
subtitle: 'sbt引入错误 & 年度小结'
---

今天使用IntelliJ IDEA创建项目，碰到个古怪的错误，创建完报错，无法引入sbt，错误信息如下
![image](/blog/blog_sbt_import_error_conclusion/import_error.jpg)

根据错误信息，大胆猜测本地可能没有sbt1.2.8，于是进入sbt本地仓库中查看
![image](/blog/blog_sbt_import_error_conclusion/sbt_local_version.jpg)

果然，本地仓库中sbt版本没有1.2.8，于是切换项目的sbt版本，切换sbt版本需要修改build.properties中sbt的版本号，切换操作如下图显示
![image](/blog/blog_sbt_import_error_conclusion/sbt_version.jpg)


PS：明天回家，准备过年。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从去年十一月份出来实习到现在，已经工作一年多了，学到了很多，技术上得到了很大的成长，虽然有时很累，但是更多的还是开心，语言也从python到scala，再到python，仍然记得当初秋招和实习找工作时我想做的是Java开发。工作中慢慢的接触到大数据领域，再慢慢接触到算法领域，也学了一段时间的算法，常见分类、聚类算法，scikit-learn等，个人更感兴趣的还是大数据领域。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;今年，告别校园生涯，走上社会，遗憾的是说好的毕业旅行泡汤了。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;毕业后没多久，公司也发生了很多变动，leader的离职对我的打击还是挺大的吧，从没想过他会离职，真的非常优秀的leader，还会给我们指导学习方向，教会了我很多东西。还记得之前他出差，还和我语音通话一起调bug到凌晨，真的非常感谢。在leader离职后的日子里，虽然技术上没什么大的成长，但是在社会阅历上也有很多成长，因为缺少了前leader这棵大树，可以看到公司的很多实际情况，或许，也是在这个时候才真正的了解公司了。才开始知道其实很多事情并没有表面看上去那么光鲜亮丽吧！也见识了各种各样的甩锅技术，可悲的是很多人怕背锅连最基本的担当都丢失了，出了事情把所有的责任往外推的一干二净。有过希望，也有过失落。这一年就这么过去了，真的好快，感觉自己还有很多东西没学，还有很多东西不懂。来年，一定变得更优秀！  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这一年里，最最最重要的是收获了真挚的友谊（小肥汪&刺猬妈），这是最开心的事情了。偶尔一起聚个餐，周末一起出去玩，让孤孤单单的我也感受到了温暖。谢谢你们！愿友谊地久天长！   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2018，再见！  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2019，加油！
