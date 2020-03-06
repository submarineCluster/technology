# RabbitMQ 高可用之镜像队列

 如果RabbitMQ集群只有一个broker节点，那么该节点的失效将导致整个服务临时性的不可用，并且可能会导致message的丢失（尤其是在非持久化message存储于非持久化queue中的时候）。可以将所有message都设置为持久化，并且使用持久化的queue，但是这样仍然无法避免由于缓存导致的问题：因为message在发送之后和被写入磁盘并执行fsync之间存在一个虽然短暂但是会产生问题的时间窗。通过publisher的confirm机制能够确保客户端知道哪些message已经存入磁盘，尽管如此，一般不希望遇到因单点故障导致服务不可用。

如果RabbitMQ集群是由多个broker节点构成的，那么从服务的整体可用性上来讲，该集群对于单点失效是有弹性的，但是同时也需要注意：尽管exchange和binding能够在单点失效问题上幸免于难，但是queue和其上持有的message却不行，这是因为queue及其内容仅仅存储于单个节点之上，所以一个节点的失效表现为其对应的queue不可用。

举例说明一下，如果一个MQ集群由三个节点组成(MQ集群节点的模式也是有讲究的，一般三个节点会有一个RAM，两个DISK)，exchange、bindings 等元数据会在三个节点之间同步，但queue上的消息是不会同步的，且不特殊设置的情况下，Queue只会在一个节点存在。可能有的同学会提另一个问题，我从三个MQ几点的监控面板，都可以看到这个Queue？这个是对的，这是由于Queue的元数据也是在三个节点之间同步，但Queue的实际存储只会在一个节点。我们发送消息到指定Queue，其实是发送消息到指定节点下的Queue。如下图所示，消息发送至队列testQueue，无论发送者通过哪个MQ节点执行发送，其最终的执行都会是在MQ03节点执行消息的存储。



