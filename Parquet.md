一、Spark SQL下的Parquet使用最佳实践
--

1）过去整个业界对大数据的分析的技术栈的Pipeline一般分为以下两种方式：

a）Data Source -> HDFS -> MR/Hive/Spark（相当于ETL）-> HDFS Parquet -> Spark SQL/Impala -> ResultService（可以放在DB中，也有可能被通过JDBC/ODBC来作为数据服务使用）；

b）Data Source -> Real timeupdate data to HBase/DB -> Export to Parquet -> Spark SQL/Impala -> ResultService（可以放在DB中，也有可能被通过JDBC/ODBC来作为数据服务使用）；

上述的第二种方式完全可以通过Kafka+Spark Streaming+Spark SQL（内部也强烈建议采用Parquet的方式来存储数据）的方式取代

2）期待的方式：DataSource -> Kafka -> Spark Streaming -> Parquet -> Spark SQL（ML、GraphX等）-> Parquet -> 其它各种Data Mining等。



参考链接
--
https://blog.csdn.net/ZYC88888/article/details/78273084 
