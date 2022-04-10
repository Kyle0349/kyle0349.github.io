# Flink的部署方式

## 1. Local

> 不单独部署运行环境，在代码中直接调试。



## 2. Standalone

> 独立运行环境，Flink自己完全管理运行资源。这种模式的资源利用比较低，在大数据集群中不方便与其他计算资源进行合理分配。

### 2.1、 集群部署

#### 2.1.1、首先要有一个在运行的hadoop-yarn集群（已经事先搭建好）

#### 2.1.2、Flink的搭建

1. 在Flink官网下载合适的Flink安装包(https://flink.apache.org/downloads.html)，这里我们选择1.12.5版本

   > Scala是基于JVM执行的一种语言，可以理解为跟java很类似的，但是它的版本兼容性没有java好，所有经常需要区分不同的版本。但是编译后都是提交到JVM上运行的。在这里下载的运行版本都是scala编译后的执行文件， 所以跟scala语言没有太多关系。可以简单选择flink-1.12.5-bin-scala_2.12.tgz即可

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h15emf5f5bj214i0n6dks.jpg" alt="flink1.12.5" style="zoom:50%;" />

2. 因为flink需要和Hadoop交互，所以也需要下载Hadoop的插件(https://flink.apache.org/downloads.html#all-stable-releases)这里选择2.6.5版本的。

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h15etzpvboj21fw0dsgnz.jpg" alt="flink1.12.5" style="zoom:40%;" />



3. 将Flink安装包解压到/opt/flink-1.12.5目录，如下，

   1. lib目录是flink的核心jar包，其中也有阿里贡献的blink的相关jar包。

   2. opt目录下是flink的一些扩展jar包。

   3. example下包含了可直接执行的flink示例。

   4. conf是Flink配置文件相关的目录（重点关注）

   5. bin是Flink启动脚本相关的目录（重点关注）

      <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h15exs32i8j20xo0dq78w.jpg" alt="flink1.12.5" style="zoom:50%;" />

4. 修改操作

   1. 将上面下载的【flink-shaded-hadoop-2-uber-2.6.5-10.0.jar】拷贝到 /opt/flink-1.12.5/lib目录下

   2. 修改conf目录下的flink-conf.yaml文件

      ```bash
      # Common
      # 指定jobmanager地址 
      jobmanager.rpc.address: cdh01
      # High Avaibility 
      # 指定使用zookeeper进行高可用部署 
      high-availability: zookeeper 
      # 元数据存储地址。这个地址是需要所有节点都能访问到的一个地址。 
      high-availability.storageDir: hdfs:///flink/ha/
      # 指定外部zookeeper节点，多个节点使用【,】隔开
      high-availability.zookeeper.quorum: 192.168.31.151:2181
      ```

   3. 修改masters文件

      cdh01:8081

   4. 修改workers文件

      cdh01

#### 2.1.3、Flink启动

1. 启动上面 flink-conf.yaml 文件中配置的 hadoop 和 zookeeper。

2. 在  /opt/flink-1.12.5/bin目录下执行 【start-cluster.sh】脚本

3. 执行jps, 可以查看到新增了 StandaloneSessionClusterEntrypoint 和  TaskManagerRunner 进程

   ```bash
   [root@cdh01 flink-1.12.5]# jps
   25312 JobHistoryServer
   6786 NameNode
   72226 Jps
   25540 NodeManager
   72077 TaskManagerRunner
   6703 SecondaryNameNode
   6543 QuorumPeerMain
   25398 ResourceManager
   26042 RunJar
   6618 DataNode
   2907 Main
   7516 HMaster
   71773 StandaloneSessionClusterEntrypoint
   7485 HRegionServer
   26078 RunJar
   [root@cdh01 flink-1.12.5]#
   ```

4. 启动完成后， 可以查看flink提供的管理页面 http://192.168.31.100:8081/#/overview

   > 在这个界面可以查看集群的相关状态。Available Task Slot是比较关键的参数，Task Slot是Flink执行具体任务的单元， 如果slots数量不够，Flink任务无法执行，在standalone的模式下， Available Task Slot可以在 flink-conf.yaml 文件配置。

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h15fhl4ciuj21l00u0tb2.jpg" alt="flink1.12.5" style="zoom:30%;" />

   





## 3. Yarn

> 1、以Hadoop提供的Yarn作为资源管理服务，对Flink的计算资源进行管理，达到分配和回收的效果，与集群中其他的计算资源共同管理，可以更高效的利用集群的机器资源。
>
> 2、上面standalone模式的配置中，可直接复用到Yarn模式下， 只是启动的方式不同。

### 3.1 Session Mode 会话模式

> 在yarn上申请一个固定的flink集群，然后所有的任务都共享这个集群内的资源，

### 3.2 Application Mode 应用模式



### 3.3 Per-job Cluster Mode 单任务模式







