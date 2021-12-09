# Flink学习笔记-B站尚硅谷



> 学习Flink

# 1.Flink的特性

+ 事件驱动

  + Flink 是基于**事件**驱动的，事件可以理解为消息。事件驱动的应用程序是一种状态应用程序，它会从一个或者多个流中注入事件，通过触发计算更新状态，或外部动作对注入的事件作出反应

+ Flink擅长处理无界流的数据，同时可以以流的方式兼容批处理。（批流统一）
  + 有界流（批处理）

    > <font size=2>在以往的数据处理过程中，我们大多按照一定周期（小时，天，周）将数据从mysql通过sqoop这类工具同步到hive，然后再对数据进行逻辑处理，那么我们在处理这些数据的时候，这些数据是**静态**的（没有新增）。这样的数据就是我们说的有界数据集，也就是有一定的时间边界，这样的处理方式也就是我们常说的批处理（批计算）</font>

    1. 有定义流的开始，也有定义流的结束。
    2. 一次读取完当前边界内所有数据后再进行计算。
    3. 所有数据可以被排序，不需要有序读取。

  + 无界流（流处理）

    > <font size=2>跟**有界流**相对比，无界流可以理解为数据是没有时间边界的，也就是没有了按周期同步的这一过程，其中的组件也切换成我们常见的消息流处理组件kafka。无界数据集是会发生持续变更的、连续追加的，即我们在处理数据的时候，数据是**动态**的。处理这样的数据流方式就是我们常说的流处理（流计算）</font>
  
    1. 有定义流的开始，但没有定义流的结束。
    2. 处理时大多以特定顺序读取处理事件，例如事件发生的顺序，以便能够推断结果的完整性。
  
    
  
    

## 1.1 Flink vs Spark Streaming （思想）

+ Spark Streaming

  Spark Streaming是Spark为了流式处场景推出的方案，思想是将时间无限缩小，达到感知上的流处理，本质是微批处理，因为Spark的RDD模型决定对数据形成一个个集合进行批处理，Spark Streaming底层也是Spark Core的RDD模型，所以Spark Streaming的DStream实际就是一个个小批数据RDD集合。

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gvypxt24qlj310006qgm8.jpg" alt="spark streaming dstream" style="zoom:70%;" />

+ Flink

  Flink 是基于**事件**驱动的，事件可以理解为消息。事件驱动的应用程序是一种状态应用程序，它会从一个或者多个流中注入事件，通过触发计算更新状态，或外部动作对注入的事件作出反应。

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gvypx01ny3j30mm09cwf5.jpg" style="zoom:80%;" />





# 2.idea快速搭建Flink Maven项目

pom.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.kyle</groupId>
    <artifactId>learn</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <flink.version>1.12.5</flink.version>
        <scala.binary.version>2.12</scala.binary.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
    </dependencies>

</project>
```

## 2.1 批处理方式实现worlfdCount

> Flink提供了很多直接读取文件的API对数据进行批处理，代码示例使用【readTextFile】api
>
> Flink对接数据源API：链接

java代码

```java
package com.day1.wc;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.operators.AggregateOperator;
import org.apache.flink.api.java.operators.DataSource;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.util.Collector;

public class WordCount {

    public static void main(String[] args) throws Exception {
        // 创建执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        // 从文件中读取数据
        String inputPath = "/Users/kyle/Documents/kyle/project/flink/learn/src/main/resources/hello.txt";

        DataSource<String> inputDataSet = env.readTextFile(inputPath);

        // 对数据集进行处理, 按空格分词展开，转换成（word, 1）这样的元祖进行统计
        AggregateOperator<Tuple2<String, Integer>> resultSet = inputDataSet.flatMap(new MyFlatMapper())
                .groupBy(0) // 按照第一个位置的word分组
                .sum(1);// 将第二个位置上的数据求和

        resultSet.print();

    }

    public static class MyFlatMapper implements FlatMapFunction<String, Tuple2<String, Integer>>{

        @Override
        public void flatMap(String value, Collector<Tuple2<String, Integer>> collector) throws Exception {
            // 按空格分词
            String[] words = value.split(" ");
            // 遍历所有word， 包装二元组输出
            for (String word : words) {
                collector.collect(new Tuple2<>(word, 1));
            }
        }
    }

}

```

在resource文件夹下创建hellt.txt

```
hello word
hello flink
hello spark
hello scala
hi susu
hi kyle
susu and kyle
```

代码输出内容：

```
(spark,1)
(and,1)
(kyle,2)
(flink,1)
(susu,2)
(hi,2)
(scala,1)
(word,1)
(hello,4)
```

## 2.2 流处理方式实现worldCount

java代码

```java
package com.day1.wc;

import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamWordcount {

    public static void main(String[] args) throws Exception {

        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(8);

        String inputpath = "/Users/kyle/Documents/kyle/project/flink/learn/src/main/resources/hello.txt";

        DataStreamSource<String> inputStream = env.readTextFile(inputpath);

        // 基于数据流进行转换计算
        SingleOutputStreamOperator<Tuple2<String, Integer>> resultSet = inputStream.flatMap(new WordCount.MyFlatMapper())
                .keyBy(0)
                .sum(1);

        resultSet.print();

        // 执行任务
         env.execute();
    }

}

```

输出：

```
3> (hello,1)
6> (word,1)
2> (susu,1)
3> (hi,1)
2> (susu,2)
2> (kyle,1)
8> (and,1)
1> (spark,1)
3> (hello,2)
2> (kyle,2)
3> (hi,2)
1> (scala,1)
3> (hello,3)
7> (flink,1)
3> (hello,4)
```



## 2.3 通过NC模拟实时数据测试流式处理（Socket流读取数据）

java代码

```java
package com.day1.wc;

import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamWordcountNC {

    public static void main(String[] args) throws Exception {

        ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String host = parameterTool.get("host");
        int port = parameterTool.getInt("port");

        // 创建流处理执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(8);
        DataStreamSource<String> inputStream = env.socketTextStream(host, port);

        // 基于数据流进行转换计算
        SingleOutputStreamOperator<Tuple2<String, Integer>> resultSet = inputStream.flatMap(new WordCount.MyFlatMapper())
                .keyBy(0)
                .sum(1);

        resultSet.print();

        // 执行任务
        env.execute("SocketStreamTest");


    }
}

