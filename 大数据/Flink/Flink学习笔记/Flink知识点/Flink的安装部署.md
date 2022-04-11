# Flink的部署方式和提交任务

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

> 在yarn上申请一个固定的flink集群，所有的任务都共享这个集群内的资源，

#### 3.1.1 启动yarn-session

1. 在  /opt/flink-1.12.5/bin目录下执行 【yarn-session.sh】脚本

   ```bash
   # --detached 转变为 detached mode 解除绑定模式。本地客户端提交任务到集群后，就会释放本地客户端的窗口。
   bin/yarn-session.sh --detached
   
   # 使用 --detached后， 启动日志里面会给到停止集群的命令
   # The Flink YARN session cluster has been started in detached mode. In order to stop Flink gracefully, use the following command:
   $ echo "stop" | ./bin/yarn-session.sh -id application_1649144595349_0005
   If this should not be possible, then you can also kill Flink via YARN's web interface or via:
   $ yarn application -kill application_1649144595349_0005
   Note that killing Flink might not clean up all job artifacts and temporary files.
   ```

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16jxfnl45j21no0js170.jpg" alt="flink1.12.5" style="zoom:40%;" />

   

2. 在日志里面也可以看到有Flink管理页面的地址， 端口每次启动都会改变，

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16k6c919hj21oe0futim.jpg" alt="image-20220412065623274" style="zoom:40%;" />

   

3. 在yarn上面可查看到对应的任务是running状态 http://192.168.31.100:8088/cluster

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16k3igv4jj22g80byju7.jpg" alt="image-20220412065335519" style="zoom:40%;" />

4. 可以在同一个yarn集群里面启动多个flink的session mode

   ```bash
   # 因为 detached 是解除绑定模式， 启动完第一个flink后， 在同一个窗口马上重复执行以下命令，就可以启动第二个flink session
   bin/yarn-session.sh --detached
   ```

   > 如下yarn上面的两个flink session cluster都处于running状态， 它们的application id是不同的。

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16kibc0svj22ga0hs42s.jpg" alt="image-20220412070753115" style="zoom:40%;" />



#### 3.1.2 session mode下提交任务

1. 提交任务

   >1、参数
   >
   >-t: 指定是使用yarn session模式提交任务
   >
   >-Dyarn.application.id: 通过这个参数指定yarn上面的flink session cluster。因为上面说到可以启动多个flink session cluster
   >
   >2、提交任务后，对应的flink session clustrer任务会获得相应的cpu和内存资源，任务执行完后，会重新释放

   ```bash
   # 使用flink自带的wordcount示例
   ./bin/flink run -t yarn-session -Dyarn.application.id=application_1649144595349_0005 ./examples/streaming/WordCount.jar 
   ```

2. 查看提交的任务

   > 在yarn上面可以看到申请的资源数量

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16ks9m7zaj22fy0giae9.jpg" alt="image-20220412071727087" style="zoom:40%;" />

   >在flink 的管理页面上可以看到任务执行时候的jobs，

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16l06ofqpj22a00bw756.jpg" alt="image-20220412072502557" style="zoom:40%;" />

   >任务执行完后，任务申请的资源会回归到 flink的 Available Task Slots 

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16l0mh4rlj21xo0ai3z8.jpg" alt="image-20220412072528875" style="zoom:40%;" />

      >过一段时间没有任务执行， flink session cluster会将这些可用资源重新释放给yarn

   <img src="https://tva1.sinaimg.cn/large/e6c9d24egy1h16l9q9tymj22260aagme.jpg" alt="image-20220412073415568" style="zoom:40%;" />

   

### 3.2 Application Mode 应用模式



### 3.3 Per-job Cluster Mode 单任务模式







