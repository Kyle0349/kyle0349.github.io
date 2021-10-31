# 尚硅谷Flink入门学习笔记



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

  <img src="https://raw.githubusercontent.com/Kyle0349/kyle0349.github.io/main/大数据/Flink/Flink学习笔记/尚硅谷Flink入门学习笔记/pic/spark_dstream_01.png" alt="spark streaming dstream" style="zoom:50%;" />

+ Flink

  Flink 是基于**事件**驱动的，事件可以理解为消息。事件驱动的应用程序是一种状态应用程序，它会从一个或者多个流中注入事件，通过触发计算更新状态，或外部动作对注入的事件作出反应。

  <img src="https://raw.githubusercontent.com/Kyle0349/kyle0349.github.io/main/大数据/Flink/Flink学习笔记/尚硅谷Flink入门学习笔记/pic/flink_stream_01.png" style="zoom:50%;" />





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

### 3.1.1 <font size=4, color='blue'>Job Manager(作业管理器)</font>

1. 控制一个应用程序执行的主程序，每个应用程序都会被一个不同的JobManager所控制执行。
2. JobManager会先接收到要执行的应用程序，这个应用程序会包括：JobGraph(作业图)，logical dataflow graph(逻辑数据流图)，和打包了所有的类，库以及其他资源的JAR包
3. JobManager把JobGraph转换成一个物理层面的数据流图，也就是“执行图”（ExecutionGraph）。
4. JobManager会想ResourceManager（资源管理器）请求执行任务必要的资源，也就是TaskManager（任务管理器）的slot（插槽）。一旦获取到足够的资源，就会将“执行图”发到真正运行它们的TaskManager上。
5. 在TaskManager运行过程中，JobManager会负责所有需要中央协调的操作，比如checkpoints(检查点)的协调



### 3.1.2 <font size=4, color='blue'>Task Manager(任务执行器)</font>

1. Flink中的工作进程。通常在Flink中会有多个TaskManager运行，每一个TaskManager都包含了一定数量的slot（插槽）。插槽的数量限制了TaskManager能够执行的任务数量
2. 启动后，TaskManager会向资源管理器注册她的插槽，收到资源管理器的指令后，TaskManager就会将一个或多个slot提供给JobManager调用，JobManager就可以向slot分配task来执行。
3. 在执行的过程中，一个TaskManager可以跟其它运行在同一个程序的TaskManager交换数据，

### 3.1.3 <font size=4, color='blue'>Resource Manager(资源管理器)</font>

1. 主要负责管理TaskManager的slot， slot是Flink中定义的处理资源单元
2. Flink为不同的环境和资源管理工具提供了不同的资源管理器，比如YARN,Mesos,K8s,以及standalone部署
3. 当JobManager申请slot资源时，ResourceManager会将有空闲slot的TaskManager分配给JobManager。如果ResourceManager没有足够的slot来满足JobManager的请求，它还可以向资源提供平台发起会话，已提供启动TaskManager进程的容器。

### 3.1.4 <font size=4, color='blue'>Dispacher(分发器)</font>

1. 可以跨作业运行，为【应用提交】提供了rest接口
2. 当一个应用被提交执行时，Dispacher就会启动并将应用移交给一个JobManager
3. Dispatcher也会启动一个Web UI，用来方便地展示和监控作业的执行信息
4. Dispatcher在架构中不是必需的。处决于【应用提交】运行的方式





## 3.2 任务提交流程



## 3.3 任务调度原理





## 3.4 Slot和任务调度

### 3.4.1 Parallelism（并行度）









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











