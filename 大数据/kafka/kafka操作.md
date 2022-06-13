# Kafka命令



```shell
# 创建topic
/opt/kafka/bin/kafka-topics.sh --create --zookeeper 192.168.1.100:2181 --replication-factor 1 --partitions 1 --topic user-behavior-us

# 生产者
/opt/kafka/bin/kafka-console-producer.sh --broker-list 192.168.1.100:9092 --topic user-behavior-us
# tid-flink-01,{05094852-e434-11ec-b034-acde48001122},100107001,104,Chinese,192.168.1.100,China,mainstart_up,1s,1654367261556,1654367261556,20220605

# 消费者
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.100:9092 --topic user-behavior-us --from-beginning

# 查看所有topic
/opt/kafka/bin/kafka-topics.sh --list --zookeeper 192.168.1.100:2181

# 查看消费者
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --list
```

