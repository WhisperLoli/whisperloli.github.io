---
layout: post
title:  "Spark SQL窗口函数使用"
date:   2019-11-07 20:36:49 +0800
tags:
- Spark SQL
- SQL
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2019_05_06_05.webp'
subtitle: '万能窗口函数'
---

数据集

```csv
year,class,student,score
2010,语文,A,84
2010,数学,A,59
2010,英语,A,30
2010,语文,B,44
2010,数学,B,76
2010,英语,B,68
2011,语文,A,51
2011,数学,A,94
2011,英语,A,71
2011,语文,B,87
2011,数学,B,44
2011,英语,B,40
2012,语文,A,91
2012,数学,A,50
2012,英语,A,81
2012,语文,B,81
2012,数学,B,84
2012,英语,B,98
```

问题一：每学年每门学科的第一名？

问题二：每年总成绩都有提升的学生？

如上两个问题使用普通SQL都不好解决，这时就该窗口函数登场了，第一个问题其实不通过窗口函数也可以解决，但会存在缺点，解法一通过SQL操作，解法二通过窗口函数，事先通过spark将数据集读入DataFrame，即为df，第二个问题则要复杂很多

```
问题一 

解法一
思路：根据成绩排序，高分排在前，低分在后，然后根据学年、学科两个字段去重，会去除掉同学年、
同学科成绩低于最高分的记录
缺点：如果有学科并列第一，也会被去掉一条记录，只保留一条记录
df.orderBy(desc("score"))
  .dropDuplicates("year","class")
  .orderBy("year")
  .show()
      
解法二
思路：通过窗口函数获取每学年、每学科的最高成绩，创建新的一列，该列的值为当前学年、当前学科
的最高成绩，之后过滤出成绩与最高成绩相等的记录，该操作不会存在上面方法的缺点
df.withColumn("max", max("score").over(Window.partitionBy("year","class")))
  .filter($"score" === $"max")
  .orderBy("year")
  .drop("max")
  .show()
      
问题二
思路：先通过窗口函数计算每学年每个学生的总成绩，然后只保留 学年、学生、总成绩 三列，筛选完
字段会有重复记录需要去重，之后通过窗口函数lag添加新的一列previous，该列的值为该学生去年的
总成绩，第一个学年的上一学年的总成绩为null，当上一学年的总成绩为null或者当前年的总成绩大于
上一学年的总成绩，对应新列flag相应的值置为1，否则置为0，最后通过窗口函数判断每一位学生的
flag列的平均值，如果平均值还是1，则该学生每年的总成绩都比上一年总成绩高
df.withColumn("sum", sum("score").over(Window.partitionBy("student","year")))
  .select("year","student", "sum")
  .distinct()
  .withColumn("previous", lag("sum",1).over(Window.partitionBy("student").orderBy("student","year")))
  .withColumn("flag", when(($"previous".isNull).or($"sum" > $"previous"),1).otherwise(0))
  .drop("sum", "previous")
  .withColumn("avg", avg("flag").over(Window.partitionBy("student")))
  .filter($"avg" === 1)
  .dropDuplicates("student")
  .select("student")
  .show()

```

