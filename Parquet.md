一、Spark SQL下的Parquet使用最佳实践
--

1）过去整个业界对大数据的分析的技术栈的Pipeline一般分为以下两种方式：

a）Data Source -> HDFS -> MR/Hive/Spark（相当于ETL）-> HDFS Parquet -> Spark SQL/Impala -> ResultService（可以放在DB中，也有可能被通过JDBC/ODBC来作为数据服务使用）；

b）Data Source -> Real timeupdate data to HBase/DB -> Export to Parquet -> Spark SQL/Impala -> ResultService（可以放在DB中，也有可能被通过JDBC/ODBC来作为数据服务使用）；

上述的第二种方式完全可以通过Kafka+Spark Streaming+Spark SQL（内部也强烈建议采用Parquet的方式来存储数据）的方式取代

2）期待的方式：DataSource -> Kafka -> Spark Streaming -> Parquet -> Spark SQL（ML、GraphX等）-> Parquet -> 其它各种Data Mining等。


Parquet的精要介绍
--
Parquet是列式存储格式的一种文件类型，列式存储有以下的核心优势：

a）可以跳过不符合条件的数据，只读取需要的数据，降低IO数据量。

b）压缩编码可以降低磁盘存储空间。由于同一列的数据类型是一样的，可以使用更高效的压缩编码（例如RunLength Encoding和Delta Encoding）进一步节约存储空间。

c）只读取需要的列，支持向量运算，能够获取更好的扫描性能。

Parquet 格式是 Spark SQL 的默认数据源，可通过 spark.sql.sources.default 配置

spark sql 可以对parquet文件进行读写

参考链接
--
https://blog.csdn.net/ZYC88888/article/details/78273084 
