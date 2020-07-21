---
layout: post
title:  "Spark源码学习笔记(二十)"
date:   2020-07-19 12:33:13 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020070805.webp'
subtitle: Spark SQL之逻辑计划
---
> 在Spark SQL中，Catalog主要用于各种资源信息及元数据信息的统一管理，spark catalog主要分为InMemoryCatalog和HiveExternalCatalog，二者均继承自ExternalCatalog，SessionCatalog起到了一个代理的作用，对底层的元数据、临时表、视图、函数等进行了封装，通过多态调用ExternalCatalog的方法进而执行InMemoryCatalog/HiveExternalCatalog中的方法。在整个Spark SQL中起到了关键性作用。
> 
> spark sql的查询分为逻辑计划和物理计划
> 
> 逻辑计划又细分为Unresolved LogicalPlan，Analyzed LogicPlan，Optimized LogicalPlan，用户执行的SQL经过SparkSqlParser处理后转换为Unresolved LogicalPlan
> 
> Unresolved LogicalPlan是逻辑算子树的最初形态，但是不包含数据信息与列信息
> 
> Analyzer将一系列的规则作用于Unresolved LogicalPlan，对树上的节点绑定各种数据信息，生成解析后的Analyzed LogicPlan
> 
> Optimizer将一系列的优化规则作用到Analyzed LogicPlan上，在确保结果正确的前提下，改写其中的低效结果，生成Optimized LogicalPlan，这步优化被称为RBO，因为是基于规则的优化
> 
> LogicalPlan继承自QueryPlan，QueryPlan继承自TreeNode，所以LogicalPlan也是属于TreeNode结构
> 
> QueryPlan中有一个statePrefix方法，如果该计划不可用，则前缀会用感叹号(!)标记
> 
> LogicalPlan有三个子类，分别是LeafNode(叶子结点), UnaryNode(一元子节点，只有一个子孩子的节点),BinaryNode(二元子节点，拥有左右孩子的节点)
> 
> 大致介绍了下涉及到的类，下面说下AST（抽象语法树）,用户的SQL被ANTLR4解析生成抽象语法树，SparkSqlParser中使用AstBuilder访问者类对语法树进行访问，在这里需要说一下，Spark采用了访问者模式进行访问树的每个节点
> 
> 用户的sql会先经过AbstractSqlParser的parsePlan进行处理，SparkSqlParser继承自AbstractSqlParser，SparkSqlParser并没有对parsePlan方法重写，parsePlan方法先调用parse(sqlText)方法，该方法会进入子类的parse方法，最终还是会回到AbstractSqlParser的parse方法，SqlBaseLexer是词法分析器，SqlBaseParser是语法解析器，最终进入toResult方法，toResult就是parsePlan方法中调用parse方法传递的第二个参数，第二个参数是个函数，在这个函数里面调用astBuilder.visitSingleStatement方法，进入访问者模式


```scala
/** Creates LogicalPlan for a given SQL string. */
override def parsePlan(sqlText: String): LogicalPlan = parse(sqlText) { parser =>
    astBuilder.visitSingleStatement(parser.singleStatement()) match {
      case plan: LogicalPlan => plan
      case _ =>
        val position = Origin(None, None)
        throw new ParseException(Option(sqlText), "Unsupported SQL statement", position, position)
    }
}

protected def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
  logDebug(s"Parsing command: $command")

  val lexer = new SqlBaseLexer(new UpperCaseCharStream(CharStreams.fromString(command)))
  lexer.removeErrorListeners()
  lexer.addErrorListener(ParseErrorListener)

  val tokenStream = new CommonTokenStream(lexer)
  val parser = new SqlBaseParser(tokenStream)
  parser.addParseListener(PostProcessor)
  parser.removeErrorListeners()
  parser.addErrorListener(ParseErrorListener)

  try {
    try {
      // first, try parsing with potentially faster SLL mode
      parser.getInterpreter.setPredictionMode(PredictionMode.SLL)
      toResult(parser)
    }
    catch {
      case e: ParseCancellationException =>
        // if we fail, parse with LL mode
        tokenStream.seek(0) // rewind input stream
        parser.reset()

        // Try Again.
        parser.getInterpreter.setPredictionMode(PredictionMode.LL)
        toResult(parser)
      }
    }
    catch {
      case e: ParseException if e.command.isDefined =>
        throw e
      case e: ParseException =>
        throw e.withCommand(command)
      case e: AnalysisException =>
        val position = Origin(e.line, e.startPosition)
        throw new ParseException(Option(command), e.message, position, position)
  }
}
  
  
class SparkSqlParser(conf: SQLConf) extends AbstractSqlParser {
  val astBuilder = new SparkSqlAstBuilder(conf)

  private val substitutor = new VariableSubstitution(conf)

  protected override def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
    super.parse(substitutor.substitute(command))(toResult)
  }
}
```
> 对树根节点的访问会递归访问树的子节点，先执行visitFromClause构造from Logical Plan，再调用visitTableName，最后构造出整个Unresolved Logical Plan并返回
> 
> Analyzer继承自RuleExecutor，通过模式匹配相应的规则并进行替换，在QueryExecution中会触发Analyzer的executeAndCheck方法，父类中的execute方法会将所有规则应用到逻辑算子树上，并且会对列名解析到相应的下标，Analyzed LogicPlan就生成了

