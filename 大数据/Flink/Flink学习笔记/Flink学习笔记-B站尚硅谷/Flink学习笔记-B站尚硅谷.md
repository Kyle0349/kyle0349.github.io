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

   

   



​		







- 相关参数：

  taskmanager.memory.process.size

| Flink Memory Model |      | taskmanager.memory.process.size(M) |         |
| ------------------ | ---- | ---------------------------------- | ------- |
|                    |      | 2048m                              | 4096m   |
| Framework Heap     |      | 128                                | 128     |
| Task Heap          |      | 538                                | 1454.08 |
| Managed Memory     |      | 635                                | 1372.16 |
| Framework Off-Heap |      | 128                                | 128     |
| Task Off-Heap      |      | 0                                  | 0       |
| Network            |      | 159                                | 343     |
| JVM Metaspace      |      | 256                                | 256     |
| JVM Overhead       |      | 205                                | 410     |





## 3.4 思考

- 怎样实现并行计算（019）

  答：多线程，不同的任务分配到不同的线程上。不同的slot其实就是执行不同的任务。

- 并行任务，需要占用多少slot（019）

  答：跟当前任务设置并行度最大值有关

- 一个流处理程序，到底包含多少个任务（019）

  答：











