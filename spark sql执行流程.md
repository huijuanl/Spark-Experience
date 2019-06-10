Catalyst是spark sql的核心，是一套针对spark sql语句执行过程中的查询优化框架。因此要理解spark sql的执行流程，理解Catalyst是关键。

spark sql 
           -> Unresolved Logical Pan -> Logical Plan -> Optimized Logical Plan -> Physical Plan
DataFrame                            ^
                                     |
                                  Catalog
                                  

Sql语句首先通过Parser模块被解析为语法树，此棵树称为Unresolved Logical Plan;Unresolved Logical Plan 通过Analyzer模块借助于Catalog中的表信息解析为Logical Plan.
此时，Optimizer再通过各种基于规则的优化策略进行深入优化，得到Optimized Logical Plan.优化后的逻辑执行计划依然是逻辑的，并不能被spark所理解，需要将此逻辑计划转换为Physical
Plan.

