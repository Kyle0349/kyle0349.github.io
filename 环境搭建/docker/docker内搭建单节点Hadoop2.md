# 不带mysql





## dockerFile

```txt
#  选择centos7.7.1908作为基础镜像
FROM centos:centos7.7.1908
# 镜像维护者信息
MAINTAINER "kyle0349@126.com"
# 描述信息
LABEL name="Hadoop-Single" \
	  build_date="2022-05-07 12:00:00" \
	  wechat="17666129120" \
	  personal_site="kyle0349@126.com"
# 构建容器时需要运行的命令
# 安装openssh-server\openssh-client\sudo\vim\net-tools
RUN yum -y install openssh-server openssh-clients sudo vim net-tools ld-linux.so.2 libstdc* libstdc++-4.8.5-44.el7.i686 numactl libnuma ld-linux.so.2 libaio.so.1 libnuma.so.1 libstdc++.so.6 libinfo.so.5 libncurses.so.5 initscripts
# 时区
ENV TZ=Asia/Shanghai
RUN echo "Asia/Shanghai" > /etc/timezone
RUN ln  -sf  /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 生成相应的主机密钥文件
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
RUN ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
RUN ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
# 创建自定义组合用户，设置密码并授予rot权限
RUN groupadd -g 1124 bigdata && useradd -m -u 1124 -g bigdata kyle0349
RUN echo "kyle0349:kyle0349" | chpasswd
RUN echo "kyle0349 ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
# 创建模块和软件目录并修改权限
RUN mkdir /opt/software && mkdir /opt/moudle
# 拷贝并解压组件，将宿主机的文件拷贝到镜像（ADD 会自动解压）
ADD jdk-8u191-linux-x64.tar.gz /opt/moudle
ADD hadoop-2.7.7.tar.gz /opt/software
ADD apache-hive-1.2.1-bin.tar.gz /opt/software
ADD flink-1.12.5-bin-scala_2.12.tgz /opt/software
# 设置环境变量
ENV CENTOS_DEFAULT_HOME /root
ENV JAVA_HOME /opt/moudle/jdk1.8.0_191
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH ${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV HADOOP_HOME /opt/software/hadoop-2.7.7
ENV HIVE_HOME=/opt/software/apache-hive-1.2.1-bin
ENV MYSQL_HOME=/usr/local/mysql
ENV PATH ${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${HIVE_HOME}/bin:$PATH
# 拷贝替换配置文件
# hadoop
COPY cluster-conf/hadoop-env.sh /tmp/hadoop-env.sh
COPY cluster-conf/core-site.xml /tmp/core-site.xml
COPY cluster-conf/hdfs-site.xml /tmp/hdfs-site.xml
COPY cluster-conf/mapred-site.xml /tmp/mapred-site.xml
COPY cluster-conf/yarn-site.xml /tmp/yarn-site.xml
COPY cluster-conf/hadoop-slaves /tmp/hadoop-slaves
RUN mv /tmp/hadoop-env.sh $HADOOP_HOME/etc/hadoop/hadoop-env.sh
RUN mv /tmp/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml 
RUN mv /tmp/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml
RUN mv /tmp/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml
RUN mv /tmp/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml
RUN mv /tmp/hadoop-slaves $HADOOP_HOME/etc/hadoop/slaves
# hive
COPY cluster-conf/hive-site.xml /tmp/hive-site.xml
COPY cluster-conf/hive-env.sh /tmp/hive-env.sh
RUN mv /tmp/hive-site.xml $HIVE_HOME/conf/hive-site.xml
RUN mv /tmp/hive-env.sh $HIVE_HOME/conf/hive-env.sh
# flink
COPY cluster-conf/flink-conf.yaml /tmp/flink-conf.yaml
COPY cluster-conf/flink-master /tmp/flink-master
COPY cluster-conf/flink-worker /tmp/flink-worker
RUN mv /tmp/flink-conf.yaml /opt/software/flink-1.12.5//conf/flink-conf.yaml
RUN mv /tmp/flink-master /opt/software/flink-1.12.5/conf/master
RUN mv /tmp/flink-worker /opt/software/flink-1.12.5/conf/worker
# jar
COPY mysql-connector-java-5.1.47-bin.jar /opt/software/apache-hive-1.2.1-bin/lib/mysql-connector-java-5.1.47-bin.jar
COPY flink-shaded-hadoop-2-uber-2.7.5-10.0.jar /opt/software/flink-1.12.5/lib/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar
COPY flink-connector-jdbc_2.12-1.12.5.jar /opt/software/flink-1.12.5/lib/flink-connector-jdbc_2.12-1.12.5.jar
COPY flink-connector-mysql-cdc-1.3.0.jar /opt/software/flink-1.12.5/lib/flink-connector-mysql-cdc-1.3.0.jar
COPY flink-connector-kafka_2.12-1.12.0.jar /opt/software/flink-1.12.5/lib/flink-connector-kafka_2.12-1.12.0.jar
COPY flink-streaming-core.jar /opt/software/flink-1.12.5/lib/flink-streaming-core.jar
COPY flink-csv-1.12.5.jar /opt/software/flink-1.12.5/lib/flink-csv-1.12.5.jar
COPY flink-json-1.12.5.jar /opt/software/flink-1.12.5/lib/flink-json-1.12.5.jar
# 创建需要的文件夹
RUN mkdir /opt/software/hadoop-2.7.7/tmp
RUN mkdir -p /opt/software/hadoop-2.7.7/dfs/namenode_data
RUN mkdir -p /opt/software/hadoop-2.7.7/dfs/datanode_data
RUN mkdir -p /opt/software/apache-hive-1.2.1-bin/tmp/scratchdir
RUN mkdir -p /opt/software/apache-hive-1.2.1-bin/tmp/hive_resources
# 修改ADD进去并解压的文件目录
RUN chown -R kyle0349:bigdata /opt/moudle && chown -R kyle0349:bigdata /opt/software

# 终端默认登录进来的工作目录
WORKDIR $CENTOS_DEFAULT_HOME
# 启动sshd服务并暴露22端口
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```



