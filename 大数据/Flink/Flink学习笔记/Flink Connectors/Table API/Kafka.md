# kafka



## 1. csv

```sql
create table user_behavir_from_kafka_01(
	  did string
  , uid string
  , product_id BIGINT
)WITH(
	'connector' = 'kafka',
  'topic' = 'user-behavior-us;user-behavior-cn',
  'properties.bootstrap.servers' = '192.168.1.100:9092',
  'properties.group.id' = 'testGroup',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'csv',
  'csv.escape-character' = '"',
  'csv.ignore-parse-errors' = 'true'
)
;

CREATE TABLE SinkTable (
      did STRING
    , uid STRING
    , product_id BIGINT
) WITH (
      'connector' = 'print'
      );
      
INSERT INTO SinkTable
SELECT did, uid, product_id from user_behavir_from_kafka_01;
   
```

```shell
/opt/software/flink-1.12.5/bin/flink run -d \
-m yarn-cluster \
-ynm test_flink_sql_kafka \
-p 1 \
-yjm 1024m \
-ytm 2048m \
-c com.flink.streaming.core.JobApplication \
/opt/software/flink-1.12.5/lib/flink-streaming-core.jar \
--sql /home/kyle0349/flink-job/flink_sql_kafka_csv.sql \
-yD table.exec.state.tt1=1800000 \
-type 0 \
--checkpointDir 'hdfs://hadoop:9000/flink/flink-job/checkpoint1' \
--tolerableCheckpointFailureNumber '200' \
--checkpointInterval '360000' \
--checkpointTimeout '360000' ;
```

