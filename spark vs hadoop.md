## hadoop streaming中-cacheArchive与-archives
```
archives是上传本地文件到hdfs然后在拉到NM本地。
cacheArchive是HDFS的路径，archives这个可能在框架上传的时候会设置更多的副本。
另外还可能在NM本地可能还有会软链和每个container一个文件的区别
```

spark 里上传的文件副本说也可以改: spark.yarn.submit.file.replication

不过对于hadoop streaming 上传文件少且作业时间长的job 来说，上传文件少，所以上传的文件多少对执行时间的影响
(上传会更慢，但 NM 从 HDFS 拉数据，即 localize 会变快)有限。作业执行时间长，这点时间差别可以忽略

对 hadoop streaming 来说，可能意义比较大，因为每个 task 都是一个 jvm / container，10万 task 就要 localize 10万次，能减少 localize 的成本也挺好

spark 不一样，jvm 是共用的，起了几千个 executor 后，一个 executor 可以执行 N 个 task，localize 的次数与 executor 个数相同，因此
spark的--archives参数可以替代hadoop streaming中-cacheArchive与-archives
