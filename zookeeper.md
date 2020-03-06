# Zookeeper ZAB协议分析

[![img](https://cdn2.jianshu.io/assets/default_avatar/6-fd30f34c8641f6f32f5494df5d6b8f3c.jpg)](https://www.jianshu.com/u/c867e3867fcb)

[吕宗胜ZJU](https://www.jianshu.com/u/c867e3867fcb)关注

2018.04.15 14:45:54字数 1,189阅读 1,686

# 1. ZAB协议

ZAB协议（Zookeeper Atomic Broadcast Protocol）是Zookeeper系统专门设计的一种支持崩溃恢复的原子广播协议。Zookeeper使用该协议来实现分布数据一致性并实现了一种主备模式的系统架构来保持各集群中各个副本之间的数据一致性。

ZAB协议理论与Zookeeper对该协议的实现还是存在一些差别，本文将针对ZAB的协议本身和Zookeeper的实现两个维度来介绍。

## 1.2 ZAB协议的四阶段

在详细介绍ZAB协议之前，我们先介绍一下ZAB协议中的一些常用术语。

#### 服务器的状态

- **Looking**：该状态表示集群中不存在群首节点，进入群首选举过程。
- **Leading**：群首状态，表示该服务器是群首节点。
- **Following**：跟随者状态，表示该服务器是群首的Follow节点。
  **注意**：服务器默认是Looking状态

### 节点的持久数据状态

- **history**: 当前节点接收到的事务提议的log
- **acceptedEpoch**：follower节点已经接受的leader更改年号的NEWEPOCH提议
- **currentEpoch**：当前所处的年代
- **lastZxid**：history中最近接收到的提议的zxid

### a. 选举阶段

在选举阶段，只要有节点有集群中超过半数的节点支持，该节点就会被作为准Leader。该节点暂不会作为Leader节点来提供服务，能否真正作为Leader节点，还依赖与后续的阶段能否正常完成。

### b. 发现阶段

在选举出Leader节点后，集群进入发现阶段。Follow与准Leader进行通信，同步集群中各个节点的状态，确认集群中最新提议历史。



![img](https://upload-images.jianshu.io/upload_images/3399477-849e4757059c50e1.png?imageMogr2/auto-orient/strip|imageView2/2/w/891/format/webp)

ZAB协议发现阶段

### c. 同步阶段

在完成发现阶段后，准Leader可以获取集群中最新的提议历史。准Leader在该阶段会把最新的提议历史同步到集群中的所有节点。当同步完成时，准Leader才会真正成为Leader，执行Leader的工作。



![img](https://upload-images.jianshu.io/upload_images/3399477-b5d64b852889b98c.png?imageMogr2/auto-orient/strip|imageView2/2/w/882/format/webp)

ZAB协议同步阶段

### d. 广播阶段

到了该阶段，Zookeeper才能真正对外提供事务服务，leader可以进行消息的广播。



![img](https://upload-images.jianshu.io/upload_images/3399477-47e4753af80bfc55.png?imageMogr2/auto-orient/strip|imageView2/2/w/881/format/webp)

ZAB协议广播阶段

# 2. Zookeeper的仲裁原则

对于ZAB协议来说，遵循如下的仲裁原则：***少数服从多数\***
深入解读，可以得出如下的几点：

- 1. 群首选举过程中有超过一半的节点达成一致则选举过程结束。
- 1. 事务的确认同样遵循该原则，只要得到半数以上的支持，则表示事务成功。
- 1. 少数服从多数可以保证集群分裂也存在至少一个公共节点。
- 1. Zookeeper的集群数配置奇数更为合理，因为n与n+1的容错是相等的，n这里为奇数。

# 3. Zookeeper对ZAB协议的实现

Java版本对ZAB协议的实现与原理有一定的区别，ZAB的实现只有三个阶段：

- 选举阶段
- 恢复阶段（发现阶段+同步阶段）
- 广播阶段

## 3.1. 选举阶段

服务选举阶段，要求服务器之间两两相交，下面我们详细介绍一下Zookeeper服务器间的连接方式。
首先，每个服务器会记录本服务器自身的sid。sid由服务器的配置文件指定。
其次，Zookeeper只允许sid较大的服务器与sid较小的服务器建立连接。这样可以避免建立多余的连接。

在选举阶段，Zookeeper有不同的算法来实现群首的选举，部分算法已经废弃，这里我们介绍其中的FastLeaderElection算法。

群首选举的流程图如下：



![img](https://upload-images.jianshu.io/upload_images/3399477-df215290ceb05799.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/700/format/webp)

群首选举步骤

选举过程中比较重要的一步是判断是否变更选票，这里的详细判断逻辑如下：

1. 优先选择epoch较大
2. epoch相等时，优先选择zxid较大的
3. epoch和zxid都相等时，选择server id较大的

### 3.2 恢复阶段

该阶段，follower同步自己的最新的zxid给leader，leader来决定如何同步。
**注意:**

1. 同步事务时，Zookeeper根据oldThreshold来判断是同步相差部分还是全量数据

2. 对于leader zxid之后的事务，leader会发送trunc指令来中支
   恢复的具体流程如下：

   ![img](https://upload-images.jianshu.io/upload_images/3399477-8764c0a1ff527a3e.png?imageMogr2/auto-orient/strip|imageView2/2/w/921/format/webp)

   image.png

### 3.3 广播阶段

该阶段接收服务并进行事务广播，不过详细介绍。

参考资料：https://www.jianshu.com/p/9f3a9528524f