```scala
// QueryExecution中的方法
lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
}

// Analyzer中的方法，第一行代码调用execute方法
def executeAndCheck(plan: LogicalPlan): LogicalPlan = {
    val analyzed = execute(plan)
    try {
      checkAnalysis(analyzed)
      EliminateBarriers(analyzed)
    } catch {
      case e: AnalysisException =>
        val ae = new AnalysisException(e.message, e.line, e.startPosition, Option(analyzed))
        ae.setStackTrace(e.getStackTrace)
        throw ae
    }
}
// 执行executeSameContext方法
override def execute(plan: LogicalPlan): LogicalPlan = {
    AnalysisContext.reset()
    try {
      executeSameContext(plan)
    } finally {
      AnalysisContext.reset()
    }
}
// executeSameContext方法会调用父类RuleExecutor的execute方法
private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
```

> Optimizer同样也是继承自RuleExecutor，而SparkOptimizer继承自Optimizer，QueryExecution中也定义了optimizedPlan，会调用optimizer的execute方法，由于Optimizer和SparkOptimizer中都没有重写execute方法，所以直接调用RuleExecutor的execute方法，然后将自己的规则batches应用到算子树中

```scala
lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)
```
> 最后生成的优化后的算子树将作为Spark SQL中生成物理计划的输入
> 
> 如下上半部分是执行select * from student where year = '2010'得到的逻辑计划，Unresolved LogicalPlan自底向上对应着相应的执行顺序，第一步是生成表student对应的逻辑计划，第二步是加入过滤后的逻辑计划，第三步是生成加入列裁剪后的逻辑计划，因为sql中查询的是\*，所以Project对应的\*

```
== Parsed Logical Plan ==
'Project [*]
+- 'Filter ('year = 2010)
   +- 'UnresolvedRelation `student`

== Analyzed Logical Plan ==
year: string, class: string, student: string, score: string
Project [year#10, class#11, student#12, score#13]
+- Filter (year#10 = 2010)
   +- SubqueryAlias student
      +- Relation[year#10,class#11,student#12,score#13] csv

== Optimized Logical Plan ==
Filter (isnotnull(year#10) && (year#10 = 2010))
+- Relation[year#10,class#11,student#12,score#13] csv

== Physical Plan ==
*(1) Project [year#10, class#11, student#12, score#13]
+- *(1) Filter (isnotnull(year#10) && (year#10 = 2010))
   +- *(1) FileScan csv [year#10,class#11,student#12,score#13] Batched: false, Format: CSV, Location: InMemoryFileIndex[file:/Users/admin/Desktop/student.csv], PartitionFilters: [], PushedFilters: [IsNotNull(year), EqualTo(year,2010)], ReadSchema: struct<year:string,class:string,student:string,score:string>
```

> 
> 本来想着也进行调试的，看代码如何运行，及运行中的变量结果，可是发现SqlBaseBaseVisitor，SqlBaseVisitor，SqlBaseParser等这些类都是由SqlBase.g4生成的代码，在spark源码中根本无法debug，但是可以在自己创建的项目中进行调试运行到spark相应的类中