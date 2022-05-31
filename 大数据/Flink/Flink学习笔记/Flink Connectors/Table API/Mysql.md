# mysql

```sql
CREATE TABLE SourceTable (
     id BIGINT,
     uid STRING,
     product_name STRING,
     price float ,
     PRIMARY KEY (id) NOT ENFORCED
) WITH (
      'connector' = 'jdbc',
      'url' = 'jdbc:mysql://mysql:3306/cbs',
      'table-name' = 'order_info',
      'username' = 'root',
      'password' = 'root'
      );


CREATE TABLE SinkTable (
    product_name STRING,
    cnt BIGINT,
    primary key (product_name) not enforced
) WITH (
      'connector' = 'jdbc',
      'url' = 'jdbc:mysql://mysql:3306/cbs',
      'table-name' = 'product_cnt',
      'username' = 'root',
      'password' = 'root'
      );


INSERT INTO SinkTable
SELECT product_name, count(1) as cnt FROM SourceTable group by product_name;
```

```shell
/opt/software/flink-1.12.5/bin/flink run -d \
-m yarn-cluster \
-ynm test_flink_sql \
-p 1 \
-yjm 1024m \
-ytm 2048m \
-c com.flink.streaming.core.JobApplication \
/opt/software/flink-1.12.5/lib/flink-streaming-core.jar \
--sql /home/kyle0349/flink-job/flink_sql_test01.sql \
-yD table.exec.state.tt1=1800000 \
-type 0 \
--checkpointDir 'hdfs://hadoop:9000/flink/flink-job/checkpoint' \
--tolerableCheckpointFailureNumber '200' \
--checkpointInterval '360000' \
--checkpointTimeout '360000' ;
```

### 基于savepoint启动任务

> -s 参数

```shell
# 基于checkpoint启动任务
/opt/software/flink-1.12.5/bin/flink run -d \
-m yarn-cluster \
-ynm test_flink_sql \
-s hdfs://hadoop:9000/flink/ha \
-p 1 \
-yjm 1024m \
-ytm 2048m \
-c com.flink.streaming.core.JobApplication \
/opt/software/flink-1.12.5/lib/flink-streaming-core.jar \
--sql /home/kyle0349/flink-job/flink_sql_test01.sql \
-yD table.exec.state.tt1=1800000 \
-type 0 \
--checkpointDir 'hdfs://hadoop:9000/flink/flink-job/checkpoint' \
--tolerableCheckpointFailureNumber '200' \
--checkpointInterval '360000' \
--checkpointTimeout '360000' ;

```

