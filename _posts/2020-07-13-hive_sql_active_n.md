---
layout: post
title:  "SQL实现连续N天活跃用户"
date:   2020-07-13 21:24:13 +0800
tags:
- Hive
- SQL
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070804.webp'
subtitle: 连续N天活跃用户
---
> 在用户行为日志中寻找连续N天活跃的用户属于常见需求，但是真实的数据会比我这个数据复杂，毕竟用户行为中，用户可能一天登录多次，时间也是精确到毫秒
> 
> 原始数据如下
> ![](/blog/blog_hive_sql_active_n/origin_data.png)
> 
> hive中建表及插入数据语句如下
> 
> ```sql
> CREATE TABLE tm_login_log
 (
   user_id int,
   login_date String
 )
 ;
```

> ```sql
 insert into table tm_login_log select 1001,'2017-01-01';
 insert into table tm_login_log select 1001,'2017-01-02';
 insert into table tm_login_log select 1001,'2017-01-04';
 insert into table tm_login_log select 1001,'2017-01-05';
 insert into table tm_login_log select 1001,'2017-01-06';
 insert into table tm_login_log select 1001,'2017-01-07';
 insert into table tm_login_log select 1001,'2017-01-08';
 insert into table tm_login_log select 1001,'2017-01-09';
 insert into table tm_login_log select 1001,'2017-01-10';
 insert into table tm_login_log select 1001,'2017-01-12';
 insert into table tm_login_log select 1001,'2017-01-13';
 insert into table tm_login_log select 1001,'2017-01-15';
 insert into table tm_login_log select 1001,'2017-01-16';
 insert into table tm_login_log select 1002,'2017-01-01';
 insert into table tm_login_log select 1002,'2017-01-02';
 insert into table tm_login_log select 1002,'2017-01-03';
 insert into table tm_login_log select 1002,'2017-01-04';
 insert into table tm_login_log select 1002,'2017-01-05';
 insert into table tm_login_log select 1002,'2017-01-06';
 insert into table tm_login_log select 1002,'2017-01-07';
 insert into table tm_login_log select 1002,'2017-01-08';
 insert into table tm_login_log select 1002,'2017-01-09';
 insert into table tm_login_log select 1002,'2017-01-10';
 insert into table tm_login_log select 1002,'2017-01-11';
 insert into table tm_login_log select 1002,'2017-01-12';
 insert into table tm_login_log select 1002,'2017-01-13';
 insert into table tm_login_log select 1002,'2017-01-16';
 insert into table tm_login_log select 1002,'2017-01-17';
> ```
> 
> 如果真实日期数据精确到毫秒，并且一天内存在多次登录，则可以截取时间到yyyy-mm-dd格式，再group by去重
> 
> 求连续活跃N天用户的这个问题可以用lag函数解决，lag函数可以向上取n行，如果取连续8天活跃的用户，所以sql就可以如下
> 
> ```sql
> SELECT  *
        ,datediff(login_date,pre_8_day)
FROM    (
            SELECT  a.user_id
                    ,a.login_date
                    ,lag(a.login_date,7) OVER(PARTITION BY a.user_id ORDER BY a.login_date) pre_8_day
            FROM    tm_login_log a
        ) b
;
> ```
> 
> 结果如下，倒数第二列就是向上取7行的结果，需要用到窗口函数对用户分组，因为连续8天活跃，所以向上取7行就行，但是得根据日期升序排序，最后一列的结果是当前日期与向上7行的日期差。原理就是lag函数取值后，如果中间没有跳跃值，向上取n的日期值与当前的日期差值应当为n，数据集存在重复日期肯定会有问题，这就是为什么要提前去重的原因
> 
> ![](/blog/blog_hive_sql_active_n/sql_result.png)
> 
> 所以到这里，取出用户就很容易了，直接过滤出日期差值为7的数据，去重用户ID即可
> 
> ```sql
> SELECT  DISTINCT(user_id)
FROM    (
            SELECT  a.user_id
                    ,a.login_date
                    ,lag(a.login_date,7) OVER(PARTITION BY a.user_id ORDER BY a.login_date) pre_8_day
            FROM    tm_login_log a
        ) b
WHERE   datediff(login_date,pre_8_day) = 7
;
> 
> ```
> 
> 结果如下，只有1002用户符合
> 
> ![](/blog/blog_hive_sql_active_n/end_result.png)
> 