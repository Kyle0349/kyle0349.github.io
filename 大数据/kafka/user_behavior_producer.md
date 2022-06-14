```python
"""
Kafka Producer Test
using kafka-python library
"""
# -*- encoding: utf-8 -*-

from kafka import KafkaProducer
from kafka.errors import KafkaTimeoutError
import time
import uuid
import random


def main():
    # 创建producer实例，并传入bootstrap_servers列表(brokers), 修改producer实例配置
    producer = KafkaProducer(bootstrap_servers=["192.168.1.100:9092"])

    # topic to be published
    topic = 'user-behavior-us'

    try:

        key = bytes('shay', encoding='utf-8')
        cnt = 0
        cnt2 = 0
        cycle = 0
        while True:
            cur_time = int(time.mktime(time.localtime(time.time() * 1000)))

            msg_str = msg_handler()

            msg = bytes(msg_str, encoding='utf-8')

            future = producer.send(topic, msg, key, partition=None, timestamp_ms=cur_time)
            producer.flush()
            # print("Message Send! --- ", msg_str)
            time.sleep(0.1)
            cnt = cnt + 1
            cnt2 = cnt2 + 1
            if cur_time % 9 == 0:
                print(stamp2day(cur_time), '>>> cnt: ', cnt, ">>>>>: cnt2", cnt2)
                cnt = 0
                cycle = cycle + 1
                print(">>>>> cycle: ", cycle)

    except  KafkaTimeoutError:
        print("Kafka Timeout")

# 随机生成产品id
def product_handler():
    x = random.randint(1, 100)
    return x


def msg_handler():
    cur_time = int(time.mktime(time.localtime(time.time() * 1000)))

    tid = "tid-flink-01"
    did = uuid_handler()
    uid = "100107001"
    product_id = str(product_handler())
    lang = "Chinese"
    ip = "192.168.1.100"
    country = "China"
    category = "main"
    act = "start_up"
    label = "1s"
    ostime = str(cur_time)
    recvtime = str(cur_time)
    recvdate = stamp2day(cur_time, '%Y%m%d')

    msg_str = tid + "," + did + "," + uid + "," + product_id + "," + lang + "," + ip + "," + country + "," + \
              category+ ","  + act + "," + label + "," + ostime + "," + recvtime + "," + recvdate

    return msg_str


def stamp2day(cur_time, date_format="%Y-%m-%d %H:%M:%S"):
    return time.strftime(date_format, time.localtime(cur_time/1000))


def uuid_handler():
    return "{" + str(uuid.uuid4()) + "}"


if __name__ == '__main__':
    main()
```