```shell
# 构建镜像
sudo docker build -f hadoop-2.7.7-Dockerfile -t kyle0349/hadoop-hive-flink:2.7.7 .

# docker 网络设置
# 创建自定义网络
# 默认情况下启动的Docker容器，都是使用bridge，Docker安装时创建的桥接网络，
# 每次Docker容器重启时，会按照顺序获取对应的ip地址，这个就导致重启后容器的ip地址变化了，因此需要创建自定义网络。
# 创建子网
sudo  docker network create --subnet=172.22.0.0/24 mynetwork

# mysql 
docker run -p 3306:3306 --name mysql \
-v /usr/local/docker/mysql/conf:/etc/mysql \
-v /usr/local/docker/mysql/logs:/var/log/mysql \
-v /usr/local/docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
--net mynetwork \
--ip 172.22.0.6 \
-d mysql:5.7

# zookeeper
docker run -d --name zookeeper --net mynetwork --ip 172.22.0.7 -p 2181:2181 -t wurstmeister/zookeeper

# hadoop
sudo docker run \
-d \
--privileged \
--name hadoop-hive-flink \
--hostname hadoop \
-P \
-p 50070:50070 \
-p 8088:8088 \
-p 19888:19888 \
-p 8041:8041 \
-p 8042:8042 \
-p 8081:8081 \
-p 9000:9000 \
-p 50020:50020 \
-p 50075:50075 \
-p 50010:50010 \
-p 8021:8021 \
-p 50030:50030 \
-p 50060:50060 \
-p 50090:50090 \
-p 10000:10000 \
-p 9083:9083 \
--net mynetwork \
--ip 172.22.0.5 \
3b1b9084f848

## 免密设置
ssh-keygen -t rsa -C "kyle0349@126.com"
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop


# 格式化namenode
hdfs namenode -format
# 启动hadoop
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver

### 初始化hive数据库
schematool -dbType mysql -initSchema
### 测试Hive
create table t_test(id int,tel string) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;
1,hello
2,world
3,kyle
4,susu
load data local inpath '/home/kyle0349/test.txt' into table t_test;
select count(1) from t_test;

### 测试flink
/opt/software/flink-1.12.5/bin/flink run -t yarn-per-job /opt/software/flink-1.12.5/examples/streaming/WordCount.jar 


### kafka
# kafka不能用子网的形式，不然本地idea的flink程序消费不到kafka， 有待研究
docker run --name kafka \
-p 9092:9092 \
-e KAFKA_BROKER_ID=0 \
-e KAFKA_ZOOKEEPER_CONNECT=192.168.1.100:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.1.100:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
-d  wurstmeister/kafka 


### pgsql
docker run -d \
--name pgsql \
-p 5432:5432 \
-e POSTGRES_PASSWORD=root \
07e2ee723e2d


docker run -d \
--name airflow  \
-v /Users/kyle/Documents/software/docker/airflow/dags:/usr/local/airflow/dags \
-v /Users/kyle/Documents/software/docker/airflow/airflowSql:/usr/local/airflow/airflowSql  \
-v /Users/kyle/Documents/software/docker/airflow/cfg:/usr/local/airflow \
-p 8080:8080 \
--net mynetwork \
--ip 172.22.0.10 \
puckel/docker-airflow 

```





