flink的代码中 check point 时间是 1分钟

滚动策略中 withRolloverInterval 是 5分钟， 

写入hdfs， 则在hdfs上面每5分钟写一个新文件， 



如果程序写入一个新文件2分钟后， 异常退出，此时已经向kafka提交了2此offset 偏移量， 等程序重启的时候， 如何保证exactly one

