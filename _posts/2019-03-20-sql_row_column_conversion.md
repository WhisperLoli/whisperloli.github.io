---
layout: post
title:  "MySQL行列转换"
date:   2019-03-20 16:33:12 +0800
tags: 
- SQL
- MySQL
color: rgb(238, 0, 0)
cover: '/blog/Halloween/2019_05_06_02.webp'
subtitle: 'MySQL行列转换'
---

先创建表

```sql
CREATE TABLE `grade` (
  `name` text,
  `course` text,
  `score` int(11) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

插入数据

```sql
INSERT INTO `grade` (`name`,`course`,`score`) VALUES('张三','语文',76);
INSERT INTO `grade` (`name`,`course`,`score`) VALUES('李四','语文',76);
INSERT INTO `grade` (`name`,`course`,`score`) VALUES('李四','数学',90);
INSERT INTO `grade` (`name`,`course`,`score`) VALUES('张三','数学',75);

```

列转行

```sql
select name,
	max(case when course = "数学" then score else 0 end) "数学",
    max(case when course = "语文" then score else 0 end) "语文" 
from grade group by `name`;
```

![image](/blog/blog_sql_row_column_conversion/column_to_row.webp)

创建另一张表及插入数据

```sql
/*建表*/
create table test.p_grade(
	`name` varchar(255),
    `math` int,
    `chinese` int
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

/*将列转行的结果作为新的结果插入表中*/
insert into test.p_grade (select name,
	max(case when course = "数学" then score else 0 end) "数学",
    max(case when course = "语文" then score else 0 end) "语文" 
from test.grade group by `name`);
```

行转列

```sql
/* database名字为test */
select `name`, '语文' `subject`, chinese from test.p_grade 
union all 
select `name`, '数学' `subject`, math from test.p_grade;
```

![image](/blog/blog_sql_row_column_conversion/row_to_column.webp)
