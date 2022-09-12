## 将容器打包成镜像

```shell
Usage:  docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

Create a new image from a container's changes

Options:
  -a, --author string    Author (e.g., "John Hannibal Smith <hannibal@a-team.com>")
  -c, --change list      Apply Dockerfile instruction to the created image
  -m, --message string   Commit message
  -p, --pause            Pause container during commit (default true)
```

>OPTIONS说明：
>
>- **-a :**提交的镜像作者；
>- **-c :**使用Dockerfile指令来创建镜像；
>- **-m :**提交时的说明文字；
>- **-p :**在commit时，将容器暂停。



```shell
# 打包hadoop节点
sudo docker commit -a "kyle0349" -m "hadoop-hive-flink-with-single" 48223d83ef3d  registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/hadoop-hive-flink-single:20220904_1.0

# 打包zookeeper
sudo docker commit -a "kyle0349" -m "zookeeper" a6daccbd65ab  registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/zookeeper:20220904_3.4.13

# 打包mysql
sudo docker commit -a "kyle0349" -m "mysql" 9f223a50c2b7  registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/mysql:20220904_5.7

# 打包 kafka
sudo docker commit -a "kyle0349" -m "kafka" 57733996da9d  registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/kafka:20220904_2.13-2.7.1



```



## 将镜像推送到云上

```shell
# 将hadoop推送阿里云
sudo docker push registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/hadoop-hive-flink-single:20220904_1.0
# 将zookeeper推送阿里云
sudo docker push registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/zookeeper:20220904_3.4.13
# 将mysql推送阿里云
sudo docker push registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/mysql:20220904_5.7
# 将kafka推送阿里云
sudo docker push registry.cn-hangzhou.aliyuncs.com/bigdata_kyle/kafka:20220904_2.13-2.7.1



```

