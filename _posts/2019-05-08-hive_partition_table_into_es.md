---
layout: post
title:  "Hive分区表导入ElasticSearch"
date:   2019-05-08 22:25:19 +0800
tags:
- Hive
- elasticsearch
color: rgb(0, 0, 205)
cover: '/blog/Halloween/2019_05_06_03.webp'
subtitle: 'hive分区表一次性导入ElasticSearch'
---
工作中总会碰到Hive数据导入ElasticSearch，可以写Spark程序读Hive表，再写入ES中。同样，ElasticSearch官方也提供了一个工具，elasticsearch-hadoop.jar工具包，在hive程序中引入该包，就可以创建表与ES相关联，向该表中导入数据，数据就会存入ES中。

hive分区表导入ES时，我们希望一次性导入就会在insert表时不加where条件，在实际使用过程中，碰到了一个问题，程序报错，信息如下，查了很久，网上关于这方面的资料很少，最终在elasticsearch-hadoop的项目issue中发现有人提了这个问题，说不是jar的问题，是parquet格式的问题

```shell
 ERROR [IPC Server handler 1 on 60702] org.apache.hadoop.mapred.TaskAttemptListenerImpl: Task: attempt_1556068676896_1760_m_000000_0 - exited : java.io.IOException: java.lang.reflect.InvocationTargetException
	at org.apache.hadoop.hive.io.HiveIOExceptionHandlerChain.handleRecordReaderCreationException(HiveIOExceptionHandlerChain.java:97)
	at org.apache.hadoop.hive.io.HiveIOExceptionHandlerUtil.handleRecordReaderCreationException(HiveIOExceptionHandlerUtil.java:57)
	at org.apache.hadoop.hive.shims.HadoopShimsSecure$CombineFileRecordReader.initNextRecordReader(HadoopShimsSecure.java:267)
	at org.apache.hadoop.hive.shims.HadoopShimsSecure$CombineFileRecordReader.next(HadoopShimsSecure.java:140)
	at org.apache.hadoop.mapred.MapTask$TrackedRecordReader.moveToNext(MapTask.java:199)
	at org.apache.hadoop.mapred.MapTask$TrackedRecordReader.next(MapTask.java:185)
	at org.apache.hadoop.mapred.MapRunner.run(MapRunner.java:52)
	at org.apache.hadoop.mapred.MapTask.runOldMapper(MapTask.java:459)
	at org.apache.hadoop.mapred.MapTask.run(MapTask.java:343)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:164)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1920)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.hadoop.hive.shims.HadoopShimsSecure$CombineFileRecordReader.initNextRecordReader(HadoopShimsSecure.java:253)
	... 11 more
Caused by: java.lang.IndexOutOfBoundsException: Index: 10, Size: 10
	at java.util.ArrayList.rangeCheck(ArrayList.java:657)
	at java.util.ArrayList.get(ArrayList.java:433)
	at org.apache.hadoop.hive.ql.io.parquet.read.DataWritableReadSupport.getProjectedGroupFields(DataWritableReadSupport.java:116)
	at org.apache.hadoop.hive.ql.io.parquet.read.DataWritableReadSupport.getSchemaByName(DataWritableReadSupport.java:176)
	at org.apache.hadoop.hive.ql.io.parquet.read.DataWritableReadSupport.init(DataWritableReadSupport.java:242)
	at org.apache.hadoop.hive.ql.io.parquet.read.ParquetRecordReaderWrapper.getSplit(ParquetRecordReaderWrapper.java:256)
	at org.apache.hadoop.hive.ql.io.parquet.read.ParquetRecordReaderWrapper.<init>(ParquetRecordReaderWrapper.java:95)
	at org.apache.hadoop.hive.ql.io.parquet.read.ParquetRecordReaderWrapper.<init>(ParquetRecordReaderWrapper.java:81)
	at org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat.getRecordReader(MapredParquetInputFormat.java:72)
	at org.apache.hadoop.hive.ql.io.CombineHiveRecordReader.<init>(CombineHiveRecordReader.java:68)
	... 16 more
```

既然导致这个错误的原因是源数据表是parquet格式，所以我们要创建张临时表，存储格式为orc格式，用于存储源表的数据，或者一开始，我们导入hive时就用orc格式

```sql
add jar elasticsearch-hadoop.jar;

USE tmp;
CREATE TABLE es_table(
        es_id  string COMMENT "id",
        name   string COMMENT "姓名",
        gender  int COMMENT "性别",
        age int COMMENT "年龄",
        birth_year  string COMMENT "出生年份"
        )
        STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler'
        TBLPROPERTIES (
	        'es.resource' = 'student/info',
	        'es.mapping.id' = 'es_id',
	        'es.index.auto.create' = 'true',
	        'es.mapping.routing' = 'es_id',
	        'es.nodes' = '127.0.0.1:9200'
        );

CREATE TABLE student_tmp (
	 es_id  string COMMENT "id",
    name   string COMMENT "姓名",
    gender  int COMMENT "性别",
    age int COMMENT "年龄",
) PATITIONED BY (birth_year String) STORED AS orc;

# 设置动态分区
SET hive.exec.dynamic.partition.mode=nonstrict;
INSERT OVERWRITE student_tmp PARTITION(birth_year) SELECT * from student;

INSERT OVERWRITE es_table SELECT * FROM student_tmp;
```

完成后，再将临时表drop掉，完美！如果想做成定时任务的话，可以用beeline执行hive sql，官方也不建议使用hive cli了，推荐使用beeline。

