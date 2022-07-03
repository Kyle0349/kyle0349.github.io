#  Flink任务的暂停和启动

## 1. Flink任务的暂停



### 1.1 直接kill任务，不管状态

### 1.2 消费kafka的Flink任务暂停并将状态保持在hdfs， 用于下次启动的基础

当遇到线上作业有需求改动的时候， 希望不要丢失数据的进行需求迭代。注意：最好在数据量低谷的时候进行，对任务造成的影响最小。

```shell
# flink cancel -s  hdfs-path 【flink-job-id】  -yid  【application-id】		
./bin/flink cancel -s  hdfs://hadoop:9000/flink/flink-job/savepoint/user_behavior/20220622/01 dd923ca530414ed6845faf3c7db8e8ec  -yid  application_1655769024051_0001

./bin/flink cancel -s  hdfs://hadoop:9000/flink/flink-job/savepoint/user_behavior/20220622/03 b6ee846a6e54cc63c56258a78cc7d799  -yid  application_1655769024051_0006

```





## 2. Flink任务的启动

### 2.1 直接启动， 无状态





### 2.2 基于之前保存在hdfs的状态启动

```shell
# 基于前面取消flink任务时保存状态在hdfs路径的flink状态启动任务
./bin/flink run \
-d \
-n \
-p 2 \
-m yarn-cluster \
-t yarn-per-job \
-yjm 2G \
-ytm 4G \
-ys 2 \
-yqu default \
-ynm "flink-job-user_behavior" \
-yD taskmanager.memory.managed.fraction=0.1 \
-s hdfs://hadoop:9000/flink/flink-job/savepoint/user_behavior/20220622/02/savepoint-44c879-d0930c583e8f \
~/job/flink/flink-1.0-SNAPSHOT.jar 
```



## 3. 有状态启动的参数发生改变后， 任务会报错， （待研究处理）

```shell



./bin/flink run \
-d \
-p 2 \
-m yarn-cluster \
-t yarn-per-job \
-yjm 2G \
-ytm 4G \
-ys 2 \
-yqu default \
-ynm "flink-job-user_behavior" \
-yD taskmanager.memory.managed.fraction=0.1 \
~/job/flink/flink-1.0-SNAPSHOT.jar 

./bin/flink cancel -s  hdfs://hadoop:9000/flink/flink-job/savepoint/user_behavior/20220703/01 7c8449a64d22ad7d957d6464306e2f57  -yid  application_1655769024051_0008

./bin/flink run \
-d \
-p 2 \
-m yarn-cluster \
-t yarn-per-job \
-yjm 2G \
-ytm 4G \
-ys 2 \
-yqu default \
-ynm "flink-job-user_behavior" \
-yD taskmanager.memory.managed.fraction=0.1 \
-s hdfs://hadoop:9000/flink/flink-job/savepoint/user_behavior/20220703/01/savepoint-7c8449-7dadf8a17889 \
~/job/flink/flink-1.0-SNAPSHOT.jar 
```





一、flink run参数：
flink run命令执行模板：flink run [option]

-c,–class : 需要指定的main方法的类

-C,–classpath : 向每个用户代码添加url，他是通过UrlClassLoader加载。url需要指定文件的schema如（file://）

-d,–detached : 在后台运行

-p,–parallelism : job需要指定env的并行度，这个一般都需要设置。

-q,–sysoutLogging : 禁止logging输出作为标准输出。

-s,–fromSavepoint : 基于savepoint保存下来的路径，进行恢复。

-sae,–shutdownOnAttachedExit : 如果是前台的方式提交，当客户端中断，集群执行的job任务也会shutdown。

二、flink run -m yarn-cluster参数
-m,–jobmanager : yarn-cluster集群
-yd,–yarndetached : 后台
-yjm,–yarnjobManager : jobmanager的内存
-ytm,–yarntaskManager : taskmanager的内存
-yn,–yarncontainer : TaskManager的个数
-yid,–yarnapplicationId : job依附的applicationId
-ynm,–yarnname : application的名称
-ys,–yarnslots : 分配的slots个数

例：flink run -m yarn-cluster -yd -yjm 1024m -ytm 1024m -ynm -ys 1

三、flink-list
flink list：列出flink的job列表。

flink list -r/–runing :列出正在运行的job

flink list -s/–scheduled :列出已调度完成的job

四、flink cancel
flink cancel [options] <job_id> : 取消正在运行的job id

flink cancel -s/–withSavepoint <job_id> ： 取消正在运行的job，并保存到相应的保存点

通过 -m 来指定要停止的 JobManager 的主机地址和端口

例： bin/flink cancel -m 127.0.0.1:8081 5e20cb6b0f357591171dfcca2eea09de

五、flink stop :仅仅针对Streaming job
flink stop [options] <job_id>

flink stop <job_id>：停止对应的job

通过 -m 来指定要停止的 JobManager 的主机地址和端口

例： bin/flink stop -m 127.0.0.1:8081 d67420e52bd051fae2fddbaa79e046bb

取消和停止（流作业）的区别如下：

cancel() 调用，立即调用作业算子的 cancel() 方法，以尽快取消它们。如果算子在接到 cancel() 调用后没有停止，Flink 将开始定期中断算子线程的执行，直到所有算子停止为止。
stop() 调用，是更优雅的停止正在运行流作业的方式。stop() 仅适用于 Source 实现了 StoppableFunction 接口的作业。当用户请求停止作业时，作业的所有 Source 都将接收 stop() 方法调用。直到所有 Source 正常关闭时，作业才会正常结束。这种方式，使作业正常处理完所有作业。
六、 flink modify 修改任务并行度
flink modify <job_id> [options]

flink modify <job_id> -p /–parallelism p : 修改job的并行度

例： flink modify -p 并行数 <job_pid>

七、flink savepoint
flink savepoint [options] <job_id>

eg: # 触发保存点

flink savepoint <job_id> hdfs://xxxx/xx/x : 将flink的快照保存到hdfs目录

使用yarn触发保存点
flink savepoint <job_id> <target_directory> -yid <application_id>

使用savepoint取消作业
flink cancel -s <tar_directory> <job_id>

从保存点恢复
flink run -s <target_directoey> [:runArgs]

如果复原的程序，对逻辑做了修改，比如删除了算子可以指定allowNonRestoredState参数复原。
flink run -s <target_directory> -n/–allowNonRestoredState [:runArgs]

savepoint 与 checkpoint 的区别

checkpoint是增量做的，每次的时间短，数据量小，只要在程序里面启用后会自动触发，用户无需感知；savepoint是全量做的，时间长，数据量大，需要用户主动触发。

checkpoint 是作业failover 的时候自动使用，不需要用户指定，savepoint 一般用于程序版本更新、bug修复、A/B Test 等场景，需要用户指定。