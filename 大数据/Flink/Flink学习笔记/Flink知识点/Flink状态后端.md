



## 什么是状态

对我们进行记住多个event的操作的时候，比如说窗口，那么我们称这种操作是有状态的，是需要缓存数据的。Flink 需要知道状态，以便使用检查点 和保存点使其容错 。简单的说就是Flink的一个存储单元。

  从 Flink 1.11 开始，检查点可以在对齐或不对齐的情况下进行。


  状态分为两种：键值状态（KeyState）和（OperatorState）非键值状态（算子状态）。



```


可用的KeydeState
ValueState<T>：保留一个可以更新和检索的值：设置update(T)和T value()；
ListState<T>：保留一个元素列表，可以追加、遍历Iterable和覆盖：add(T) ， addAll(List) ， Iterable Iterable<T> get() ， update(List)；
ReducingState<T>：只保留一个值，但是这个值代表添加到State的所有值的聚合，故为reduce。
AggregatingState<IN, OUT>：只保留一个值，但是这个值代表添加到State的所有值的聚合。与ReducingState不同的是，IN的元素可能会不一样。
MapState<UK, UV>：保留一个Map，可以增加（put(UK, UV)，putAll(Map<UK, UV>)），检索（get(UK)），遍历（entries()->key()和value()）和判空（isEmpty()）；
所有类型的状态也有一个方法clear()可以清除当前活动键的状态，即输入元素的key。
```

