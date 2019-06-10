Catalyst是spark sql的核心，是一套针对spark sql语句执行过程中的查询优化框架。因此要理解spark sql的执行流程，理解Catalyst是关键。

spark sql 
           -> Unresolved Logical Pan -> Logical Plan -> Optimized Logical Plan -> Physical Plan
DataFrame                            ^
                                     |
                                  Catalog
                                  

Sql语句首先通过Parser模块被解析为语法树，此棵树称为Unresolved Logical Plan(是一种AST(抽象语法树);Unresolved Logical Plan 通过Analyzer模块借助于Catalog中的表信息解析为Logical Plan.
此时，Optimizer再通过各种基于规则的优化策略进行深入优化，得到Optimized Logical Plan.优化后的逻辑执行计划依然是逻辑的，并不能被spark所理解，需要将此逻辑计划转换为Physical
Plan.

进入sparkshell后，输入一下代码即可显示一个sql查询的执行计划：

```
spark.sql("select sum(chineseScore) from " +
      "(select x.id,x.chinese+20+30 as chineseScore,x.math from  studentTable x inner join  scoreTable y on x.id=y.sid)z" +
      " where z.chineseScore <100").explain(true)
```

```
/**
   * Prints the plans (logical and physical) to the console for debugging purposes.
   *
   * @group basic
   * @since 1.6.0
   */
  def explain(extended: Boolean): Unit = {
    val explain = ExplainCommand(queryExecution.logical, extended = extended)
    sparkSession.sessionState.executePlan(explain).executedPlan.executeCollect().foreach {
      // scalastyle:off println
      r => println(r.getString(0))
      // scalastyle:on println
    }
  }
 ```
 
 