```

输出：

![image-20211029082426839](https://tva1.sinaimg.cn/large/008i3skNgy1gvvvh7r78cj320e0reaem.jpg)

## 2.4 批处理和流处理对比

1. 创建执行环境不同

   批处理：ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

   流处理：StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

2. 输出结果不同

   批处理：直接输出总的结果，没有每行的计算过程

   流处理：一行行的统计结果都输出，有明显的计算过程，体现出流处理的来一条数据处理一条数据。输出结果前面的数字是线程编号。

3. 聚合计算API不同

   批处理：groupby()

   ​	对数据进行分组，前提是要处理的一批数据都要已经上报到达了，可以一次性读取到，

   流处理：keyby()

   ​	流处理的思想是数据来一条处理一条，不需要等数据到到达才处理，所以这里的意思是按照指定的key（默认hashcode）进行重分区的操作。进过keyby()后，数据分到哪里去是跟数据的key相关的。注意看输出结果，会发现相同的单次出现在相同的线程编号中。

4. 任务执行

   批处理：逻辑代码写完后，不需要特意调用API执行任务

   流处理：逻辑代码写完后， 需要调用env.execute();来把流人物调起来，等待数据到了再进行计算。

# 3.Flink 运行时架构

## 3.1 Flink运行时的组件

### 3.1.1 <font >Job Manager(作业管理器)</font>

1. 控制一个应用程序执行的主程序，每个应用程序都会被一个不同的JobManager所控制执行。
2. JobManager会先接收到要执行的应用程序，这个应用程序会包括：JobGraph(作业图)，logical dataflow graph(逻辑数据流图)，和打包了所有的类，库以及其他资源的JAR包
3. JobManager把JobGraph转换成一个物理层面的数据流图，也就是“执行图”（ExecutionGraph）。
4. JobManager会想ResourceManager（资源管理器）请求执行任务必要的资源，也就是TaskManager（任务管理器）的slot（插槽）。一旦获取到足够的资源，就会将“执行图”发到真正运行它们的TaskManager上。
5. 在TaskManager运行过程中，JobManager会负责所有需要中央协调的操作，比如checkpoints(检查点)的协调

### 3.1.2 <font >Task Manager(任务执行器)</font>

1. 每一个TaskManager是一个JVM进程。通常在Flink中会有多个TaskManager运行，每一个TaskManager都包含了一定数量的slot（插槽）。插槽的数量就是TaskManager能够并行执行任务的最大数量。

2. 启动TaskManager会向资源管理器注册她的插槽，也就是在这个时候指定了taskManager的slot数量，当我们提交任务后，jobManager会向资源管理器申请资源，资源管理器就会根据taskManager的slot情况向合适的taskManager发送分配资源的指令，TaskManager收到指令后就会将一个或多个slot提供给JobManager调用，JobManager就可以向slot分配具体的task来执行。

3. taskManager的内存和slot数

   **Standalone模式：**

   ​	taskManager默认就是1个，可以在flink-conf.yaml中设置slot的数量（taskmanager.numberOfTaskSlots）

   **yarn Session-Cluster 模式：**

   ​	需要线启动集群，然后再提交作业，接着会向yarn申请一块空间后，资源永远保持不变。如果资源满了，下一个作业就无法提交，只能等到其中一个作业执行完后，释放了资源，下个作业才会正常提交，所有作业共享Dispatcher和resourceManager；共享资源，适合规模小执行时间短的作业。类似standalone

   ​	注意：在yarn中初始化一个flink集群，申请了指定的资源后，这个flink集群就会常驻yarn集群中，占用固定资源，即时没有任务跑，资源也不会释放回给到yarn。

   **yarn Per-Job-Cluster模式：**

   ​	启动时一个container就是一个taskManager，可以在提交任务的时候通过命令行配置taskManager的内存和slot数

   ​	--yarntaskManagerMemory 8192 \
   ​	--yarnslots 2 \

4. 在执行的过程中，一个TaskManager可以跟其它运行在同一个程序的TaskManager交换数据，

   

### 3.1.3 <font >Resource Manager(资源管理器)</font>

1. 主要负责管理TaskManager的slot， slot是Flink中定义的处理资源单元
2. Flink为不同的环境和资源管理工具提供了不同的资源管理器，比如YARN,Mesos,K8s,以及standalone部署
3. 当JobManager申请slot资源时，ResourceManager会将有空闲slot的TaskManager分配给JobManager。如果ResourceManager没有足够的slot来满足JobManager的请求，它还可以向资源提供平台发起会话，已提供启动TaskManager进程的容器。

### 3.1.4 <font >Dispacher(分发器)</font>

1. 可以跨作业运行，为【应用提交】提供了rest接口
2. 当一个应用被提交执行时，Dispacher就会启动并将应用移交给一个JobManager
3. Dispatcher也会启动一个Web UI，用来方便地展示和监控作业的执行信息
4. Dispatcher在架构中不是必需的。处决于【应用提交】运行的方式





## 3.2 任务提交流程

1. Client向HDFS上传Flink的jar包和配置
2. 向ResourceManager(Yarn)提交任务。
3. ResourceManager分配一个Container启动ApplicationMaster，ApplicationMaster启动后加载Flink的jar包和配置构建环境，然后启动JobManager.
4. JobManager根据配置或者代码设置的最大并行度向ResourceManager申请slot资源。
5. ResourceManager根据NodeManager的空闲资源情况分配Container，并将Container资源信息传递回JobManager。
6. JobManager获取到Container信息后，通知NodeManager启动TaskManager。
7. NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager，TaskManager向JobManager发送心跳资源，表示taskManager启动完成
8. JobManager向TaskManager发送具体的执行任务。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6cy0nsmdj316d0u0n09.jpg" style="zoom:50%;" />



思考问题:

在Flink on Yarn的情况下， ApplicationMaster和JobManager是什么关系？同一个jvm进程还是两个jvm进程？









## 3.3 任务调度原理





## 3.4 Slot和Parallelism

> Slot是静态的概念，是指TaskManager具有的并发执行能力，可以通过参数taskmanager.numberOfTaskSlots进行配置。
>
> Parallelism是动态概念，即TaskManager运行程序时实际使用的并发能力，可以通过参数parallelism.default进行配置。

### 3.4.1 Slot

- 每个 worker（TaskManager）都是一个 *JVM 进程*，可以在单独的线程中执行一个或多个 subtask。为了控制一个 TaskManager 中接受多少个 task，就有了所谓的 **task slots**（至少一个）。

- 每个 *task slot* 代表 TaskManager 中资源的固定子集。例如，具有 3 个 slot 的 TaskManager，会将其托管内存 1/3 用于每个 slot。分配资源意味着 subtask 不会与其他作业的 subtask 竞争托管内存，而是具有一定数量的保留托管内存。注意此处没有 CPU 隔离；当前 slot 仅分离 task 的托管内存。

- 每个 TaskManager 有一个 slot，这意味着每个 task 组都在单独的 JVM 中运行（例如，可以在单独的容器中启动）。具有多个 slot 意味着更多 subtask 共享同一 JVM。同一 JVM 中的 task 共享 TCP 连接（通过多路复用）和心跳信息。它们还可以共享数据集和数据结构，从而减少了每个 task 的开销。<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gvytie7bilj31c40dmdi3.jpg" alt="flink_slot_01" style="zoom:50%;" />

- 默认情况下，Flink 允许 subtask 共享 slot，即便它们是不同的 task 的 subtask，只要是来自于同一作业即可。结果就是一个 slot 可以持有整个作业管道。允许 *slot 共享*有两个主要优点：

  1. Flink 集群所需的 task slot 和作业中使用的最大并行度恰好一样。无需计算程序总共包含多少个 task（具有不同并行度）。

  2. 容易获得更好的资源利用。如果没有 slot 共享，非密集 subtask（*source/map()*）将阻塞和密集型 subtask（*window*） 一样多的资源。<font color = 'red'>通过 slot 共享，我们示例中的基本并行度从 2 增加到 6，可以充分利用分配的资源，同时确保繁重的 subtask 在 TaskManager 之间公平分配。</font>

     <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gvyto3k12yj31fm0iudkj.jpg" alt="flink_slot_02" style="zoom:50%;" />

### 3.4.2 Parallelism（并行度）

- 一个特定算子的子任务（subtask）的个数，称之为并行度（parallelism）。

- 一般情况下，一个strea的并行度，可以认为就是其所有算子中最大的并行度。

- eg.

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6ee3ejumj31ij0u0aer.jpg" alt="flink_slot_03" style="zoom:40%;" />

- eg.

  1.实际生产中，大多数sink需要合并，如小文件处理，数据库压力控制等，则需要单独设置parallelism，如example4

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6elg2cuuj311m0u0afi.jpg" alt="flink_parallelism_01" style="zoom:60%;" />

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6ettqbzkj31960u0aho.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

  

  

### 3.4.3 小结

1. solt是静态的概念，是指taskManager具有的并发能力；parallelism是动态的概念，是指程序运行的时候实际的并发情况

2. slot只隔离内存，不隔离cpu，每个slot的内存就是taskManger的总内存/slot数

3. 每个slot数怎么设置合理呢？考虑几点

   a. slot的数量等于taskManger的core数，这样slot不会互相抢core

   b. slot的数量大于taskManger的core数，slot会抢core，在适合的情况下这样会不会更高效一些（参考spark的官方建议并发度是core的2-3倍）

   c. slot的数量小于taskManger的core数，这样会浪费core数

   d. slot的数量尽可能少，即等于taskManager数，这样某些taskManager挂掉的情况下影响的slot数少。

   e. 在yarn模式下，slot要小于container的最大cpu数(yarn.scheduler.maximun-allocation-vcores)，否则启动报错 ？

4. 在standalone模式下一个物理节点配置一个worker，在yarn的pre-job模式下一个container一个taskManager，然后再配置文件中配置这个worker的总内存和总CPU，也就是Flink在这个节点上可以使用的总cpu和内存资源，要剩余一部分来保证节点的正常执行。然后再worker的基础上配置slot的数量，相当于是将worker的总资源平均分到各个slot上，官方建议`taskmanager.numberOfTaskSlots`配置的Slot数量和CPU相等或成比例?（未找到出处，需要求证）。

5. Flink on yarn时，使用pre-job模式，yarn负责管理和提供资源（作为resource manager），yarn的一个container里面启动一个taskManager，所以这时候container，taskManager，solt，parallelism的关系是：

   > container.num = taskManager.num = max(parallelism)/taskmanager.numberOfTaskSlots

6. yarn是有固定参数来确定一个container中cpu和内存的最大值和最小值，Flink在执行任务的时候可以指定taskMananger的内存和slot，也就是可设置一个container中执行任务的并行度已经每个slot的内存（在container最大最小值之间），

```shell
/opt/flink-1.14.0/bin/flink run \
--detached \
--jobmanager yarn-cluster \
--yarnname "workCoundNC" \
--yarnjobManagerMemory 4096 \
--yarntaskManagerMemory 8192 \
--yarnslots 2 \
--parallelism 20 \
--class com.day1.wc.StreamWordcountNC \
learn-1.0-SNAPSHOT.jar
# 以上任务需要container数=parallelism/yarnslots(如果程序内没有设置parallelism或者设置小于20)
```



## 3.5 程序结构和数据流图

### 3.5.1 程序与数据流（DataFlow）

- Flink程序三部分：
  1. Source：负责读取数据源
  2. Transformation：利用各种算子进行处理加工
  3. Sink：负责输出结果
- 每个dataflow以一个或多个sources开始，以一个或多个sinks结束。
- 在大部分情况下，程序中的转换运算（transformation）跟dataflow中的算子（operator）是一一对应关系。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6fhsmgkbj318i0t8gob.jpg" alt="flink_dataFlow_01" style="zoom:40%;" />

### 3.5.2 执行图（ExecutionGraph）

- Flink中的执行图分为四层：

  1. StreamGraph：根据用户通过Stream API编写的代码生成的最初的图，用来表示程序的拓扑结构

  2. **JobGraph**：StreamGraph经过优化后生成了JobGraph，提交给JobManager的数据结构。主要优化：将多个符合条件的节点chain在一起作为一个节点。

  3. **ExecutionGraph**：JobManager根据JobGraph生产ExecutionGraph，是JobGraph的并行化版本，是调度层最核心的数据结构。

  4. 物理执行图：JobManager根据ExecutionGraph对Job进行调度后，在各个TaskManager上部署Task形成的“图”，并不是一个具体的数据结构。

     <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6fwpmv4dj30ut0u0djp.jpg" alt="flink_parallelism_02" style="zoom:70%;" />

## 3.6 数据传输和任务链

### 3.6.1 数据传输形式

- 一个程序中，不同的算子可能具有不同的 并行度
- 算子之间传输数据的形式：
  1. one-to-one：stream维护着分区以及元素的顺序（比如source和map之间），这意味着map算子的子任务看到的元素个数以及顺序跟source算子的子任务生产的元素的个数，顺序相同。map，filter，flatmap等算子都是one-to-one的对应关系
  2. redistributing：stream的分区会发生改变。每一个算子的任务依据所选择的transformation发送数据到不同的目标任务。例如keyby基于hashCode重分区，而broadcast和rebalance会随机重新分区（轮询），这些算子都会引起redistribute过程，redistribute过程类似spark的shuffle过程。

### 3.6.2 任务链（Operator Chains）

- Flink采用了一种称为**任务链**的优化技术，可以在特定条件下减少本地通信的开销。

- 满足任务链的要求：

  1. slot共享组相同

  2. 两个算子并行度相同

  3. 两个算子是one-to-one操作（前后数据传输关系是forward）

  4. 取消chain操作：

     ~~~java
     env.disableOperatorChaining();
     xxx.startNewChain()
     ~~~

     

  5. eg.

  <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gw6jzi4m5wj31100u0tck.jpg" alt="flink_parallelism_01" style="zoom:50%;" />





# 4.Flink流处理API

## 4.1 创建执行环境

> 创建一个执行环境，表示当前执行程序的上下文。如果程序是独立调用的，则此方法返回本地执行环境，如果从命令行客户端调用程序以提交答集群，则此方法返回目标集群的执行环境，也就是说， getExecutionEnvironment 会根据运行的方式决定返回什么样的运行环境，是最常用的一种创建执行环境的方式。

~~~java
// 创建流处理执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
~~~

### 4.1.1 createLocalEnvironment

​	返回本地执行环境，调用的时候需要指定默认的并行度，不指定则以当前cpu核数为准

~~~java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);
~~~

### 4.1.2 createRemoteEnvironment

​	返回集群执行环境，将Jar提交到远程服务器。调用的时候需要指定JobManager的IP和端口号，并指定要在集群中运行的Jar包

~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment("", 6123, "");
~~~



## 4.2 Source(读取数据源)

### 4.2.1 从内存读取数据

1. Collection

   ~~~java
   StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   List<SensorReading> data = Arrays.asList(
           new SensorReading("sensor_01", 1547718199L, 35.8),
           new SensorReading("sensor_02", 1547718201L, 15.4),
           new SensorReading("sensor_03", 1547718202L, 6.7),
           new SensorReading("sensor_04", 1547718205L, 38.1)
   );
   DataStream<SensorReading> dataStreamSource = env.fromCollection(data);
   dataStreamSource.print("data");
   ~~~

2. Element

   ~~~java
   DataStream<Integer> integerDataStream = env.fromElements(1, 2, 3, 4, 5);
   integerDataStream.print("integerDataStream");
   ~~~

### 4.2.2 从文件读取数据

1. textFile

   ~~~java
   StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   dataStreamSource.print();
   env.execute();
   ~~~

2. file

   ~~~java
   // 需要自行设置读取格式
   env.readFile()
   ~~~

### 4.2.3 从ksocket读取数据

1. socket

~~~java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStreamSource<String> inputStream = env.socketTextStream(host, port);
// 基于数据流进行转换计算
SingleOutputStreamOperator<Tuple2<String, Integer>> resultSet = inputStream.flatMap(new WordCount.MyFlatMapper())
        .keyBy(0)
        .sum(1);

resultSet.print();
// 执行任务
env.execute("SocketStreamTest");
~~~

### 4.2.4 从kafka读取数据

> Flink读取kafka的时候要注意Flink和kafka的版本兼容问题

1. 增加kafka依赖

~~~xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
    <version>${kafka.version}</version>
</dependency>
~~~

2. 消费kafka代码

~~~java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "192.168.2.113:9092");
properties.setProperty("group.id", "flink-group");
String inputTopic = "my_log";

FlinkKafkaConsumer<String> stringFlinkKafkaConsumer = new FlinkKafkaConsumer<>(inputTopic, new SimpleStringSchema(), properties);

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStreamSource<String> kafkaSource = env.addSource(stringFlinkKafkaConsumer);

kafkaSource.print();

env.execute();
~~~

### 4.2.5 自定义数据源

1. bean

   ~~~java
   package com.kyle.bean;
   
   /**
    * @author kyle on 2021-11-07 2:40 下午
    */
   
   
   // 传感器温度读数的数据类型
   public class SensorReading {
       // 属性： id， 时间戳， 温度值
   
       private String id;
       private Long timestamp;
       private Double temperature;
   
       public SensorReading() {
       }
   
       public SensorReading(String id, Long timestamp, Double temperature) {
           this.id = id;
           this.timestamp = timestamp;
           this.temperature = temperature;
       }
   
       public String getId() {
           return id;
       }
   
       public void setId(String id) {
           this.id = id;
       }
   
       public Long getTimestamp() {
           return timestamp;
       }
   
       public void setTimestamp(Long timestamp) {
           this.timestamp = timestamp;
       }
   
       public Double getTemperature() {
           return temperature;
       }
   
       public void setTemperature(Double temperature) {
           this.temperature = temperature;
       }
   
       @Override
       public String toString() {
           return "SensorReading{" +
                   "id='" + id + '\'' +
                   ", timestamp=" + timestamp +
                   ", temperature=" + temperature +
                   '}';
       }
   }
   
   ~~~

2. 自定义source

   ~~~java
   package com.kyle.api.source;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.source.SourceFunction;
   
   import java.util.HashMap;
   import java.util.Random;
   
   /**
    * @author kyle on 2021-11-17 8:23 上午
    */
   public class SourceTest4_UDF {
   
   
      public static void main(String[] args) throws Exception {
   
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         env.setParallelism(1);
   
         DataStreamSource<SensorReading> streamSource = env.addSource(new MySensorSource());
   
         streamSource.print();
   
         env.execute();
   
   
      }
   
      // 实现自定义的 SourceFunction
      public static class MySensorSource implements SourceFunction<SensorReading>{
   
         // 定义一个标识位， 用来控制数据的产生
         private boolean running = true;
   
         @Override
         public void run(SourceContext<SensorReading> ctx) throws Exception {
            //定义一个随机数发生器
            Random random = new Random();
   
            // 设置10个传感器的初始温度
            HashMap<String, Double> sensorTempMap = new HashMap<>();
            for (int i = 0; i < 10; i++){
               sensorTempMap.put("sensor_" + (i + 1), 60 + random.nextGaussian() * 20);
            }
            while (running){
               for (String sensorId : sensorTempMap.keySet()) {
                  // 在当前温度基础上随机波动
                  double newTemp = sensorTempMap.get(sensorId) + random.nextGaussian();
                  sensorTempMap.put(sensorId, newTemp);
                  ctx.collect(new SensorReading(sensorId, System.currentTimeMillis(), newTemp));
               }
               // 控制输出频率
               Thread.sleep(1000L);
            }
         }
   
         @Override
         public void cancel() {
            running = false;
         }
      }
   }
   
   ~~~



## 4.3 Transform(转换算子， 逻辑处理阶段)

### 4.3.1 基本操作

1. map、flatMap、filter

   ~~~java
   public class TransformTest1_Base {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         // 1. map. 把String转换成长度输出
         SingleOutputStreamOperator<Integer> mapStream = dataStreamSource.map(new MapFunction<String, Integer>() {
            @Override
            public Integer map(String s) throws Exception {
               return s.length();
            }
         });
   
         // 2. flatMap
         SingleOutputStreamOperator<String> flatMapStream = dataStreamSource.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public void flatMap(String s, Collector<String> collector) throws Exception {
               String[] fields = s.split(",");
               for (String field : fields) {
                  collector.collect(field);
               }
            }
         });
   
         // 3. filter 不能改变流的类型
         SingleOutputStreamOperator<String> filterStream = dataStreamSource.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String s) throws Exception {
               return s.startsWith("sensor_01");
            }
         });
   
         mapStream.print("map");
         flatMapStream.print("flatMap");
         filterStream.print("filter");
   
         env.execute();
      }
   }
   ~~~



### 4.3.2 聚合操作

1. 分组数据流：

   keyBy：DataStream ---> KeyedStream, 根据指定的key（字段），通过hash获取到hashCode后，取模运算分配到对应的分区，即逻辑地将一个流拆分成多个不相交的分区。最终得到的结果是：**包含相同key的所有数据一定分配到相同的一个分区中，同一个分区中可以有多个key的数据。**

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwhvrakmo8j315i0fg0tm.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

2. 滚动聚合算子

   > 注意 max maxBy的区别

   max maxby

   

3. 一般化聚合 Reduce（归约）

   > 在reduceFunction中 第一个参数t1是旧的值， 第二个参数t2是最新的值

   ~~~java
   package com.kyle.api.Transform;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.functions.ReduceFunction;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.KeyedStream;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   
   /**
    * @author kyle on 2021-11-18 8:20 上午
    */
   public class TransformTest3_Reduce {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStream = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         //分组
         KeyedStream<SensorReading, Tuple> keyedStream = mapStream.keyBy("id");
   
         // reduce 组合 ， 取最大的温度值，以及当前最新的时间戳
         // 在reduceFunction中 第一个参数t1是旧的值， 第二个参数t2是最新的值
         SingleOutputStreamOperator<SensorReading> reduceStream = keyedStream.reduce(new ReduceFunction<SensorReading>() {
            @Override
            public SensorReading reduce(SensorReading t1, SensorReading t2) throws Exception {
               return new SensorReading(t1.getId(), t2.getTimestamp(), Math.max(t1.getTemperature(), t2.getTemperature()));
            }
         });
   
         reduceStream.print();
         env.execute();
      }
   }
   ~~~

   

4. 分流：Split和Select <font color='red'>注意 split函数在1.13.1被删除了， 代替的是侧流输出</font>

   4.1、 Split

   ​	**DataStram ---> SplitStream:** 根据某些特征把一个DataStream拆分成两个或者多个DataStream

   ​	<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwj02ikodxj30yu0het9p.jpg" alt="flink_split_01" style="zoom:50%;" />

   

   4.2、Select

   ​	**SplitStream ---> DataStream:** 从一个SplitStream中获取一个或者多个DataStream.

   ​	<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwj043jv01j30uw0e4dgx.jpg" alt="flink_select_01" style="zoom:50%;" />

   

5. 分流的最新方式: 旁路输出

   ~~~java
   package com.kyle.api.Transform;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.functions.ReduceFunction;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.KeyedStream;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.ProcessFunction;
   import org.apache.flink.util.Collector;
   import org.apache.flink.util.OutputTag;
   
   /**
    * @author kyle on 2021-11-18 8:40 上午
    */
   public class TransformTest4_OutputTag {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStream = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         // 转换成 SensorReading
         SingleOutputStreamOperator<SensorReading> mapStream = dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         // 分流
         // 注意 定义outputTag的时候最后的两个{}不能省略， 否则报错
         OutputTag<SensorReading> highStream = new OutputTag<SensorReading>("high"){};
         OutputTag<SensorReading> lowStream = new OutputTag<SensorReading>("low"){};
         SingleOutputStreamOperator<SensorReading> processStream = mapStream.process(new ProcessFunction<SensorReading, SensorReading>() {
            @Override
            public void processElement(SensorReading value, Context ctx, Collector<SensorReading> out) throws Exception {
               if (value.getTemperature() > 30) {
                  ctx.output(highStream, value);
               } else if (value.getTemperature() < 30){
                  ctx.output(lowStream, value);
               }else{
                  out.collect(value);
               }
            }
         });
   
         // 获取对应的流
         DataStream<SensorReading> highSideOut = processStream.getSideOutput(highStream);
         DataStream<SensorReading> lowSideOut = processStream.getSideOutput(lowStream);
   
         highSideOut.print("high");
         lowSideOut.print("low");
   
         env.execute();
   
      }
   
   }
   
   ~~~

6. 合流

   6.1 connect：将两条流合并到一条流，两条流的数据结构可以不同，但不能合并3条及以上的流（代码片段基于上面的分流）

   ~~~java
   
   // 获取对应的流
   DataStream<SensorReading> highSideOut = processStream.getSideOutput(highStream);
   DataStream<SensorReading> lowSideOut = processStream.getSideOutput(lowStream);
   
   
   highSideOut.print("high");
   lowSideOut.print("low");
   
   // 合流 connect， 将高温流转换成二元组类型，再和低温流链接合并后， 输出状态信息
   SingleOutputStreamOperator<Tuple2<String, Double>> warnStream = highSideOut.map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
    @Override
    public Tuple2 map(SensorReading value) throws Exception {
       return new Tuple2<>(value.getId(), value.getTemperature());
    }
   });
   
   ConnectedStreams<Tuple2<String, Double>, SensorReading> connectStream = warnStream.connect(lowSideOut);
   
   
   SingleOutputStreamOperator<Object> resStream = connectStream.map(new CoMapFunction<Tuple2<String, Double>, SensorReading, Object>() {
    @Override
    public Object map1(Tuple2<String, Double> value) throws Exception {
       return new Tuple3<>(value.f0, value.f1, "high temp warning");
    }
   
    @Override
    public Object map2(SensorReading value) throws Exception {
       return new Tuple2<>(value.getId(), "normal");
    }
   });
   
   resStream.print("res");
   
   
   env.execute();
   ~~~

   6.2 union：对两个或以上的dataStream合并生成一个包含所有dataStream元素的新dataStream，要求所有的dataStream的数据结构一致，可以合并多条流。

   ~~~java
   // 获取对应的流
   DataStream<SensorReading> highSideOut = processStream.getSideOutput(highStream);
   DataStream<SensorReading> lowSideOut = processStream.getSideOutput(lowStream);
   
   
   highSideOut.print("high");
   lowSideOut.print("low");
   
   // 合流 connect， 
   SingleOutputStreamOperator<Tuple2<String, Double>> warnStream = highSideOut.map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
    @Override
    public Tuple2 map(SensorReading value) throws Exception {
       return new Tuple2<>(value.getId(), value.getTemperature());
    }
   });
   
   SingleOutputStreamOperator<Tuple2<String, Double>> normalStream = lowSideOut.map(new MapFunction<SensorReading, Tuple2<String, Double>>() {
    @Override
    public Tuple2<String, Double> map(SensorReading value) throws Exception {
       return new Tuple2<>(value.getId(), value.getTemperature());
    }
   });
   // union
   DataStream<Tuple2<String, Double>> unionStream = warnStream.union(normalStream);
   
   SingleOutputStreamOperator<Tuple3<String, Double, String>> resUnionStream = unionStream
         .map(new MapFunction<Tuple2<String, Double>, Tuple3<String, Double, String>>() {
    @Override
    public Tuple3<String, Double, String> map(Tuple2<String, Double> value) throws Exception {
       return new Tuple3<>(value.f0, value.f1, "mixed");
    }
   });
   
   resUnionStream.print("resUnion");
   
   env.execute();
   ~~~

   

## 4.4 Flink支持的数据类型

### 4.4.1 基础数据类型

​	Flink 支持所有的 Java 和 Scala 基础数据类型，Int, Double, Long, String, …

### 4.4.2 **Java** **和** **Scala** 元组（Tuples）

### 4.4.3  **Scala** 样例类（case classes）

### 4.4.4 其他（Arrays， Lists， Maps，Enums 等）



## 4.5 UDF 函数 --- 更细粒度的控制流

> Flink 暴露了所有udf函数的接口（实现方式为接口或者抽象类）。例如MapFunciton，FilterFuncion， ProcessFunction...

### 4.5.1 普通UDF

~~~java
package com.kyle.api.udf;

import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author kyle on 2021-11-20 7:30 上午
 */
public class FlinkUDFTest_01 {

   public static void main(String[] args) throws Exception {

      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
      
      SingleOutputStreamOperator<String> sensor_01Stream = dataStreamSource.filter(new KeyWordFilter("sensor_01"));
      sensor_01Stream.print();

      env.execute();

   }
   
   public static class KeyWordFilter implements FilterFunction<String>{
      private String keyWord;

      public KeyWordFilter(String keyWord) {
         this.keyWord = keyWord;
      }

      @Override
      public boolean filter(String value) throws Exception {
         return value.contains(this.keyWord);
      }
   }

}

~~~

### 4.5.2 RicUDF

>RichUDF的功能很强大，可以获取上下文很多信息
>
>疑问：RichFunction中的 open的什么周期是？？？

1. 在RichFunction中可以做初始化的工作， 比如初始化一些映射，建立外部数据库链接，这样可以在一个substask中只建立一个

2. 可以获取广播变量，减少广播变量的次数

3. 这里有点类似spark的foreachPartition

   ~~~java
   package com.kyle.api.udf;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.functions.RichMapFunction;
   import org.apache.flink.api.java.tuple.Tuple2;
   import org.apache.flink.api.java.tuple.Tuple3;
   import org.apache.flink.api.java.tuple.Tuple4;
   import org.apache.flink.configuration.Configuration;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   
   import java.util.ArrayList;
   import java.util.HashMap;
   import java.util.List;
   
   /**
    * @author kyle on 2021-11-20 7:49 上午
    */
   public class FlinkUDFTest_02Rich {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(3);
   
         SingleOutputStreamOperator<SensorReading> projoStream = dataStreamSource.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         ArrayList<HashMap<String, String>> sensorCodes = new ArrayList<>();
         HashMap<String, String> sensorCode = new HashMap<>();
         sensorCode = sensorCode;
         sensorCode.put("sensor_01", "1楼");
         sensorCode.put("sensor_02", "2楼");
         sensorCode.put("sensor_03", "3楼");
         sensorCode.put("sensor_04", "4楼");
         sensorCodes.add(sensorCode);
         DataStreamSource<HashMap<String, String>> sensorStream = env.fromElements(sensorCode);
   
         SingleOutputStreamOperator<Tuple4<String, String, Double, Integer>> richMapStream = projoStream.map(new MyRichMap());
   
         richMapStream.print();
   
         env.execute();
   
      }
   
      public static class MyRichMap extends RichMapFunction<SensorReading, Tuple4<String, String, Double,Integer>> {
         HashMap<String,String> sensorCode;
         List<HashMap<String, String>> bcSensorCodes;
   
         @Override
         public Tuple4<String, String, Double, Integer> map(SensorReading value) throws Exception {
   
            // 通过每个taskManager本地初始化的map获取数据
            return new Tuple4<>(sensorCode.get(value.getId()), value.getId(), value.getTemperature(),getRuntimeContext().getIndexOfThisSubtask());
   
            // 通过广播变量中获取映射数据 注意 广播变量只能用在 DataSet programs
   //         HashMap<String, String> bcSensorCode = bcSensorCodes.get(0);
   //         return new Tuple4<>(bcSensorCode.get(value.getId()), value.getId(), value.getTemperature(), getRuntimeContext().getIndexOfThisSubtask());
   
         }
   
         // 疑问： 这个open的生命周期是？？？
         @Override
         public void open(Configuration parameters) throws Exception {
            // 初始化工作， 一般是用来定义状态，或者建立外部数据库链接，减少数据库链接的次数,
            // 有点类似spark的 foreachPartition
            System.out.println("open");
            sensorCode = new HashMap<>();
            sensorCode.put("sensor_01", "1楼");
            sensorCode.put("sensor_02", "2楼");
            sensorCode.put("sensor_03", "3楼");
            sensorCode.put("sensor_04", "4楼");
   
            // 广播变量，减少广播变量传递次数
            // Caused by: java.lang.UnsupportedOperationException: Broadcast variables can only be used in DataSet programs
   //         bcSensorCodes = getRuntimeContext().getBroadcastVariable("sensorCodes");
   
         }
   
         @Override
         public void close() throws Exception {
            // 一般做关闭链接和清空状态的事情
            System.out.println("close");
         }
      }
   
   }
   
   ~~~

## 4.6 重分区操作，数据在传输中的定义方式

### 4.6.1  shuffle, rebalance, rescale, global

~~~java
package com.kyle.api.partition;

import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

/**
 * @author kyle on 2021-11-20 9:09 上午
 */
public class PartitionTest_01 {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);
        DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");

        dataStreamSource.print("input");

        // shuffle， 将数据随机打散，随机分布到下游分区中
        DataStream<String> shuffleStream = dataStreamSource.shuffle();
        shuffleStream.print("shuffle");

        // rebalance, 以轮询的方式将输出的元素均匀的分配到下游的分区中
        // 前后两个算子的分区不同的情况下，默认使用的重分区方式就是 rebalance
        dataStreamSource.rebalance().print("rebalance");

        // rescale, 以轮询的方式将输出的元素按照原来分区数分组的方式均匀的分配到下游的分区
        // 即，上游是2个分区而下游是4个分区的时候，上游的1个分区以轮询的方式将元素分配到下游其中的两个分区，上游另外1个分区也以轮询的方式将元素分配到下游另外两个分区中
        // 当 上游有4个分区而下游有2个分区的情况，上游的2个分区将元素分配到下游其中一个分区，上游另外2个分区将元素分配到下游另外一个分区中。
        // 可以看成是一个分组的重分区，重新平衡的方式
        dataStreamSource.rescale().print("rescale");

        // global , 将所有数据汇聚到第一个分区中
        dataStreamSource.global().print("global");

        env.execute();

    }

}

~~~



## 4.7 Sink

### 4.7.1 kafka sink

1. 从kafka消费数据处理后写会kafka，形成一条数据管道

   > 遗留问题：脏数据导致的异常处理？？？ 保证数据流正常跑不会因为脏数据的崩溃

   ~~~java
   package com.kyle.api.sink;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.serialization.SimpleStringSchema;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
   import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
   
   import java.util.Properties;
   
   /**
    * @author kyle on 2021-11-20 9:39 上午
    */
   public class SinkTesk01_kafka {
   
      public static void main(String[] args) throws Exception {
   
         Properties properties = new Properties();
         properties.setProperty("bootstrap.servers", "192.168.2.113:9092");
         properties.setProperty("group.id", "flink-group");
         String inputTopic = "my_log";
   
         FlinkKafkaConsumer<String> stringFlinkKafkaConsumer = new FlinkKafkaConsumer<>(inputTopic, new SimpleStringSchema(), properties);
         stringFlinkKafkaConsumer.setStartFromLatest();
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> kafkaSource = env.addSource(stringFlinkKafkaConsumer);
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<String> mapStream = kafkaSource.map(new MapFunction<String, String>() {
            @Override
            public String map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2])).toString();
            }
         });
   
         Properties sinkProperties = new Properties();
         sinkProperties.setProperty("bootstrap.servers", "192.168.2.113:9092");
         FlinkKafkaProducer<String> stringFlinkKafkaProducer = new FlinkKafkaProducer<String>("from_flink", new SimpleStringSchema(), sinkProperties);
   
         mapStream.addSink(stringFlinkKafkaProducer);
         env.execute();
   
      }
   
   }
   
   ~~~

### 4.7.2 redis

1. 将数据写入redis

   ~~~java
   package com.kyle.api.sink;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.redis.RedisSink;
   import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommand;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommandDescription;
   import org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper;
   
   /**
    * @author kyle on 2021-11-20 10:45 上午
    */
   public class SinkTest02_redis {
   
      public static void main(String[] args) throws Exception {
   
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStreamSource.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         // 定义redis jedis 链接配置
         FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                 .setHost("127.0.0.1")
                 .setPort(6380)
                 .build();
   
         RedisSink<SensorReading> sensorReadingRedisSink = new RedisSink<>(config, new MyRedisMapper());
         mapStream.addSink(sensorReadingRedisSink);
   
         env.execute();
   
      }
   
      // 自定义redisMapper
      public static class MyRedisMapper implements RedisMapper<SensorReading>{
   
         // 定义保存数据到redis的命令, 存成hash表，hset sensor_temp id temperature
         @Override
         public RedisCommandDescription getCommandDescription() {
            return new RedisCommandDescription(RedisCommand.HSET, "sensor_tmp");
         }
   
         @Override
         public String getKeyFromData(SensorReading sensorReading) {
            return sensorReading.getId();
         }
   
         @Override
         public String getValueFromData(SensorReading sensorReading) {
            return sensorReading.getTemperature().toString();
         }
      }
   
   }
   
   ~~~

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwlg8n3pbvj30uc0n4td6.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

### 4.7.3 Elasticsearch

1. 将数据写入ES

   > 以下代码只是简单的api学习调用，跟写入ES相关的性能调优还要后续深入了解。

   ~~~java
   package com.kyle.api.sink;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.functions.RuntimeContext;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.connectors.elasticsearch.ElasticsearchSinkFunction;
   import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
   import org.apache.flink.streaming.connectors.elasticsearch7.ElasticsearchSink;
   import org.apache.http.HttpHost;
   import org.elasticsearch.action.index.IndexRequest;
   import org.elasticsearch.client.Requests;
   
   import java.util.ArrayList;
   import java.util.HashMap;
   
   /**
    * @author kyle on 2021-11-20 11:26 上午
    */
   public class SinkTesk03_Es {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStreamSource.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         ArrayList<HttpHost> httpHost = new ArrayList<>();
         httpHost.add(new HttpHost("192.168.2.113", 9200));
         mapStream.addSink(new ElasticsearchSink.Builder<SensorReading>(httpHost, new MyEsSinkFunction()).build());
   
         env.execute();
   
      }
   
      // 实现自定义的ES写入操作
      public static class MyEsSinkFunction implements ElasticsearchSinkFunction<SensorReading>{
   
         @Override
         public void process(SensorReading sensorReading, RuntimeContext runtimeContext, RequestIndexer requestIndexer) {
            // 定义写入的数据source
            HashMap<String, String> dataSource = new HashMap<>();
            dataSource.put("id", sensorReading.getId());
            dataSource.put("temp", sensorReading.getTemperature().toString());
            dataSource.put("ts", sensorReading.getTimestamp().toString());
   
            //创建请求， 作为向es发起的写入命令
            IndexRequest indexRequest = Requests.indexRequest()
                    .index("sensor2")
   //                 .id(sensorReading.getId())
                    .source(dataSource);
   
            // 用index发送请求
            requestIndexer.add(indexRequest);
   
         }
      }
   }
   
   ~~~

### 4.7.4 自定义Sink --- jdbc

> 在实现sink接口的时候就要考虑链接的问题，切记要避免每条数据链接一次，spark里面使用foreachPartition处理，flink这里使用RichSinkFunction处理

1. 继承RichSinkFunction实现自定义Sink的逻辑

   ~~~java
   package com.kyle.api.sink;
   
   import com.kyle.api.source.SourceTest4_UDF;
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.configuration.Configuration;
   import org.apache.flink.streaming.api.datastream.DataStreamSink;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
   
   import java.sql.Connection;
   import java.sql.DriverManager;
   import java.sql.PreparedStatement;
   
   /**
    * @author kyle on 2021-11-21 9:17 上午
    */
   public class SinkTest04_UDF_Jdbc {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   //      DataStreamSource<String> dataStreamSource = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   //
   //      env.setParallelism(1);
   //      SingleOutputStreamOperator<SensorReading> mapStream = dataStreamSource.map(new MapFunction<String, SensorReading>() {
   //         @Override
   //         public SensorReading map(String s) throws Exception {
   //            String[] fields = s.split(" ");
   //            return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
   //         }
   //      });
         DataStreamSource<SensorReading> dataStream = env.addSource(new SourceTest4_UDF.MySensorSource());
         dataStream.addSink(new MyJdbcSink());
         env.execute();
   
      }
   
      // 自定义的jdbcSink
      // 使用richFunction， 避免每来一条数据创建一个数据库链接。
      public static class MyJdbcSink extends RichSinkFunction<SensorReading>{
         Connection connection = null;
         PreparedStatement insertStmt = null;
         PreparedStatement updateStmt = null;
   
         @Override
         public void open(Configuration parameters) throws Exception {
            Class.forName("com.mysql.jdbc.Driver");
            connection = DriverManager.getConnection("jdbc:mysql://192.168.2.113:3306/test", "root", "root");
            insertStmt = this.connection.prepareStatement("insert into sensor_tmp(id, temp) values(?, ?)");
            updateStmt = this.connection.prepareStatement("update sensor_tmp set temp = ? where id = ?");
         }
   
         // 没来一条数据， 调用链接， 执行sql
         @Override
         public void invoke(SensorReading value, Context context) throws Exception {
            //直接执行更新语句，如果没有更新，那么插入
            updateStmt.setDouble(1, value.getTemperature());
            updateStmt.setString(2, value.getId());
            updateStmt.execute();
            if (updateStmt.getUpdateCount() == 0 ){
               insertStmt.setString(1, value.getId());
               insertStmt.setDouble(2, value.getTemperature());
               insertStmt.execute();
            }
         }
   
         @Override
         public void close() throws Exception {
            insertStmt.close();
            updateStmt.close();
            connection.close();
         }
      }
   }
   
   ~~~

## 4.8 关于sink的总结

### 4.8.1 sink的类型

1. 目前主流的流组件如kafka和数据存储ES、mysql、redis等都有官方提供的sink，如果没有可以通过自定义来实现自己特殊的sink。

参考链接https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/connectors/datastream/overview/

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwmjzewyqyj30wd0u0437.jpg" alt="flink_parallelism_01" style="zoom:80%;" />

### 4.8.2 实现sink的时候性能问题

1. 注意数据库的压力问题
2. 注意避免每条数据都进行一次链接操作
3. 可以考虑分批次写入的可能



# 5.Window API 窗口函数

## 5.1 窗口

### 5.1.1 Flink 窗口的简述

1. Flink底层是引擎是一个流式引擎，Flink认为Batch是Streaming的一个特例，窗口（window）这是从Streaming到Batch的桥梁
2. 在无界流上截取一段，也就是有界流，这就是叫做开了一个窗口。
3. 窗口即是把无界流切分成有界流的方式，它会将流数据分发到有限大小的桶（bucket）中进行分析。
4. 窗口可以基于时间驱动（Time Window，例如：每30秒钟），也可以基于数据驱动（Count Window，例如：每100条数据）

### 5.1.2 Window 类型

1. 时间窗口（Time Window）
   * 滚动时间窗口
   * 滑动时间窗口
   * 会话窗口
2. 计数窗口（Count Window）
   * 滚动计数窗口
   * 滑动计数窗口

### 5.1.3 滚动窗口（Tumbling Windows）

1. 将数据依据固定的窗口长度对数据进行切分，如图

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwmkt86k7vj30q00g8gmi.jpg" alt="flink_parallelism_01" style="zoom:70%;" />

2. 窗口时间/数量对齐，长度固定，没有重叠

3. 窗口默认左闭右开，当前区间数据包含起始时间数据，不包含结束点时间。即 9点到10点的窗口包含9点整的数据不包含10点整的数据。

4. 场景：基于时间 --- 我们需要统计每一分钟中用户购买的商品的总数，需要将用户的行为事件按每一分钟进行切分，这种切分被成为翻滚时间窗口（Tumbling Time Window）

5. 场景：基于事件 --- 当我们想要每100个用户的购买行为作为驱动，那么每当窗口中填满100个”相同”元素了，就会对窗口进行计算。

### 5.1.4 滑动窗口（Sliding Windows）

1. 滑动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口长度和滑动间隔组成， 如图

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwml29o63oj30p20eodh1.jpg" alt="flink_parallelism_01" style="zoom:70%;" />

2. 窗口长度固定，可以有重叠

3. 场景：基于时间 --- 我们可以每30秒计算一次最近一分钟用户购买的商品总数。

4. 场景：基于事件 --- 每10个 “相同”元素计算一次最近100个元素的总和.



### 5.1.5 会话窗口

1. 由一系列事件组合一个指定时间长度的timeout间隙组成，也就是一段时间没有接收到新数据就会关闭当前窗口生成新的窗口。

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwml7ayqzcj30ne0e0q3q.jpg" alt="flink_parallelism_01" style="zoom:70%;" />

2. 时间不对齐，但只有时间驱动，没有计数驱动

3. 场景：基于时间 --- 计算每个用户在活跃期间总共购买的商品数量，如果用户5分钟没有活动则视为会话断开。



## 5.2 窗口分配器 window assigner (开窗)

> 开窗操作
>
> - window()方法接收的输入参数是一个WindowAssigner
> - WindowAssigner负责将每条输入的数据分发到正确的window中
> - Flink提供了通用的WindowAssigner
>   - 滚动窗口 (tumbling window)
>   - 滑动窗口 (sliding window)
>   - 会话窗口 (session window)
>   - 全局窗口 (global window)
>
> 

### 5.2.1 window() --- 底层开窗api

1. 计数窗口使用底层开窗api的方式会比较复杂， 设计GlobalWindows。建议直接使用 countWindos

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.windowing.assigners.*;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.streaming.api.windowing.triggers.CountTrigger;
   import org.apache.flink.streaming.api.windowing.triggers.PurgingTrigger;
   import org.apache.hadoop.fs.shell.Count;
   
   /**
    * @author kyle on 2021-11-23 8:05 上午
    */
   public class WindowTest01_TimeWindow {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStream = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         mapStream.keyBy("id")
   //              .window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(100)));  // 计数滚动窗口， 每100个元素开新窗口。
   //              .window(SlidingProcessingTimeWindows.of(Time.seconds(15), Time.seconds(5))); // 时间滑动窗口 15秒窗口长度， 5秒滑动长度
   //              .window(EventTimeSessionWindows.withGap(Time.minutes(1)));   // 会话窗口，1分钟没有新数据则开启新窗口
                 .window(TumblingProcessingTimeWindows.of(Time.seconds(15))); // 时间滚动窗口
   
         env.execute();
   
      }
   
   }
   
   ~~~

2. 各自封装的开窗独立api

   > - 计数窗口使用底层开窗api的方式会比较复杂， 设计GlobalWindows。建议直接使用 countWindos
   > - 会话窗口没有地理封装的api，使用window
   > - 时间窗口独立封装的timeWindow在1.14被标记为过时，建议使用底层api Window

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.windowing.assigners.EventTimeSessionWindows;
   import org.apache.flink.streaming.api.windowing.time.Time;
   
   /**
    * @author kyle on 2021-11-23 8:05 上午
    */
   public class WindowTest02_TimeWindow {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStream = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         mapStream.keyBy("id")
   //              .countWindow(100, 5); //计数滑动窗口
                 .countWindow(100);  //计数滚动窗口
   //              .timeWindow(Time.seconds(15));  //时间滚动窗口
   //              .timeWindow(Time.seconds(15), Time.seconds(5));  //时间滑动窗口
         
         env.execute();
   
      }
   
   }
   
   ~~~

## 5.3 窗口函数 window function

> - 定义了要对窗口中收集的数据做的计算操作
> - 可以分为两类
>   - 增量聚合函数
>     - 实时性更好，计算效率更好，延迟更低
>   - 全窗口函数
>     - 可以拿到上下文信息， 更灵活
>     - 前面计算比较复杂且结果对最后结果意义不大的可以考虑使用全窗口函数
>     - 归并操作，计算中位数或者百分比分位数 等

### 5.3.1 增量聚合函数 (incremental aggregation functions)

> - 每条数据到来就进行计算，保持一个简单的状态
> - ReduceFuntion， AggregateFunction
> - eg.  计算8点到9点的总和，每来一条就聚合一次，把结果状态记住，不输出，直到9点后立马输出

1. AggregateFunction  ---  eg. 滚动时间窗口

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.AggregateFunction;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.functions.ReduceFunction;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.datastream.WindowedStream;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.windowing.assigners.*;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.streaming.api.windowing.triggers.CountTrigger;
   import org.apache.flink.streaming.api.windowing.triggers.PurgingTrigger;
   import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
   import org.apache.hadoop.fs.shell.Count;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-11-23 8:05 上午
    */
   public class WindowTest01_TimeWindow {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
          
          DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
          SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
              @Override
              public boolean filter(String value) throws Exception {
                  return Strings.isNotBlank(value);
              }
          });
   
          env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         SingleOutputStreamOperator<Integer> aggregateStream = mapStream.keyBy("id")
   //              .window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(100)));  // 计数滚动窗口， 每100个元素开新窗口。
   //              .window(SlidingProcessingTimeWindows.of(Time.seconds(15), Time.seconds(5))); // 时间滑动窗口 15秒窗口长度， 5秒滑动长度
   //              .window(EventTimeSessionWindows.withGap(Time.minutes(1)));   // 会话窗口，1分钟没有新数据则开启新窗口
                 .window(TumblingProcessingTimeWindows.of(Time.seconds(15)))// 时间滚动窗口
   
                 .aggregate(new AggregateFunction<SensorReading, Integer, Integer>() {
                    @Override
                    public Integer createAccumulator() {
                       return 0;
                    }
   
                    @Override
                    public Integer add(SensorReading value, Integer accumulator) {
                       return accumulator + 2;
                    }
   
                    @Override
                    public Integer getResult(Integer accumulator) {
                       return accumulator;
                    }
   
                    @Override
                    public Integer merge(Integer a, Integer b) {
                       return a + b;
                    }
                 });
   
         aggregateStream.print();
         env.execute();
      }
     
   }
   ~~~

   

2. AggregateFunction  ---  eg. 滑动计数窗口

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.AggregateFunction;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.api.java.tuple.Tuple2;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
   import org.apache.flink.streaming.api.windowing.windows.GlobalWindow;
   import org.apache.flink.util.Collector;
   import org.apache.logging.log4j.util.Strings;
   
   import java.lang.annotation.Documented;
   
   /**
    * @author kyle on 2021-11-23 8:05 上午
    */
   public class WindowTest05_CountWindow_Incremental {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
          DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
          SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
              @Override
              public boolean filter(String value) throws Exception {
                  return Strings.isNotBlank(value);
              }
          });
   
          env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         // 计算平均值
          SingleOutputStreamOperator<Double> avgStream = mapStream.keyBy("id")
   //              .window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(100)));  // 计数滚动窗口， 每100个元素开新窗口。
   //              .window(SlidingProcessingTimeWindows.of(Time.seconds(15), Time.seconds(5))); // 时间滑动窗口 15秒窗口长度， 5秒滑动长度
   //              .window(EventTimeSessionWindows.withGap(Time.minutes(1)));   // 会话窗口，1分钟没有新数据则开启新窗口
                  .countWindow(5, 2)
                  .aggregate(new MyAvgTempCounter());
   
          avgStream.print();
   
         env.execute();
   
      }
   
      public static class MyAvgTempCounter implements AggregateFunction<SensorReading, Tuple2<Double, Integer>, Double>{
   
          @Override
          public Tuple2<Double, Integer> createAccumulator() {
              return new Tuple2<>(0.0, 0);
          }
   
          @Override
          public Tuple2<Double, Integer> add(SensorReading value, Tuple2<Double, Integer> accumulator) {
              return new Tuple2<>(accumulator.f0 + value.getTemperature(), accumulator.f1 + 1);
          }
   
          @Override
          public Double getResult(Tuple2<Double, Integer> accumulator) {
              return accumulator.f0 / accumulator.f1;
          }
   
          @Override
          public Tuple2<Double, Integer> merge(Tuple2<Double, Integer> a, Tuple2<Double, Integer> b) {
              return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
          }
      }
   
   }
   
   ~~~

   





### 5.3.2 全窗口函数 (full window functions)

> - 先把窗口所有数据收集起来，等到计算的时候会遍历所有函数
> - ProcessWindowFunction, WindowFunction
> - 类似批处理
> - eg. 来一条保存一条，保存所有函数，不保存状态，窗口结束的时候拿出所有数据计算再输出结果

1. ProcessWindowFunction

   > 滚动计数窗口，计算每个sensor出现温度最多次数的温度
   >
   > 需要传4个参数：
   >
   > <IN> – The type of the input value.  --- 调用keyby的流的类型
   > <OUT> – The type of the output value.  --- 窗口函数返回的类型
   > <KEY> – The type of the key.   --- 
   > <W> – The type of Window that this window function can be applied on.   --- 需要实现窗口的类型，

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.api.java.tuple.Tuple2;
   import org.apache.flink.api.java.tuple.Tuple3;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.streaming.api.windowing.windows.GlobalWindow;
   import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
   import org.apache.flink.util.Collector;
   import org.apache.logging.log4j.util.Strings;
   
   import java.util.*;
   
   /**
    * @author kyle on 2021-11-25 8:09 上午
    */
   public class WindowTest04_CountWindow_Full {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
          DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
          SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
              @Override
              public boolean filter(String value) throws Exception {
                  return Strings.isNotBlank(value);
              }
          });
   
          env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         // 计算每个sensor出现温度最多次数的温度
          SingleOutputStreamOperator<Tuple3<String, Double, Integer>> process = mapStream.keyBy("id")
                  .countWindow(3)
                  .process(new MyFrequencyFunction());
   
          process.print();
          env.execute();
   
      }
   
   
      // 自定义一个全窗口函数
      public static class MyFrequencyFunction extends ProcessWindowFunction<SensorReading, Tuple3<String, Double, Integer>, Tuple, GlobalWindow>{
          @Override
          public void process(Tuple tuple, Context context, Iterable<SensorReading> elements, Collector<Tuple3<String, Double, Integer>> out) throws Exception {
              HashMap<Double, Integer> countMap = new HashMap<Double, Integer>();
              for (SensorReading element : elements) {
                  int count = countMap.getOrDefault(element.getTemperature(), 0);
                  countMap.put(element.getTemperature(), count + 1);
              }
   
              ArrayList<Map.Entry<Double, Integer>> entries = new ArrayList<>(countMap.entrySet());
              entries.sort(new Comparator<Map.Entry<Double, Integer>>() {
                  @Override
                  public int compare(Map.Entry<Double, Integer> o1, Map.Entry<Double, Integer> o2) {
                      return o2.getValue() - o1.getValue();
                  }
              });
   
              out.collect(new Tuple3<String, Double, Integer>(tuple.getField(0), entries.get(0).getKey(), entries.get(0).getValue()));
          }
   
      }
   
   }
   
   ~~~

### 5.3.3 窗口中延迟数据处理API 

> 达到数据的尽快接近真实输出，保证数据最后的绝对准确。
>
> 注意：这种场景下只针对事件事件的窗口有意义

1. 在窗口没有关闭前允许一定时间的延迟

   > .allowedLateness --- 允许处理迟到的数据

   疑问：对迟到数据的定义，开一个8点到9点的窗口，超过9点的数据就应该算9点到10点的数据，为什么说他是8点到9点的窗口迟到的数据。这个涉及到**时间语义**的概念, 即不仅仅要看数据处理的时间，而是数据产生的时间。

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.util.OutputTag;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-11-26 8:34 上午
    */
   public class WindowTest06_allowLateness {
   
   
      public static void main(String[] args) {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
         SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
               return Strings.isNotBlank(value);
            }
         });
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         OutputTag<SensorReading> tagLate = new OutputTag<SensorReading>("late"){};
         SingleOutputStreamOperator<SensorReading> sum = mapStream.keyBy("id")
                 .window(TumblingProcessingTimeWindows.of(Time.seconds(10)))
                 .allowedLateness(Time.seconds(5))
                 .sum("temperature");
   
      }
   
   }
   
   ~~~

2. 将窗口期间无法等待到的数据单独输出处理

   > .sideOutputLateData() --- 将迟到的数据放入侧输出流
   >
   > .getSideOutput() --- 获取侧输出流

   ~~~java
   package com.kyle.api.window;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStream;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.util.OutputTag;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-11-26 8:57 上午
    */
   public class WindowTest07_lateSideOutput {
   
      public static void main(String[] args) {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
         SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
               return Strings.isNotBlank(value);
            }
         });
   
         env.setParallelism(1);
   
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         OutputTag<SensorReading> tagLate = new OutputTag<SensorReading>("late"){};
         SingleOutputStreamOperator<SensorReading> sum = mapStream.keyBy("id")
                 .window(TumblingProcessingTimeWindows.of(Time.seconds(10)))
                 .sideOutputLateData(tagLate)
                 .sum("temperature");
   
         DataStream<SensorReading> sideOutput = sum.getSideOutput(tagLate);
   
      }
   
   }
   
   ~~~

   

### 5.3.4 .trigger() 定义window关闭时机

> 定义window什么时候关闭，触发计算并输出结果

### 5.3.5 .evictor()  移除器

> 定义移除某些数据的逻辑

### 5.3.6 窗口函数总结

> **出走半生最终仍是dataStream**

1. 数据流 keyBy

   > - dataStream 经过 keyBy 后 得到 keyedStream
   > - keyedStream 经过 开窗操作 得到一个windowStream
   > - windowStream 经过聚合操作  得到一个 dataStream

2. 数据流2 不经过keyBy直接windowAll

   > - dataStream 直接 windowAll 得到 allWindowStream
   > - allWindowStream 经过 apply 等方法 得到 dataStream



## 6 时间语义和Watermark

## 6.1 时间（Time）语义

> - Event Time：事件创建的时间
> - Ingestion Time：数据进入Flink的时间
> - Processing Time：执行操作算子的本地系统时间，与机器相关
>
> <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwsa8rhe0dj30v40dkmy9.jpg" alt="flink_parallelism_01" style="zoom:70%;" />

## 6.2 时间语义的场景

>我们需要根据具体场景需要来采用不同的时间语义

### 6.2.1 哪种时间语义更重要

1. 不同的时间语义在不同的场景下侧重点不同

   以电影【星球大战】为例，

   - 1977年 第一部 星球大战4-新希望
   - 1980年 第二部 星球大战5-帝国反击战
   - 1983年 第三部 星球大战6-绝地归来
   - 1999年 第四部 星球大战1-幽灵的威胁
   - 2002年 第五部 星球大战2-克隆人的进攻
   - 2005年 第六部 星球大战3-西斯的复仇
   - 2015年 第七部 星球大战7-原力觉醒

   拍电影的时间就相当于我们说的处理时间

   星球大战电影本身的1，2，3，4，5，6，7系列就是我们说的事件时间。

   如果对电影本身的故事感兴趣，就应该关注电影本身系列的时间线

   如果是观众或者关注票房的就更关注电影拍摄时间，什么时候拍摄上映什么时候看。

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwuopice1fj30uc0dst9i.jpg" alt="flink_parallelism_01" style="zoom:70%;" />

2. 在Flink 1.12， setStreamTimeCharacteristic已经被标记为过期





## 6.3 watermark 水印

### 6.3.1 怎样避免乱序数据带来的计算不正确



### 6.3.2 watermark的作用

1. Watermark是一种衡量Event Time进展的机制，可以设定延迟触发

2. Watermark用于处理乱序事件的，而正确的处理乱序事件，通常是使用Watermark的机制结合window来实现

3. 数据流中的Watermark用于表现timestamp小于Watermark的数据，都已经到达了，因此，window的执行也是由Watermark触发的。

   >**学生秋游等车例子**
   >
   >1、约定早上9点出发
   >
   >2、实际上大部分同学会在9点01分到达，那使用watermark的机制将时间延迟1分钟，即实际9.01分发车，（输出第一个结果）
   >
   >3、还有小部分同学9.01分还没到，要9.10分前才到，那就使用window的allowedLateness，即9.01分发车了， 但慢慢开车，如果9.10分前那小部分同学通过各种方式追赶上巴士，也可以开门上车，这时候没来一个就更新一次输出。
   >
   >4、最后还有极少部分9.10分都还没来，那么就不等了，关上车门直接上高速出发
   >
   >5、这极少部分的同学就通过另外的交通方式到达目的地，相当于使用window的sideOutputLateData。

4. watermark用来让程序自己平衡延迟和结果正确性。 （权衡）

   > - 如果希望程序延迟更低更快一些，就把watermark设置小一些，这样等待乱序事件的时间就短一些，正确性相对差一些
   > - 如果希望程序的准确性更高一些，就把watermark设置大一些，这样等待乱序事件的时间就长一些，延迟会相对高一些

5. watermark的特点

   ><img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwwux3hgldj30v80ast9a.jpg" alt="flink_parallelism_01" style="zoom:80%;" />
   >
   >记录1来到后，可以插入时间戳为2的watermark
   >
   >当记录5和3来到后，可以认为事件时间已经到了时间戳5的时间，所以3是延迟数据了，这时候不能插入3的时间戳watermark，要插入5的时间戳的watermark

   - watermark是一条特殊的数据记录
   - watermark必需单调递增，以确保任务的事件时间始终在向前推进，而不是在后退
   - watermark与数据的时间戳相关

### 6.3.3 watermark的画图理解

1. 当Flink以Event Time模式处理数据流时，它会根据数据里的时间戳来处理基于时间的算子

2. 由于网络，分布式等原因，会导致乱序数据的产生，就事件的产生顺序和实际到来的顺序不一致。

3. 乱序数据会让窗口计算不准确，所以要考虑乱序数据的处理。一般是通过window+watermark结合处理。

4. 如下图时间线：

   > 设置watermark延迟时间为3秒

   1. 第1条数据分配到[0,5)窗口，因为watermark=1-3=-2，[0,5)窗口不关闭
   2. 第2条数据分配到[0,5)窗口，因为watermark=4-3=1，[0,5)窗口不关闭
   3. 第3条数据分配到[5,10)窗口，因为watermark=5-3=2，[5,10)窗口不关闭
   4. 第4条数据分配到[0,5)窗口，因为watermark=(2-3=-1)<2，取2，[0,5)窗口不关闭
   5. 第5条数据分配到[0,5)窗口，因为watermark=(3-3=0)<2，取2，[0,5)窗口不关闭
   6. 第6条数据分配到[5,10)窗口，因为watermark=6-3=3，[5,10)窗口不关闭
   7. 第7条数据分配到[5,10)窗口，因为watermark=7-3=4，[5,10)窗口不关闭
   8. 第8条数据分配到[5,10)窗口，因为watermark=5-3=2，[5,10)窗口不关闭
   9. **第9条数据分配到[5,10)窗口，因为watermark=8-3=5，[0,5)窗口关闭**
   10. 第10条数据迟到数据，本应分配[0,5)但无法分配，因为watermark=(4-3=1)<5，取5，[0,5)窗口已关闭
   11. 第11条数据分配到[5,10)窗口，因为watermark=11-3=8，[5,10)窗口不关闭
   12. 第12条数据分配到[5,10)窗口，因为watermark=12-3=9，[5,10)窗口不关闭

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwy06lx859j31c50u078g.jpg" alt="flink_parallelism_01" style="zoom:50%;" />



### 6.3.4 watermark的传递

>watermark的传递就是需要上游的任务将watermark广播给下游的任务，Flink的流中有多个子任务，每个子任务的处理速度不一样，下游子任务可能会接收到来自上游不同子任务不同的watermark时间戳，那么以哪个为准呢。
>
>依据 watermark的本质：事件时间进展到现在这个时间点，之前的数据都到齐了。

1. 取最小的partition watermark作为当前任务的事件时钟

   一个任务有4个并行的上游任务， 3个并行的下游任务，对于上游的4个任务，每个任务都会分配一个空间保存当前这个分区的watermark(partition watermark)，那么当前这个任务就是以最小的那个partition watermark作为自己的watermark，并且广播到下游任务。

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwy1ifftmrj30ss0i8q4d.jpg" alt="flink_parallelism_01" style="zoom:80%;" />



### 6.3.5 watermark的代码测试理解

1. 基于事件时间的开窗聚合：统计15秒内温度最小值

   ~~~java
   package com.kyle.api.watermark;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.eventtime.WatermarkStrategy;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.TimeCharacteristic;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
   import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
   import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.logging.log4j.util.Strings;
   
   import java.time.Duration;
   
   /**
    * @author kyle on 2021-12-01 8:48 上午
    */
   public class WatermarkTest01_EventTime {
   
      public static void main(String[] args) throws Exception {
   
   
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
          DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
          SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
              @Override
              public boolean filter(String value) throws Exception {
                  return Strings.isNotBlank(value);
              }
          });
   
          env.setParallelism(1);
          env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
          env.getConfig().setAutoWatermarkInterval(100L);
   
         SingleOutputStreamOperator<SensorReading> mapStream = inputStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         })
                 // 明确升序数据设置时间戳和watermark
   //              .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<SensorReading>() {
   //                 @Override
   //                 public long extractAscendingTimestamp(SensorReading element) {
   //                    return element.getTimestamp() * 1000L;
   //                 }
   //              })
   
                 // 1.12 版本
   //              .assignTimestampsAndWatermarks(WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5)))
   
                 // 乱序数据设置时间戳和watermark
                 .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
                     @Override
                     public long extractTimestamp(SensorReading element) {
                        return element.getTimestamp() * 1000L;
                     }
                  });
   
           // 基于事件时间的开窗聚合： 统计15秒内温度的最小值
          SingleOutputStreamOperator<SensorReading> minTempStream = mapStream.keyBy("id")
                  .window(TumblingEventTimeWindows.of(Time.seconds(15)))
                  .minBy("temperature");
   
          minTempStream.print();
          env.execute();
      }
   }
   ~~~

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwz76e8kbsj30qd0ffdj5.jpg" alt="flink_parallelism_01" style="zoom:80%;" />

   <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwz77d8hwij31ho0regqi.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

   ~~~python
   sensor_01 1547718199 35.8
   sensor_02 1547718201 15.4
   sensor_03 1547718202 6.7
   sensor_04 1547718205 38.1
   sensor_04 1547718206 38.2
   sensor_04 1547718207 38.3
   sensor_04 1547718208 38.4
   sensor_04 1547718209 38.5
   sensor_04 1547718210 38.6
   sensor_04 1547718211 38.7
   sensor_04 1547718212 38.8
   
   # 212触发窗口计算，同时watermark设置时间是2秒，窗口长度是15
   # 所以关闭的窗口是[195, 210）  ？？？  为什么窗口是[195, 210), 不是[192, 207).... 解释在6.3.6
   # 下一个窗口是[210, 225） 事件时间戳到达227才触发
   # 下下一个窗口是[225, 240)  事件时间戳到达242才会触发
   
   sensor_04 1547718213 39.0
   sensor_04 1547718214 39.1
   sensor_04 1547718215 39.2
   sensor_04 1547718216 39.3
   sensor_04 1547718217 39.4
   sensor_04 1547718218 39.5
   sensor_04 1547718219 39.6
   sensor_04 1547718220 39.7
   sensor_04 1547718221 39.8
   sensor_04 1547718222 39.9
   sensor_04 1547718223 39.10
   sensor_04 1547718224 40.0
   sensor_04 1547718225 40.1
   sensor_04 1547718226 40.2
   sensor_04 1547718227 40.3
   
   # 227 触发了[210,225)窗口的关闭计算，
   # 在[210,225)窗口中温度最低的是 210，所以输出的结果也就是210的温度
   
   sensor_04 1547718228 40.4
   sensor_04 1547718229 40.5
   sensor_04 1547718230 40.6
   sensor_04 1547718231 40.7
   sensor_04 1547718239 30.0
   sensor_04 1547718240 23.5
   sensor_04 1547718241 20.1
   sensor_04 1547718242 22.2
   
   # 242 触发了[225, 240)窗口的关闭计算，
   # 同时发现240， 241， 242 的温度更低，但是这个窗口输出的是239的温度， 
   # 因为这个窗口是[225, 240), 不包含240，241，242，在这个窗口中239的温度是最低的
   
   
   ~~~

   

### 6.3.6 窗口的起始点和偏移量

> 在6.3.5的测试中有个疑问，第一个时间窗口是[195, 210), 这个时间窗口是怎么确定的?

1. 源码

   ~~~java
   @Override
       public Collection<TimeWindow> assignWindows(
               Object element, long timestamp, WindowAssignerContext context) {
           if (timestamp > Long.MIN_VALUE) {
               if (staggerOffset == null) {
                   staggerOffset =
                           windowStagger.getStaggerOffset(context.getCurrentProcessingTime(), size);
               }
               // Long.MIN_VALUE is currently assigned when no timestamp is present
               long start =
                       TimeWindow.getWindowStartWithOffset(
                               timestamp, (globalOffset + staggerOffset) % size, size);
               return Collections.singletonList(new TimeWindow(start, start + size));
           } else {
               throw new RuntimeException(
                       "Record has Long.MIN_VALUE timestamp (= no timestamp marker). "
                               + "Is the time characteristic set to 'ProcessingTime', or did you forget to call "
                               + "'DataStream.assignTimestampsAndWatermarks(...)'?");
           }
       }
   ~~~

   ~~~java
   public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
           return timestamp - (timestamp - offset + windowSize) % windowSize;
       }
   ~~~

2. 解释

   在测试代码中，我们调用的是 事件时间语义下的滚动窗口【TumblingEventTimeWindows】,在 【TumblingEventTimeWindows】里面可以看到有【assignWindows】这个方法调用了一个最终计算起始点和偏移量的方法【getWindowStartWithOffset】，所以可以看到最终计算起始点和偏移量的公式就是：

   > timestamp - (timestamp - offset + windowSize) % windowSize;

   - 忽略offset

     假设我们先不考虑偏移量offset，timestamp是我们获取到的数据里面的时间戳，那么公式相当于：

     1. 先用timestamp加上我们设置的时间窗口长度，再对时间窗口长度取模，这个运算得到的时间戳对窗口长度的余数

     2. 用时间戳减去这个余数，得到的是我们设置窗口长度的整倍数，这个整倍数就用我们窗口的起始点

     3. eg.测试代码中第一个触发第一个窗口时的时间戳是1547718212，同时我们设置的watermark是2， 可以推算出这个窗口结束点是1547718212， 然后拿这个窗口中随意一条记录的时间戳按照公式计算如:

        >  1547718201 - (1547718201 + 15)%15 = 1547718201- 6 = 1547718195,

        即时间窗口是第一个时间窗口是[1547718195, 1547718210),因为是滚动窗口，第一个窗口确定了，后续所有窗口也固定了。

     4. 从上述看出，最后算出来一定是 从0开始， 0到15， 15到30， 一点点推移过来的， 按照这个规则， 得到的起始点和结束点一定是15的整倍数。**这个公式其实就是计算当前这条数据数据哪个窗口。**

   - 设置偏移量offset 

     > 用于一些特殊的场景，如不同时区时处理数据的要求.
     >
     > 比如我们想开1天的窗口处理数据，因为我们习惯使用北京时间UTC+08:00，而flink默认是UTC+00:00时间，如果我们不设置偏移量的话，每天开窗拿到的时间区间都会是我们的早8点到晚8点，就不是我们所认知的0点到0点的一天数据，
     >
     > 源码注释建议:
     >
     > ~~~java
     > /**Rather than that,if you are living in somewhere which is not using UTC±00:00 time,
     > 	 * such as China which is using UTC+08:00,and you want a time window with size of one day,
     > 	 * and window begins at every 00:00:00 of local time,you may use {@code of(Time.days(1),Time.hours(-8))}.
     > 	 * The parameter of offset is {@code Time.hours(-8))} since UTC+08:00 is 8 hours earlier than UTC time.
     > ~~~
     >
     > 

     1. 从上面忽略offset的计算看出，我们把offset加上去后得到的结果就不是从0开始的时间窗口，而是我们设置偏移量开始的时间窗口。

     2. 如设置偏移量为5，窗口的起始点计算如：

        > 1547718199 - (1547718199 - 5 + 15)%15 = 1547718199 - 14 = 1547718185
        >
        > 1547718201 - (1547718201 - 5 + 15)%15 = 1547718201- 1 = 1547718200

        因为窗口是左闭右开，可以看出 

        - 1547718199 这条记录应该是数据[1547718185, 1547718200)这个窗口

        - 1547718201 这条记录应该是数据[1547718200, 1547718215)这个窗口

        下图中可以看到虽然已经输入到达了3条记录，但是第一个窗口只输出了一条记录，因为这个窗口是[185, 200),前三条记录中的后两条的时间戳都大于200，不属于这个窗口，就不进行输出计算了。

        <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gx0d8u98o3j30f8060gmr.jpg" alt="flink_parallelism_01" style="zoom:100%;" />

        <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gx0dar3ypvj31fg0h8gpr.jpg" alt="flink_parallelism_01" style="zoom:50%;" />







### 6.3.7 事件时间语义下的窗口测试 - 迟到数据处理

> watermark hold不住的那些超过 watermark时间的迟到数据，就需要通过window的迟到数据处理结合。

1. 代码

   ~~~java
   package com.kyle.api.watermark;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.TimeCharacteristic;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
   import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
   import org.apache.flink.streaming.api.windowing.time.Time;
   import org.apache.flink.util.OutputTag;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-12-01 8:48 上午
    */
   public class WatermarkTest01_EventTime_Late {
   
      public static void main(String[] args) throws Exception {
   
   
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
          DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
          SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
              @Override
              public boolean filter(String value) throws Exception {
                  return Strings.isNotBlank(value);
              }
          });
   
          env.setParallelism(1);
          env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
          env.getConfig().setAutoWatermarkInterval(100L);
   
         SingleOutputStreamOperator<SensorReading> mapStream = inputStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         })
                 // 乱序数据设置时间戳和watermark
                 .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
                     @Override
                     public long extractTimestamp(SensorReading element) {
                        return element.getTimestamp() * 1000L;
                     }
                  });
   
          OutputTag<SensorReading> outputTag = new OutputTag<SensorReading>("late") {};
   
          // 基于事件时间的开窗聚合： 统计15秒内温度的最小值
          SingleOutputStreamOperator<SensorReading> minTempStream = mapStream.keyBy("id")
                  .window(TumblingEventTimeWindows.of(Time.seconds(15)))
                  .allowedLateness(Time.minutes(1))
                  .sideOutputLateData(outputTag)
                  .minBy("temperature");
   
          minTempStream.print("minTemp");
          minTempStream.getSideOutput(outputTag).print("late");
   
          env.execute();
   
      }
   
   
   }
   
   
   ~~~

2. 测试解释

   - 测试数据

     ~~~
     迟到数据测试
     sensor_01 1547718199 35.8
     sensor_01 1547718206 36.8
     sensor_01 1547718210 34.7
     sensor_01 1547718211 31
     sensor_01 1547718209 34.9
     sensor_01 1547718212 37.1
     sensor_01 1547718213 33
     sensor_01 1547718206 34.2
     sensor_01 1547718202 36
     sensor_01 1547718270 31
     sensor_01 1547718203 31.9
     sensor_01 1547718272 34
     sensor_01 1547718203 30.6
     sensor_01 1547718203 40
     sensor_01 1547718205 41
     sensor_01 1547718212 41
     ~~~

   - 测试过程解释

     <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gx4xzqrxp6j311n0u0afy.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

     <img src="https://tva1.sinaimg.cn/large/008i3skNgy1gx4xfmce55j314j0u048f.jpg" alt="flink_parallelism_01" style="zoom:50%;" />

     

     1. 212：【输出】达到[195, 210)窗口的watermark设定的延迟2秒， 输出**第1条**记录， 因为设置了**【.allowedLateness(Time.minutes(1))】**，[195, 210)窗口不会在此时关闭，会在此时基础上延迟1分钟，即272时刻才会关闭。在后续1分钟内，只要该窗口内来的每一条数据都会触发一次计算更新输出。
     2. 213：【不输出】属于[210, 225)， 但该窗口未触发计算
     3. 206：【输出】属于[195, 210)的记录，[195, 210)窗口未关闭，输出**第2条**记录，该窗口最低温度是206的34.2°
     4. 202：【输出】属于[195, 210)的记录，[195, 210)窗口未关闭，输出**第3条**记录，该窗口最低温度是206的34.2°
     5. 270：【输出】 超过227，[210, 225)窗口触发计算，输出**第4条**记录，该低温度记录211的31°
     6. 203：【输出】属于[195, 210)的记录，[195, 210)窗口未关闭，输出**第5条**记录，该窗口最低温度是203的31.9°
     7. 272：【不输出】达到[195, 210)窗口1分钟延迟的时间，[195, 210窗口在此时关闭，后续[195, 210)的数据会进入指定的侧输出流。
     8. 203：【输出】属于[195, 210)的记录，但[195, 210)窗口已关闭，不再和之前窗口共同计算最低值，直接输出**第6条**记录
     9. 203：【输出】属于[195, 210)的记录，但[195, 210)窗口已关闭，不再和之前窗口共同计算最低值，直接输出**第7条**记录
     10. 205：【输出】属于[195, 210)的记录，但[195, 210)窗口已关闭，不再和之前窗口共同计算最低值，直接输出**第8条**记录
     11. 212：【输出】属于[210, 225)的记录，[210, 225)窗口未关闭，输出第9条记录，该窗口最低温度是211的31.0°



## 7 状态管理

### 7.1、状态后端 State Backend

### 7.1.1 什么是 状态后端

1. 每传入一条数据，有状态的算子任务都会读取和更新状态
2. 由于有效的状态访问对于处理数据的低延迟很重要，因此每个并行任务都会在本地维护状态，以确保快速的状态访问
3. 状态的存储，访问以及维护，由一个可插入的组件决定，这个组件就叫做**状态后端**
4. **状态后端**主要负责两件事，本地的状态管理，以及将检查点（checkpoint）状态写入远程存储

### 7.1.2 状态后端的分类

1. **MemoryStateBackend**
   - 内存级别的状态后端，会将监控状态作为内存中的对象进行管理，将它们存在TaskManager的JVM堆上，而将checkpoint存储在JobManager的内存中
   - 特点：快速，低延迟， 但不稳定可靠，生产环境一般不适用，适用于测试场景
2. **FsStateBackend**
   - 将checkpoint存储到远程的持久化文件系统上（FilsSystem），而对于本地状态，跟MemoryStateBackend一样，也会存在TaskManager的JVM堆上
   - 特点：同时拥有内存级别的本地访问速度，和远程文件系统故障恢复的容错保证。
3. **RockDBStateBackend**
   - RockDB是facebook研发的类似no sql的数据库
   - 将所有状态序列化后，存入本地的TocksDB中。
   - 解决状态信息特别大或者随着时间不断增长的情况，因为在状态信息特别大的时候，前面两种状态后端把状态信息保存在TaskManager的JVM中，会导致OOM，就使用RockDBStateBackend。

### 7.1.3 集群中的 状态后端配置

1. flink-conf.yaml

   > 默认就是 【filesystem】
   >
   > state.backend.incremental 默认是false， rockdb支持 增量化checkpoints
   >
   > jobmanager.execution.failover-strategy：默认region， 1.9新引入的特性，区域化重启，flink会解析出所有任务的关系，当某个任务挂了，区域化重启就可以只重启有关系的任务，不需要所有任务重启

   ~~~yaml
   #==============================================================================
   # Fault tolerance and checkpointing
   #==============================================================================
   
   # The backend that will be used to store operator state checkpoints if
   # checkpointing is enabled. Checkpointing is enabled when execution.checkpointing.interval > 0.
   #
   # Execution checkpointing related parameters. Please refer to CheckpointConfig and ExecutionCheckpointingOptions for more details.
   #
   # execution.checkpointing.interval: 3min
   # execution.checkpointing.externalized-checkpoint-retention: [DELETE_ON_CANCELLATION, RETAIN_ON_CANCELLATION]
   # execution.checkpointing.max-concurrent-checkpoints: 1
   # execution.checkpointing.min-pause: 0
   # execution.checkpointing.mode: [EXACTLY_ONCE, AT_LEAST_ONCE]
   # execution.checkpointing.timeout: 10min
   # execution.checkpointing.tolerable-failed-checkpoints: 0
   # execution.checkpointing.unaligned: false
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
   ~~~

### 7.1.4 代码中 状态后端的设置

1. env设置

   > RocksDB需要引入jar包
   >
   > ```xml
   > <dependency> 
   >     <groupId>org.apache.flink</groupId>
   >     <artifactId>flink-statebackend-rocksdb_${scala.binary.version}</artifactId>
   >     <version>${flink.version}</version>
   > </dependency>
   > ```

   ~~~java
   package com.kyle.api.StateBackend;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.contrib.streaming.state.RocksDBStateBackend;
   import org.apache.flink.runtime.state.filesystem.FsStateBackend;
   import org.apache.flink.runtime.state.memory.MemoryStateBackend;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   
   /**
    * @author kyle on 2021-12-08 7:50 上午
    */
   public class StateTest01_StateBackend {
   
      public static void main(String[] args) throws Exception {
   
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
         DataStreamSource<String> dataStream = env.readTextFile("/Users/kyle/Documents/kyle/project/learn/flink/src/main/resources/sensor.txt");
   
         env.setParallelism(1);
   
   
         // 设置状态后端
         env.setStateBackend(new MemoryStateBackend());
         env.setStateBackend(new FsStateBackend(""));
         env.setStateBackend(new RocksDBStateBackend(""));
   
   
         SingleOutputStreamOperator<SensorReading> mapStream = dataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         mapStream.print();
   
         env.execute();
   
      }
   
   }
   
   ~~~

   

## 8 ProcessFunction API （底层API）

### 8.1 简述和分类

### 8.1.1 简述

1. **转换算子**是无法访问时间的时间戳信息和水位线信息的，如MapFunction这样的map转换算子无法访问时间戳或者当前事件的事件时间，而这类信息在某些场景下极为重要。
2. 基于此，**DataStream API** 提供了一系列Low-Level转换算子。可以**访问时间戳，watermark以及注册定时事件**。还可以输出特定的一些事件，如超时事件等。
3. Process Function用来构建事件驱动的应用以及实现自定义的业务逻辑（使用之前的window函数和算子无法实现）。例如Flink SQL就是使用Process Function实现的。

### 8.1.2 Porcess Function 的分类

1. ProcessFunction

2. KeyedProcessFunction

3. CoProcessFunction

   - 合流

4. ProcessJoinFunction

5. BroadcastProcessFunction

   - 广播流

6. KeyedBroadcastProcessFunction

7. ProcessWindowFunction

8. ProcessAllWindowFunction

   

## 8.2 KeyedProcessFunction

### 8.2.1 简述

1. KeyedProcessFunction 用来操作KeyedStream。KeyedProcessFunction会处理流的每一个元素，输出为0个，1个或者多个元素。

2. 所有的Process Function都继承自RichFunction接口，所以都有 open(), close()和getRuntimeContext()等方法

3. KeyedProcessFunction<K, I, O>还额外提供了两个方法：
   1. processElement(I value, Context ctx, Collector<O> out)，流中的每一个元素都会调用这个方法，调用结果将会放在Collector数据类型中输出。Context可以访问元素的时间戳，元素的key，以及TimerService时间服务。Context还可以将结果输出到别的流(side outputs)
   2. onTimer(long timestamp, OnTimerContext ctx, Collector<O> out)是一个回调函数。当之前注册的定时器触发时调用。参数timestamp为定时器所设定的触发的时间戳。Collector为输出结果集合。OnTimerContext和processElement的Context参数一样，提供了上下文的信息，例如定时器触发的时间信息（事件时间或者处理时间）。
   
### 8.2.2 代码
1. demo代码

   ~~~java
   package com.kyle.api.processFunction;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.state.ValueState;
   import org.apache.flink.api.common.state.ValueStateDescriptor;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.configuration.Configuration;
   import org.apache.flink.contrib.streaming.state.RocksDBStateBackend;
   import org.apache.flink.runtime.state.filesystem.FsStateBackend;
   import org.apache.flink.runtime.state.memory.MemoryStateBackend;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
   import org.apache.flink.util.Collector;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-12-08 8:35 上午
    */
   public class PorcessTuncTest01_Keyed {
   
      public static void main(String[] args) throws Exception {
   
   
         // socket 文本流
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
         SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
               return Strings.isNotBlank(value);
            }
         });
   
         env.setParallelism(1);
   
         // 转换成 SensorReading 类型
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
         // 测试 KeyedProcessFunction， 先分组然后自定义处理
         mapStream.keyBy("id")
                 .process(new MyProcess())
                 .print();
         env.execute();
   
      }
   
   
      // 实现自定义的处理函数
      public static class MyProcess extends KeyedProcessFunction<Tuple, SensorReading, Integer>{
         ValueState<Long> tsTimerState;
   
         @Override
         public void open(Configuration parameters) throws Exception {
            tsTimerState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("ts-timer", Long.class));
         }
   
         @Override
         public void processElement(SensorReading value, Context ctx, Collector<Integer> out) throws Exception {
            out.collect(value.getId().length());
   
            // context
            ctx.timestamp();
            ctx.getCurrentKey();
            // 侧输出流输出
   //         ctx.output();
            ctx.timerService().currentProcessingTime();
            ctx.timerService().currentWatermark();
            // 设定定时器： 注意， 参数时间是绝对时间戳 , 当前时间延迟10秒
   //         ctx.timerService().registerEventTimeTimer((value.getTimestamp() + 10) * 1000);
            ctx.timerService().registerProcessingTimeTimer(ctx.timerService().currentProcessingTime() + 5000L);
            tsTimerState.update(ctx.timerService().currentProcessingTime() + 5000L);
   
            // 取消定时器
   //         ctx.timerService().deleteEventTimeTimer(tsTimerState.value());
   
         }
   
         @Override
         public void onTimer(long timestamp, OnTimerContext ctx, Collector<Integer> out) throws Exception {
            System.out.println(timestamp + " 定时器触发");
         }
   
   
         @Override
         public void close() throws Exception {
            tsTimerState.clear();
         }
      }
   
   }
   
   ~~~

   

## 8.3 Process Function 的应用例子

### 8.3.1 TimerService 和 定时器（Timers）

1. 10s内温度上升，则输出报警信息

   > 逻辑：
   >
   > - 通过keyby对温度传感器id进行分组，避免传感器混乱
   >
   > - 自定义一个温度连续上升的监控类
   >
   >   1. 定义一个私有属性，时间间隔，作为接收设定的统计时间长度的变量
   >   2. 还要定义两个状态，分别保存上一次的温度值和定时器时间戳，用于跟新来的数据对比
   >   3. 在open的方法中初始化两个状态
   >
   >      - 温度值状态的初始化给一个Double的最小值，认为从第一条数据来就是上升趋势
   >      - 定时器定义的时候不需要赋值。
   >   4. 在processElement方法中进行逻辑处理
   >
   >      - 从状态属性中取出上一次的状态温度值和定时器时间戳
   >      - 将上一次的温度值和新来记录的温度值对比，
   >        - 如果新来记录的温度值比上一次的温度值高：
   >          - 上一次的定时器时间戳为空的时候，认为温度是连续上涨，且本周期还没开始定时，所以可以开始设定定时器，同时把定时器的时间戳和新来的温度值更新到状态中。
   >          - 上一次的定时器时间戳为空的时候，认为温度是连续上涨，但是本周期已经有定时器了，只需要将新来的温度更新到状态中即可
   >        - 如果新来记录的温度值比上一次的温度值低：
   >          - 同时定时器时间戳不为空的时候（正常来说不会为空，但要做判断预防空指针），将该定时器时间戳删除，同时将定时器时间戳状态清空，最后也要将新来的温度更新到状态中
   >   5. 在onTimer方法中
   >      - 只要触发了onTimer方法，都会认为符合了设定时间内连续上升的条件，那么直接输出告警信息
   >      - 然后清空定时器时间戳状态
   >   6. 在close方法中把上一次的温度值状态清空即可
   >
   > 

   ~~~java
   package com.kyle.api.processFunction;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.api.common.state.ValueState;
   import org.apache.flink.api.common.state.ValueStateDescriptor;
   import org.apache.flink.api.java.tuple.Tuple;
   import org.apache.flink.configuration.Configuration;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
   import org.apache.flink.util.Collector;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-12-09 8:35 上午
    */
   public class PorcessTuncTest02_Timer {
   
      public static void main(String[] args) throws Exception {
   
   
         // socket 文本流
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
         SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
               return Strings.isNotBlank(value);
            }
         });
   
         env.setParallelism(1);
   
         // 转换成 SensorReading 类型
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
   
         // 测试 KeyedProcessFunction， 先分组然后自定义处理
         mapStream.keyBy("id")
                 .process(new TempConsIncreWarning(10))
                 .print();
   
         env.execute();
   
      }
      
      // 实现自定义的处理函数
      public static class TempConsIncreWarning extends KeyedProcessFunction<Tuple, SensorReading, String>{
         // 定义私有属性， 当前统计的时间间隔
         private Integer interval;
   
         public TempConsIncreWarning(Integer interval) {
            this.interval = interval;
         }
   
         //定义状态， 保存上一次的温度值，定时器时间戳
         private ValueState<Double> lastTempState;
         private ValueState<Long> timerTsState;
   
         @Override
         public void open(Configuration parameters) throws Exception {
            lastTempState = getRuntimeContext().getState(new ValueStateDescriptor<Double>("last-temp", Double.class, Double.MIN_VALUE));
            timerTsState = getRuntimeContext().getState(new ValueStateDescriptor<Long>("timer-ts", Long.class));
         }
   
         @Override
         public void processElement(SensorReading value, Context ctx, Collector<String> out) throws Exception {
            //取出状态值
            Double lastTemp = lastTempState.value();
            Long timerTs = timerTsState.value();
   
            //如果温度上升并且没有定时器的时候，注册10秒后的定时器， 开始等待
            if (value.getTemperature() > lastTemp && timerTs == null){
               Long ts = ctx.timerService().currentProcessingTime() + interval * 1000L;
               ctx.timerService().registerProcessingTimeTimer(ts);
               timerTsState.update(ts);
            }
   
            //如果温度下降， 删除定时器
            else if (value.getTemperature() < lastTemp && timerTs != null){
               ctx.timerService().deleteProcessingTimeTimer(timerTs);
               timerTsState.clear();
            }
   
            // 更新温度状态
            lastTempState.update(value.getTemperature());
   
         }
   
         @Override
         public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
            //定时器触发， 输出报警信息
            out.collect("传感器" + ctx.getCurrentKey().getField(0) + "温度值连续" + interval + "s上升");
            timerTsState.clear();
         }
   
   
         @Override
         public void close() throws Exception {
            lastTempState.clear();
         }
      }
   
   
   }
   
   
   ~~~

   


### 8.3.2 侧输出流(SideOutput)

1. 高低温分流

   > - 侧输出流通过OutputTag来实现
   >   - 定义OutputTag的时候注意是以匿名类的方式，后面需要加上大括号。
   > - 分流的时候使用到底层API的context才会有从output API。
   > - 获取分流的方式
   >   - 使用主流调用getSideOutput
   >   - 主流需要是 SingleOutputStreamOperator, DataStream是没有getSideOutput方法的。

   ~~~java
   package com.kyle.api.processFunction;
   
   import com.kyle.bean.SensorReading;
   import org.apache.flink.api.common.functions.FilterFunction;
   import org.apache.flink.api.common.functions.MapFunction;
   import org.apache.flink.streaming.api.datastream.DataStreamSource;
   import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
   import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
   import org.apache.flink.streaming.api.functions.ProcessFunction;
   import org.apache.flink.util.Collector;
   import org.apache.flink.util.OutputTag;
   import org.apache.logging.log4j.util.Strings;
   
   /**
    * @author kyle on 2021-12-10 7:25 上午
    */
   public class ProcessFuncTest03_SideOutput {
   
      public static void main(String[] args) throws Exception {
   
         // socket 文本流
         StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   
         DataStreamSource<String> inputStream = env.socketTextStream("localhost", 9999);
         SingleOutputStreamOperator<String> filterStream = inputStream.filter(new FilterFunction<String>() {
            @Override
            public boolean filter(String value) throws Exception {
               return Strings.isNotBlank(value);
            }
         });
   
         env.setParallelism(1);
   
         // 转换成 SensorReading 类型
         SingleOutputStreamOperator<SensorReading> mapStream = filterStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
               String[] fields = s.split(" ");
               return new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            }
         });
   
   
         // 定义一个OutputTag， 用来表示侧输出 低温流
         OutputTag<SensorReading> lowTempTag = new OutputTag<SensorReading>("lowTemp") {};
   
         // 测试ProcessFunction 自定义侧输出流实现分流操作
         SingleOutputStreamOperator<Object> highTempStream = mapStream.process(new ProcessFunction<SensorReading, Object>() {
            @Override
            public void processElement(SensorReading value, Context ctx, Collector<Object> out) throws Exception {
               // 温度高于30， 高温流输出到主流， 低于30度， 输出到侧输出流
               if(value.getTemperature() > 30){
                  out.collect(value);
               }
               else {
                  ctx.output(lowTempTag, value);
               }
   
            }
         });
   
         highTempStream.print("high-temp");
   
         highTempStream.getSideOutput(lowTempTag).print("low-temp");
   
         env.execute();
   
      }
   
   }
   
   ~~~

# 9 Flink 容错机制







   

























​	











