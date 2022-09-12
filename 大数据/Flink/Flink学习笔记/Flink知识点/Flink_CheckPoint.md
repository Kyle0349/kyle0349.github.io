



理解一下  

enableCheckpointing  

setMinPauseBetweenCheckpoints 

setMaxConcurrentCheckpoints

的使用场景



```java
// 5秒触发一次checkpoint
// enableCheckpointing 是这是上一个checkpoint的开始时间到下一个checkpoint的开始时间点的间隔时间
env.enableCheckpointing(5* 1000L);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// checkpoint 超时时间
env.getCheckpointConfig().setCheckpointTimeout(10 * 1000L);
env.getCheckpointConfig().setMaxConcurrentCheckpoints(2);
// 两个checkpoint之间的最小间隔， 设置了这个参数后， 就不会同时存在两个checkpoint
// setMinPauseBetweenCheckpoints 是设置上一个checkpoint结束到下一个checkpoint的间隔时间
// 为了防止有的checkpoint超时：
// 例如：时间线为： 0 --- 5 --- 10 --- 15 --- 20
// 在5秒的时候开始第一次checkpoint， 该checkpoint花了9秒，
// 如果没有设置 setMinPauseBetweenCheckpoints， 在 10秒的时候， 下一个checkpoint就会开始
// 如果设置了 setMinPauseBetweenCheckpoints = 3， 那么10秒的时候就不会启动checkpoint， 在14秒完成了第一个checkpoint后，再过3秒，即17秒的时候才会开始下一次的checkpoint
//
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(3 * 1000L);
```



