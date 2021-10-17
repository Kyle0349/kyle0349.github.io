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






# 2.Flink的流式梳理API

+ a
+ b
  + c

# 3.Flink SQL

+ 1







