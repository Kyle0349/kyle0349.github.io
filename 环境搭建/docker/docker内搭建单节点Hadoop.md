# dockerFile搭建单节点Hadoop



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
# 将宿主机的文件拷贝到镜像（ADD 会自动解压）
ADD jdk-8u191-linux-x64.tar.gz /opt/moudle
ADD hadoop-2.7.7.tar.gz /opt/software
ADD apache-hive-1.2.1-bin.tar.gz /opt/software
ADD flink-1.12.5-bin-scala_2.12.tgz /opt/software
ADD mysql-5.7.10-linux-glibc2.5-i686.tar.gz /usr/local
COPY copyright.txt /home/kyle0349/copyright.txt
COPY mysql-connector-java-5.1.47-bin.jar /opt/software/apache-hive-1.2.1-bin/lib/mysql-connector-java-5.1.47-bin.jar
COPY flink-shaded-hadoop-2-uber-2.7.5-10.0.jar /opt/software/flink-1.12.5/lib/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar
COPY flink-connector-jdbc_2.12-1.12.5.jar /opt/software/flink-1.12.5/lib/flink-connector-jdbc_2.12-1.12.5.jar
copy flink-connector-mysql-cdc-1.3.0.jar /opt/software/flink-1.12.5/lib/flink-connector-mysql-cdc-1.3.0.jar
# 创建需要的文件夹
RUN mkdir /opt/software/hadoop-2.7.7/tmp
RUN mkdir -p /opt/software/hadoop-2.7.7/dfs/namenode_data
RUN mkdir -p /opt/software/hadoop-2.7.7/dfs/datanode_data
RUN mkdir -p /opt/software/apache-hive-1.2.1-bin/tmp/scratchdir
RUN mkdir -p /opt/software/apache-hive-1.2.1-bin/tmp/hive_resources
# 修改ADD进去并解压的文件目录
RUN chown -R kyle0349:bigdata /opt/moudle && chown -R kyle0349:bigdata /opt/software
# 安装mysql (mysql无法自动启动)
RUN mv /usr/local/mysql-5.7.10-linux-glibc2.5-i686 /usr/local/mysql
RUN mkdir /usr/local/mysql/data
RUN groupadd mysql && useradd mysql -g mysql 
RUN chown -R mysql:mysql /usr/local/mysql
RUN /usr/local/mysql/bin/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
RUN mkdir -p /var/lib/mysql
RUN chown mysql:mysql /var/lib/mysql
RUN cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
RUN chkconfig mysql on
# 设置环境变量
ENV CENTOS_DEFAULT_HOME /root
ENV JAVA_HOME /opt/moudle/jdk1.8.0_191
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH ${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV HADOOP_HOME /opt/software/hadoop-2.7.7
ENV HIVE_HOME=/opt/software/apache-hive-1.2.1-bin
ENV MYSQL_HOME=/usr/local/mysql
ENV PATH ${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${HIVE_HOME}/bin:${MYSQL_HOME}/bin:$PATH
# 终端默认登录进来的工作目录
WORKDIR $CENTOS_DEFAULT_HOME
# 启动sshd服务并暴露22端口
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```



## 构建镜像
sudo docker build -f hadoop-2.7.7-Dockerfile -t kyle0349/hadoop-single:2.7.7 .

sudo docker build -f hadoop-2.7.7-Dockerfile -t kyle0349/hadoop-hive-flink:2.7.7 .

##创建自定义网络
默认情况下启动的Docker容器，都是使用bridge，Docker安装时创建的桥接网络，每次Docker容器重启时，会按照顺序获取对应的ip地址，这个就导致重启后容器的ip地址变化了，因此需要创建自定义网络。
 查看
sudo  docker network inspect a32cbc08f949

### 创建子网
sudo  docker network create --subnet=172.22.0.0/24 mynetwork



## 创建并启动一个容器

```shell
#####
sudo docker run \
-d \
--privileged \
--name hadoop-hive-flink \
--hostname hadoop \
-P \
-p 50070:50070 \
-p 8088:8088 \
-p 19888:19888 \
-p 8042:8042 \
-p 3306:3306 \
-p 8081:8081 \
--net mynetwork \
--ip 172.22.0.5 \
3042a220af60 
```



## 进入容器
sudo docker exec -it 77aef8c1db52 /bin/bash

## 免密设置
ssh-keygen -t rsa -C "kyle0349@126.com"
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop



## 配置mysql











## 修改集群配置文件



### Hadoop

hadoop-env.sh

```sh
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Set Hadoop-specific environment variables here.

# The only required environment variable is JAVA_HOME.  All others are
# optional.  When running a distributed configuration it is best to
# set JAVA_HOME in this file, so that it is correctly defined on
# remote nodes.

# The java implementation to use.
export JAVA_HOME=/opt/moudle/jdk1.8.0_191


# The jsvc implementation to use. Jsvc is required to run secure datanodes
# that bind to privileged ports to provide authentication of data transfer
# protocol.  Jsvc is not required if SASL is configured for authentication of
# data transfer protocol using non-privileged ports.
#export JSVC_HOME=${JSVC_HOME}

export HADOOP_CONF_DIR=/opt/software/hadoop-2.7.7/etc/hadoop

# Extra Java CLASSPATH elements.  Automatically insert capacity-scheduler.
for f in $HADOOP_HOME/contrib/capacity-scheduler/*.jar; do
  if [ "$HADOOP_CLASSPATH" ]; then
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$f
  else
    export HADOOP_CLASSPATH=$f
  fi
done

# The maximum amount of heap to use, in MB. Default is 1000.
#export HADOOP_HEAPSIZE=
#export HADOOP_NAMENODE_INIT_HEAPSIZE=""

# Extra Java runtime options.  Empty by default.
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"

# Command specific options appended to HADOOP_OPTS when specified
export HADOOP_NAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_NAMENODE_OPTS"
export HADOOP_DATANODE_OPTS="-Dhadoop.security.logger=ERROR,RFAS $HADOOP_DATANODE_OPTS"

export HADOOP_SECONDARYNAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,NullAppender} $HADOOP_SECONDARYNAMENODE_OPTS"

export HADOOP_NFS3_OPTS="$HADOOP_NFS3_OPTS"
export HADOOP_PORTMAP_OPTS="-Xmx512m $HADOOP_PORTMAP_OPTS"

# The following applies to multiple commands (fs, dfs, fsck, distcp etc)
export HADOOP_CLIENT_OPTS="-Xmx512m $HADOOP_CLIENT_OPTS"
#HADOOP_JAVA_PLATFORM_OPTS="-XX:-UsePerfData $HADOOP_JAVA_PLATFORM_OPTS"

# On secure datanodes, user to run the datanode as after dropping privileges.
# This **MUST** be uncommented to enable secure HDFS if using privileged ports
# to provide authentication of data transfer protocol.  This **MUST NOT** be
# defined if SASL is configured for authentication of data transfer protocol
# using non-privileged ports.
export HADOOP_SECURE_DN_USER=${HADOOP_SECURE_DN_USER}

# Where log files are stored.  $HADOOP_HOME/logs by default.
#export HADOOP_LOG_DIR=${HADOOP_LOG_DIR}/$USER

# Where log files are stored in the secure data environment.
export HADOOP_SECURE_DN_LOG_DIR=${HADOOP_LOG_DIR}/${HADOOP_HDFS_USER}

###
# HDFS Mover specific parameters
###
# Specify the JVM options to be used when starting the HDFS Mover.
# These options will be appended to the options specified as HADOOP_OPTS
# and therefore may override any similar flags set in HADOOP_OPTS
#
# export HADOOP_MOVER_OPTS=""

###
# Advanced Users Only!
###

# The directory where pid files are stored. /tmp by default.
# NOTE: this should be set to a directory that can only be written to by 
#       the user that will run the hadoop daemons.  Otherwise there is the
#       potential for a symlink attack.
export HADOOP_PID_DIR=${HADOOP_PID_DIR}
export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR}

# A string representing this instance of hadoop. $USER by default.
export HADOOP_IDENT_STRING=$USER
```



core-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop:9000</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/software/hadoop-2.7.7/tmp</value>
</property>
</configuration>
```





hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>
<property>
    <name>dfs.name.dir</name>
    <value>/opt/software/hadoop-2.7.7/dfs/namenode_data</value>
</property>
<property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/software/hadoop-2.7.7/dfs/datanode_data</value>
</property>
<property>
    <name>dfs.permission</name>
    <value>false</value>
</property>
</configuration>
```





mapred-site.xml 

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoop:10020</value>
</property>
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop:19888</value>
</property>
<property>
    <name>mapreduce.map.memory.mb</name>
    <value>512</value>
</property>

<property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>512</value>
</property>
<property>
    <name>mapreduce.map.java.opts</name>
    <value>-Xmx1200m</value>
</property>

<property>
    <name>mapreduce.reduce.java.opts</name>
    <value>-Xmx2600m</value>
</property>
</configuration>
```





yarn-site.xml

```xml
<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<configuration>

<!-- Site specific YARN configuration properties -->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoop</value>
</property>
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>
<property>
    <name>yarn.log-aggregation-retain-seconds</name>
    <value>604800</value>
</property>
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>10384</value>
</property>
<property>
   <name>yarn.scheduler.minimum-allocation-mb</name>
   <value>1024</value>
</property>
<property>
    <name>yarn.nodemanager.vmem-pmem-ratio</name>
    <value>3</value>
</property>
</configuration>
```



### Hive

hive-env.sh

```sh
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Set Hive and Hadoop environment variables here. These variables can be used
# to control the execution of Hive. It should be used by admins to configure
# the Hive installation (so that users do not have to set environment variables
# or set command line parameters to get correct behavior).
#
# The hive service being invoked (CLI/HWI etc.) is available via the environment
# variable SERVICE


# Hive Client memory usage can be an issue if a large number of clients
# are running at the same time. The flags below have been useful in 
# reducing memory usage:
#
# if [ "$SERVICE" = "cli" ]; then
#   if [ -z "$DEBUG" ]; then
#     export HADOOP_OPTS="$HADOOP_OPTS -XX:NewRatio=12 -Xms10m -XX:MaxHeapFreeRatio=40 -XX:MinHeapFreeRatio=15 -XX:+UseParNewGC -XX:-UseGCOverheadLimit"
#   else
#     export HADOOP_OPTS="$HADOOP_OPTS -XX:NewRatio=12 -Xms10m -XX:MaxHeapFreeRatio=40 -XX:MinHeapFreeRatio=15 -XX:-UseGCOverheadLimit"
#   fi
# fi

# The heap size of the jvm stared by hive shell script can be controlled via:
#
# export HADOOP_HEAPSIZE=1024
#
# Larger heap size may be required when running queries over large number of files or partitions. 
# By default hive shell scripts use a heap size of 256 (MB).  Larger heap size would also be 
# appropriate for hive server (hwi etc).


# Set HADOOP_HOME to point to a specific hadoop install directory
# HADOOP_HOME=${bin}/../../hadoop

# Hive Configuration Directory can be controlled by:
# export HIVE_CONF_DIR=

# Folder containing extra ibraries required for hive compilation/execution can be controlled by:
# export HIVE_AUX_JARS_PATH=

export JAVA_HOME=/opt/moudle/jdk1.8.0_191
export HADOOP_HOME=/opt/software/hadoop-2.7.7
export HIVE_HOME=/opt/software/apache-hive-1.2.1-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
```



hive-site.xml 

> javax.jdo.option.ConnectionURL 使用 mysql 是因为关联了另外一个docker容器

```xml
<configuration>
  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/opt/software/apache-hive-1.2.1-bin/tmp/scratchdir</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/opt/software/apache-hive-1.2.1-bin/tmp/hive_resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root</value>
    <description>password to use against metastore database</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://mysql:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;characterEncoding=UTF-8</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>Username to use against metastore database</description>
  </property>
  <property>
    <name>hive.querylog.location</name>
    <value>/opt/software/apache-hive-1.2.1-bin/tmp/${system:user.name}</value>
    <description>Location of Hive run time structured log file</description>
  </property>
  <property>
    <name>hive.server2.logging.operation.log.location</name>
    <value>/opt/software/apache-hive-1.2.1-bin/tmp/${system:user.name}/operation_logs</value>
    <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
<property>
    <name>system:java.io.tmpdir</name>
    <value>/opt/software/apache-hive-2.3.5-bin/hive_tmp/</value>
</property>
<property>
    <name>system:user.name</name>
    <value>hive</value>
</property>
</configuration>
```



### Flink

1. 将上面下载的【flink-shaded-hadoop-2-uber-2.6.5-10.0.jar】拷贝到 /opt/flink-1.12.5/lib目录下

2. 修改conf目录下的flink-conf.yaml文件

   ```bash
   ################################################################################
   #  Licensed to the Apache Software Foundation (ASF) under one
   #  or more contributor license agreements.  See the NOTICE file
   #  distributed with this work for additional information
   #  regarding copyright ownership.  The ASF licenses this file
   #  to you under the Apache License, Version 2.0 (the
   #  "License"); you may not use this file except in compliance
   #  with the License.  You may obtain a copy of the License at
   #
   #      http://www.apache.org/licenses/LICENSE-2.0
   #
   #  Unless required by applicable law or agreed to in writing, software
   #  distributed under the License is distributed on an "AS IS" BASIS,
   #  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   #  See the License for the specific language governing permissions and
   # limitations under the License.
   ################################################################################
   
   
   #==============================================================================
   # Common
   #==============================================================================
   
   # The external address of the host on which the JobManager runs and can be
   # reached by the TaskManagers and any clients which want to connect. This setting
   # is only used in Standalone mode and may be overwritten on the JobManager side
   # by specifying the --host <hostname> parameter of the bin/jobmanager.sh executable.
   # In high availability mode, if you use the bin/start-cluster.sh script and setup
   # the conf/masters file, this will be taken care of automatically. Yarn/Mesos
   # automatically configure the host name based on the hostname of the node where the
   # JobManager runs.
   
   jobmanager.rpc.address: hadoop
   
   # The RPC port where the JobManager is reachable.
   
   jobmanager.rpc.port: 6123
   
   
   # The total process memory size for the JobManager.
   #
   # Note this accounts for all memory usage within the JobManager process, including JVM metaspace and other overhead.
   
   jobmanager.memory.process.size: 1600m
   
   
   # The total process memory size for the TaskManager.
   #
   # Note this accounts for all memory usage within the TaskManager process, including JVM metaspace and other overhead.
   
   taskmanager.memory.process.size: 1728m
   
   # To exclude JVM metaspace and overhead, please, use total Flink memory size instead of 'taskmanager.memory.process.size'.
   # It is not recommended to set both 'taskmanager.memory.process.size' and Flink memory.
   #
   # taskmanager.memory.flink.size: 1280m
   
   # The number of task slots that each TaskManager offers. Each slot runs one parallel pipeline.
   
   taskmanager.numberOfTaskSlots: 1
   
   # The parallelism used for programs that did not specify and other parallelism.
   
   parallelism.default: 1
   
   # The default file system scheme and authority.
   # 
   # By default file paths without scheme are interpreted relative to the local
   # root file system 'file:///'. Use this to override the default and interpret
   # relative paths relative to a different file system,
   # for example 'hdfs://mynamenode:12345'
   #
   # fs.default-scheme
   
   #==============================================================================
   # High Availability
   #==============================================================================
   
   # The high-availability mode. Possible options are 'NONE' or 'zookeeper'.
   #
   high-availability: zookeeper
   
   # The path where metadata for master recovery is persisted. While ZooKeeper stores
   # the small ground truth for checkpoint and leader election, this location stores
   # the larger objects, like persisted dataflow graphs.
   # 
   # Must be a durable file system that is accessible from all nodes
   # (like HDFS, S3, Ceph, nfs, ...) 
   #
   high-availability.storageDir: hdfs://hadoop:9000/flink/ha/
   
   # The list of ZooKeeper quorum peers that coordinate the high-availability
   # setup. This must be a list of the form:
   # "host1:clientPort,host2:clientPort,..." (default clientPort: 2181)
   #
   high-availability.zookeeper.quorum: zookeeper:2181
   
   
   # ACL options are based on https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#sc_BuiltinACLSchemes
   # It can be either "creator" (ZOO_CREATE_ALL_ACL) or "open" (ZOO_OPEN_ACL_UNSAFE)
   # The default value is "open" and it can be changed to "creator" if ZK security is enabled
   #
   # high-availability.zookeeper.client.acl: open
   
   #==============================================================================
   # Fault tolerance and checkpointing
   #==============================================================================
   
   # The backend that will be used to store operator state checkpoints if
   # checkpointing is enabled.
   #
   # Supported backends are 'jobmanager', 'filesystem', 'rocksdb', or the
   # <class-name-of-factory>.
   #
   # state.backend: filesystem
   
   # Directory for checkpoints filesystem, when using any of the default bundled
   # state backends.
   #
   # state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints
   
   # Default target directory for savepoints, optional.
   #
   # state.savepoints.dir: hdfs://namenode-host:port/flink-savepoints
   
   # Flag to enable/disable incremental checkpoints for backends that
   # support incremental checkpoints (like the RocksDB state backend). 
   #
   # state.backend.incremental: false
   
   # The failover strategy, i.e., how the job computation recovers from task failures.
   # Only restart tasks that may have been affected by the task failure, which typically includes
   # downstream tasks and potentially upstream tasks if their produced data is no longer available for consumption.
   
   jobmanager.execution.failover-strategy: region
   
   #==============================================================================
   # Rest & web frontend
   #==============================================================================
   
   # The port to which the REST client connects to. If rest.bind-port has
   # not been specified, then the server will bind to this port as well.
   #
   #rest.port: 8081
   
   # The address to which the REST client will connect to
   #
   #rest.address: 0.0.0.0
   
   # Port range for the REST and web server to bind to.
   #
   #rest.bind-port: 8080-8090
   
   # The address that the REST & web server binds to
   #
   #rest.bind-address: 0.0.0.0
   
   # Flag to specify whether job submission is enabled from the web-based
   # runtime monitor. Uncomment to disable.
   
   #web.submit.enable: false
   
   #==============================================================================
   # Advanced
   #==============================================================================
   
   # Override the directories for temporary files. If not specified, the
   # system-specific Java temporary directory (java.io.tmpdir property) is taken.
   #
   # For framework setups on Yarn or Mesos, Flink will automatically pick up the
   # containers' temp directories without any need for configuration.
   #
   # Add a delimited list for multiple directories, using the system directory
   # delimiter (colon ':' on unix) or a comma, e.g.:
   #     /data1/tmp:/data2/tmp:/data3/tmp
   #
   # Note: Each directory entry is read from and written to by a different I/O
   # thread. You can include the same directory multiple times in order to create
   # multiple I/O threads against that directory. This is for example relevant for
   # high-throughput RAIDs.
   #
   # io.tmp.dirs: /tmp
   
   # The classloading resolve order. Possible values are 'child-first' (Flink's default)
   # and 'parent-first' (Java's default).
   #
   # Child first classloading allows users to use different dependency/library
   # versions in their application than those in the classpath. Switching back
   # to 'parent-first' may help with debugging dependency issues.
   #
   # classloader.resolve-order: child-first
   
   # The amount of memory going to the network stack. These numbers usually need 
   # no tuning. Adjusting them may be necessary in case of an "Insufficient number
   # of network buffers" error. The default min is 64MB, the default max is 1GB.
   # 
   # taskmanager.memory.network.fraction: 0.1
   # taskmanager.memory.network.min: 64mb
   # taskmanager.memory.network.max: 1gb
   
   #==============================================================================
   # Flink Cluster Security Configuration
   #==============================================================================
   
   # Kerberos authentication for various components - Hadoop, ZooKeeper, and connectors -
   # may be enabled in four steps:
   # 1. configure the local krb5.conf file
   # 2. provide Kerberos credentials (either a keytab or a ticket cache w/ kinit)
   # 3. make the credentials available to various JAAS login contexts
   # 4. configure the connector to use JAAS/SASL
   
   # The below configure how Kerberos credentials are provided. A keytab will be used instead of
   # a ticket cache if the keytab path and principal are set.
   
   # security.kerberos.login.use-ticket-cache: true
   # security.kerberos.login.keytab: /path/to/kerberos/keytab
   # security.kerberos.login.principal: flink-user
   
   # The configuration below defines which JAAS login contexts
   
   # security.kerberos.login.contexts: Client,KafkaClient
   
   #==============================================================================
   # ZK Security Configuration
   #==============================================================================
   
   # Below configurations are applicable if ZK ensemble is configured for security
   
   # Override below configuration to provide custom ZK service name if configured
   # zookeeper.sasl.service-name: zookeeper
   
   # The configuration below must match one of the values set in "security.kerberos.login.contexts"
   # zookeeper.sasl.login-context-name: Client
   
   #==============================================================================
   # HistoryServer
   #==============================================================================
   
   # The HistoryServer is started and stopped via bin/historyserver.sh (start|stop)
   
   # Directory to upload completed jobs to. Add this directory to the list of
   # monitored directories of the HistoryServer as well (see below).
   #jobmanager.archive.fs.dir: hdfs:///completed-jobs/
   
   # The address under which the web-based HistoryServer listens.
   #historyserver.web.address: 0.0.0.0
   
   # The port under which the web-based HistoryServer listens.
   #historyserver.web.port: 8082
   
   # Comma separated list of directories to monitor for completed jobs.
   #historyserver.archive.fs.dir: hdfs:///completed-jobs/
   
   # Interval in milliseconds for refreshing the monitored directories.
   #historyserver.archive.fs.refresh-interval: 10000
   ```

3. 修改masters文件

   hadoop:8081

4. 修改workers文件

   hadoop



## 启动

### 格式化namenode
hdfs namenode -format

### 启动hadoop
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver

### 查看进程
jps

### 初始化hive数据库
schematool -dbType mysql -initSchema

### 测试Hive
create table t_test(id int,tel string) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;
1,hello
2,world
3,kyle0349
load data local inpath '/home/kyle0349/test.txt' into table t_test;





## 将运行中的容器打成镜像
sudo docker commit -a "kyle0349" -m "hadoop-hive-flink-with-mysql-single" 9c202662c09f kyle0349/hadoop-hive-flink-with-mysql-single-1.0


## 备份镜像
sudo docker save -0 /home/hadoop-hive-flink-with-mysql-single-1.0.tar.gz kyle0349/hadoop-hive-flink-with-mysql-single-1.0