![img](https://user-gold-cdn.xitu.io/2018/12/28/167f36e554748f68?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**说到这儿，可能有的小伙伴就要问了？说好的，RabbitMQ集群提供高可用性呢。**

分析一下，RabbitMQ集群搭建完成后，如果不进行任何高可用配置，会有哪些**问题**呢？

1. 单点故障会导致消息丢失：如果MQ03节点故障，那么MQ03 中的消息就会丢失
2. 无法最大化的利用MQ提供，提升执行效率：既然每次发送到队列testQueue的消息都会在MQ03节点存储，那么何必搭建集群。

引入RabbitMQ的镜像队列机制，将queue镜像到cluster中其他的节点之上。在该实现下，如果集群中的一个节点失效了，queue能自动地切换到镜像中的另一个节点以保证服务的可用性。在通常的用法中，针对每一个镜像队列都包含一个master和多个slave，分别对应于不同的节点。slave会准确地按照master执行命令的顺序进行命令执行，故slave与master上维护的状态应该是相同的。**除了publish外所有动作都只会向master发送，然后由master将命令执行的结果广播给slave**们，故看似从镜像队列中的消费操作实际上是在master上执行的。
一旦完成了选中的slave被提升为master的动作，发送到镜像队列的message将不会再丢失：publish到镜像队列的所有消息总是被直接publish到master和所有的slave之上。这样一旦master失效了，message仍然可以继续发送到其他slave上。

简单来说，镜像队列机制就是将队列在三个节点之间设置主从关系，消息会在三个节点之间进行自动同步，且如果其中一个节点不可用，并不会导致消息丢失或服务不可用的情况，提升MQ集群的整体高可用性。

先来看下设置镜像队列后的效果： 镜像队列会出现+2标识。

![img](https://user-gold-cdn.xitu.io/2018/12/28/167f36e55636eef9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



**1.设置队列为镜像队列：How**

两种方式：

1. 通过监控面板设置

   ![img](https://user-gold-cdn.xitu.io/2018/12/28/167f36e55430e5bf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

   

2. 通过命令设置

```
rabbitmqctl set_policy [-p Vhost] Name Pattern Definition [Priority]

-p Vhost： 可选参数，针对指定vhost下的queue进行设置
Name: policy的名称
Pattern: queue的匹配模式(正则表达式)
Definition：镜像定义，包括三个部分ha-mode, ha-params, ha-sync-mode
        ha-mode:指明镜像队列的模式，有效值为 all/exactly/nodes
        all：表示在集群中所有的节点上进行镜像
        exactly：表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
        nodes：表示在指定的节点上进行镜像，节点名称通过ha-params指定
        ha-params：ha-mode模式需要用到的参数
        ha-sync-mode：进行队列中消息的同步方式，有效值为automatic和manual
priority：可选参数，policy的优先级复制代码
```

请注意一个事实，镜像配置的pattern 采用的是正则表达式匹配，也就是说会匹配一组。



**RabbitMQ集群节点失效，MQ处理策略**：

如果某个slave失效了，系统处理做些记录外几乎啥都不做：master依旧是master，客户端不需要采取任何行动，或者被通知slave失效。
如果master失效了，那么slave中的一个必须被选中为master。**被选中作为新的master的slave通常是最老的那个**，因为最老的slave与前任master之间的同步状态应该是最好的。然而，特殊情况下，**如果存在没有任何一个slave与master完全同步的情况，那么前任master中未被同步的消息将会丢失。**



**镜像队列消息的同步：**

将新节点加入已存在的镜像队列时，默认情况下ha-sync-mode=manual，镜像队列中的消息不会主动同步到新节点，除非显式调用同步命令。当调用同步命令后，队列开始阻塞，无法对其进行操作，直到同步完毕。当ha-sync-mode=automatic时，新加入节点时会默认同步已知的镜像队列。由于同步过程的限制，所以不建议在生产的active队列（有生产消费消息）中操作。

```
rabbitmqctl list_queues name slave_pids synchronised_slave_pids   查看那些slaves已经完成同步复制代码
rabbitmqctl sync_queue name    手动的方式同步一个queue复制代码
rabbitmqctl cancel_sync_queue name 取消某个queue的同步功能 复制代码
```

以上针对消息同步的命令，均可以通过监控界面来进行操作，最终也是通过这些操作命令执行。



说明：

1. 镜像队列不是负载均衡，镜像队列无法提升消息的传输效率，或者更进一步说，由于镜像队列会在不同节点之间进行同步，会消耗消息的传输效率。

2. 对exclusive队列设置镜像并不会有任何作用，因为exclusive队列是连接独占的，当连接断开，队列自动删除。所以实际上这两个参数对exclusive队列没有意义。那么有哪些队列是exclusive呢？一般来说，发布订阅队列及设置了该参数的队列都是exclusive 排他性队列。 如何确定一个队列是不是排他性队列呢？ 如果队列的features包含Excl，就代表它是排他性队列。

   

   

**镜像队列中某个节点宕掉的后果：**

当slave宕掉了，除了与slave相连的客户端连接全部断开之外，没有其他影响。

当master宕掉时，会有以下连锁反应：

\1. 与master相连的客户端连接全部断开；
2.选举最老的slave节点为master。若此时所有slave处于未同步状态，则未同步部分消息丢失；
3.新的master节点requeue所有unack消息，因为这个新节点无法区分这些unack消息是否已经到达客户端，亦或是ack消息丢失在老的master的链路上，亦或者是丢在master组播ack消息到所有slave的链路上。所以处于消息可靠性的考虑，requeue所有unack的消息。此时客户端可能有重复消息；
4.如果客户端连着slave，并且Basic.Consume消费时指定了x-cancel-on-ha-failover参数，那么客户端会受到一个Consumer Cancellation Notification通知。如果未指定x-cancal-on-ha-failover参数，那么消费者就无法感知master宕机，会一直等待下去。
这就告诉我们，集群中存在镜像队列时，重新master节点有风险。

**镜像队列中节点启动顺序，非常有讲究：**

假设集群中包含两个节点，一般生产环境会部署三个节点，但为了方便说明，采用两个节点的形式进行说明。

**场景1：A先停，B后停**
该场景下B是master，只要先启动B，再启动A即可。或者先启动A，再在30s之内启动B即可恢复镜像队列。（**如果没有在30s内回复B，那么A自己就停掉自己**）

**场景2：A，B同时停**
该场景下可能是由掉电等原因造成，只需在30s内联系启动A和B即可恢复镜像队列。

**场景3：A先停，B后停，且A无法恢复。**
因为B是master，所以等B起来后，在B节点上调用rabbitmqctl forget_cluster_node A以接触A的cluster关系，再将新的slave节点加入B即可重新恢复镜像队列。

**场景4：A先停，B后停，且B无法恢复**
该场景比较难处理，旧版本的RabbitMQ没有有效的解决办法，在现在的版本中，因为B是master，所以直接启动A是不行的，当A无法启动时，也就没版本在A节点上调用rabbitmqctl forget_cluster_node B了，新版本中forget_cluster_node支持-offline参数，offline参数允许rabbitmqctl在离线节点上执行forget_cluster_node命令，迫使RabbitMQ在未启动的slave节点中选择一个作为master。当在A节点执行rabbitmqctl forget_cluster_node -offline B时，RabbitMQ会mock一个节点代表A，执行forget_cluster_node命令将B提出cluster，然后A就能正常启动了。最后将新的slave节点加入A即可重新恢复镜像队列

**场景5：A先停，B后停，且A和B均无法恢复，但是能得到A或B的磁盘文件**
这个场景更加难以处理。将A或B的数据库文件（$RabbitMQ_HOME/var/lib目录中）copy至新节点C的目录下，再将C的hostname改成A或者B的hostname。如果copy过来的是A节点磁盘文件，按场景4处理，如果拷贝过来的是B节点的磁盘文件，按场景3处理。最后将新的slave节点加入C即可重新恢复镜像队列。

场景6：A先停，B后停，且A和B均无法恢复，且无法得到A和B的磁盘文件
无解。



启动顺序中有一个30s 的概念，这个是MQ 的时间间隔，用于检测master、slave是否可用，因此30s 非常关键。

对于生产环境MQ集群的重启操作，需要分析具体的操作顺序，不可无序的重启，会有可能带来无法弥补的伤害(数据丢失、节点无法启动)。

简单总结下：**镜像队列是用于节点之间同步消息的机制，避免某个节点宕机而导致的服务不可用或消息丢失，且针对排他性队列设置是无效的。另外很重要的一点，镜像队列机制不是负载均衡。**