# Kafka命令



```shell
# 创建topic
/opt/kafka/bin/kafka-topics.sh --create --zookeeper 192.168.1.100:2181 --replication-factor 1 --partitions 1 --topic user-behavior-us

# 生产者
/opt/kafka/bin/kafka-console-producer.sh --broker-list 192.168.1.100:9092 --topic user-behavior-us

# 消费者
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.100:9092 --topic user-behavior-us --from-beginning

# 查看所有topic
/opt/kafka/bin/kafka-topics.sh --list --zookeeper 192.168.1.100:2181

# 查看消费者
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --list
```

