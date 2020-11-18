## 详述MySQL事务及ACID特性的实现原理

事务是 MySQL 等关系型数据库区别于 NoSQL 的重要方面，是保证数据一致性的重要手段。



本文将首先介绍 MySQL 事务相关的基础概念，然后介绍事务的 ACID 特性，并分析其实现原理。MySQL 博大精深，文章疏漏之处在所难免，欢迎批评指正。



MySQL 事务基础概念



事务（Transaction）是访问和更新数据库的程序执行单元；事务中可能包含一个或多个 sql 语句，这些语句要么都执行，要么都不执行。



作为一个关系型数据库，MySQL 支持事务，本文介绍基于 MySQL 5.6。首先回顾一下 MySQL 事务的基础知识。





**逻辑架构和存储引擎**



![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67q9rDVwC3Z7sWibwUXwZVo5fpGEqkN62LvfmO2dZM7icXqCsfEt1NDzMQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，MySQL 服务器逻辑架构从上往下可以分为三层：

- **第一层：**处理客户端连接、授权认证等。

- **第二层：**服务器层，负责查询语句的解析、优化、缓存以及内置函数的实现、存储过程等。

- **第三层：**存储引擎，负责 MySQL 中数据的存储和提取。MySQL 中服务器层不管理事务，事务是由存储引擎实现的。

  MySQL 支持事务的存储引擎有 InnoDB、NDB Cluster 等，其中 InnoDB 的使用最为广泛；其他存储引擎不支持事务，如 MyIsam、Memory 等。



如无特殊说明，后文中描述的内容都是基于 InnoDB。





**提交和回滚**



典型的 MySQL 事务是如下操作的：

```
start transaction;
……  #一条或多条sql语句
commit;
```



其中 start transaction 标识事务开始，commit 提交事务，将执行结果写入到数据库。



如果 sql 语句执行出现问题，会调用 rollback，回滚所有已经执行成功的 sql 语句。当然，也可以在事务中直接使用 rollback 语句进行回滚。



**自动提交**



MySQL 中默认采用的是自动提交（autocommit）模式，如下所示：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67oXYyBR2UVq1MicL1GiclMu3C7xcyoM0No27r5yYN85rBP2DJS4ZMGlqQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在自动提交模式下，如果没有 start transaction 显式地开始一个事务，那么每个 sql 语句都会被当做一个事务执行提交操作。



通过如下方式，可以关闭 autocommit；需要注意的是，autocommit 参数是针对连接的，在一个连接中修改了参数，不会对其他连接产生影响。

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67Y2cHgydB1tICjIfwJlgMDXuUCu5ibL3Ur7t0jVsjPdM4cKHEgIfGibYQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果关闭了 autocommit，则所有的 sql 语句都在一个事务中，直到执行了 commit 或 rollback，该事务结束，同时开始了另外一个事务。



**特殊操作**



在 MySQL 中，存在一些特殊的命令，如果在事务中执行了这些命令，会马上强制执行 commit 提交事务；如 DDL 语句(create table/drop table/alter/table)、lock tables 语句等等。



不过，常用的 select、insert、update 和 delete 命令，都不会强制提交事务。





**ACID 特性**



ACID 是衡量事务的四个特性：

- **原子性（Atomicity，或称不可分割性）**
- **一致性（Consistency）**
- **隔离性（Isolation）**
- **持久性（Durability）**



按照严格的标准，只有同时满足 ACID 特性才是事务；但是在各大数据库厂商的实现中，真正满足 ACID 的事务少之又少。



例如 MySQL 的 NDB Cluster 事务不满足持久性和隔离性；InnoDB 默认事务隔离级别是可重复读，不满足隔离性；Oracle 默认的事务隔离级别为 READ COMMITTED，不满足隔离性……



因此与其说 ACID 是事务必须满足的条件，不如说它们是衡量事务的四个维度。



下面将详细介绍 ACID 特性及其实现原理，为了便于理解，介绍的顺序不是严格按照 A-C-I-D。



ACID 特性及其实现原理





**原子性**



**定义**



原子性是指一个事务是一个不可分割的工作单位，其中的操作要么都做，要么都不做。



如果事务中一个 sql 语句执行失败，则已执行的语句也必须回滚，数据库退回到事务前的状态。



## **实现原理：undo log**



在说明原子性原理之前，首先介绍一下 MySQL 的事务日志。MySQL 的日志有很多种，如二进制日志、错误日志、查询日志、慢查询日志等。



此外 InnoDB 存储引擎还提供了两种事务日志：

- **redo log(重做日志)**
- **undo log(回滚日志)**



其中 redo log 用于保证事务持久性；undo log 则是事务原子性和隔离性实现的基础。



下面说回 undo log。实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的 sql 语句。



InnoDB 实现回滚，靠的是 undo log：

- 当事务对数据库进行修改时，InnoDB 会生成对应的 undo log。
- 如果事务执行失败或调用了 rollback，导致事务需要回滚，便可以利用 undo log 中的信息将数据回滚到修改之前的样子。



undo log 属于逻辑日志，它记录的是 sql 执行相关的信息。当发生回滚时，InnoDB 会根据 undo log 的内容做与之前相反的工作：

- 对于每个 insert，回滚时会执行 delete。
- 对于每个 delete，回滚时会执行 insert。
- 对于每个 update，回滚时会执行一个相反的 update，把数据改回去。



**以 update 操作为例：**当事务执行 update 时，其生成的 undo log 中会包含被修改行的主键(以便知道修改了哪些行)、修改了哪些列、这些列在修改前后的值等信息，回滚时便可以使用这些信息将数据还原到 update 之前的状态。





**持久性**



## **定义**



持久性是指事务一旦提交，它对数据库的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。



## **实现原理：redo log**



redo log 和 undo log 都属于 InnoDB 的事务日志。下面先聊一下 redo log 存在的背景。



InnoDB 作为 MySQL 的存储引擎，数据是存放在磁盘中的，但如果每次读写数据都需要磁盘 IO，效率会很低。



为此，InnoDB 提供了缓存(Buffer Pool)，Buffer Pool 中包含了磁盘中部分数据页的映射，作为访问数据库的缓冲：

- 当从数据库读取数据时，会首先从 Buffer Pool 中读取，如果 Buffer Pool 中没有，则从磁盘读取后放入 Buffer Pool。
- 当向数据库写入数据时，会首先写入 Buffer Pool，Buffer Pool 中修改的数据会定期刷新到磁盘中（这一过程称为刷脏）。



**Buffer Pool 的使用大大提高了读写数据的效率，但是也带来了新的问题：**如果 MySQL 宕机，而此时 Buffer Pool 中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证。



**于是，redo log 被引入来解决这个问题：**当数据修改时，除了修改 Buffer Pool 中的数据，还会在 redo log 记录这次操作；当事务提交时，会调用 fsync 接口对 redo log 进行刷盘。



如果 MySQL 宕机，重启时可以读取 redo log 中的数据，对数据库进行恢复。



redo log 采用的是 WAL（Write-ahead logging，预写式日志），所有修改先写入日志，再更新到 Buffer Pool，保证了数据不会因 MySQL 宕机而丢失，从而满足了持久性要求。



既然 redo log 也需要在事务提交时将日志写入磁盘，为什么它比直接将 Buffer Pool 中修改的数据写入磁盘(即刷脏)要快呢？



主要有以下两方面的原因：

- **刷脏是随机 IO，**因为每次修改的数据位置随机，但写 redo log 是追加操作，属于顺序 IO。
- **刷脏是以数据页（Page）为单位的，**MySQL 默认页大小是 16KB，一个 Page 上一个小修改都要整页写入；而 redo log 中只包含真正需要写入的部分，无效 IO 大大减少。



## **redo log 与 binlog**



我们知道，在 MySQL 中还存在 binlog(二进制日志)也可以记录写操作并用于数据的恢复，但二者是有着根本的不同的。



作用不同：

- redo log 是用于 crash recovery 的，保证 MySQL 宕机也不会影响持久性；
- binlog 是用于 point-in-time recovery 的，保证服务器可以基于时间点恢复数据，此外 binlog 还用于主从复制。



层次不同：

- redo log 是 InnoDB 存储引擎实现的，
- 而 binlog 是 MySQL 的服务器层(可以参考文章前面对 MySQL 逻辑架构的介绍)实现的，同时支持 InnoDB 和其他存储引擎。



内容不同：

- redo log 是物理日志，内容基于磁盘的 Page。
- binlog 是逻辑日志，内容是一条条 sql。



写入时机不同：

- **redo log 的写入时机相对多元。**前面曾提到，当事务提交时会调用 fsync 对 redo log 进行刷盘；这是默认情况下的策略，修改 innodb_flush_log_at_trx_commit 参数可以改变该策略，但事务的持久性将无法保证。

  除了事务提交时，还有其他刷盘时机：如 master thread 每秒刷盘一次 redo log 等，这样的好处是不一定要等到 commit 时刷盘，commit 速度大大加快。

- **binlog 在事务提交时写入。**

#  



**隔离性**

##  

## **定义**



与原子性、持久性侧重于研究事务本身不同，隔离性研究的是不同事务之间的相互影响。



隔离性是指事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能互相干扰。



严格的隔离性，对应了事务隔离级别中的 Serializable(可串行化)，但实际应用中出于性能方面的考虑很少会使用可串行化。



隔离性追求的是并发情形下事务之间互不干扰。简单起见，我们仅考虑最简单的读操作和写操作(暂时不考虑带锁读等特殊操作)。



那么隔离性的探讨，主要可以分为两个方面：

- **(一个事务)写操作对(另一个事务)写操作的影响：**锁机制保证隔离性。
- **(一个事务)写操作对(另一个事务)读操作的影响：**MVCC 保证隔离性。

##  

## **锁机制**



首先来看两个事务的写操作之间的相互影响。隔离性要求同一时刻只能有一个事务对数据进行写操作，InnoDB 通过锁机制来保证这一点。



锁机制的基本原理可以概括为：

- 事务在修改数据之前，需要先获得相应的锁。
- 获得锁之后，事务便可以修改数据。
- 该事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待当前事务提交或回滚后释放锁。



**行锁与表锁：**按照粒度，锁可以分为表锁、行锁以及其他位于二者之间的锁。



表锁在操作数据时会锁定整张表，并发性能较差；行锁则只锁定需要操作的数据，并发性能好。



但是由于加锁本身需要消耗资源(获得锁、检查锁、释放锁等都需要消耗资源)，因此在锁定数据较多情况下使用表锁可以节省大量资源。



MySQL 中不同的存储引擎支持的锁是不一样的，例如 MyIsam 只支持表锁，而 InnoDB 同时支持表锁和行锁，且出于性能考虑，绝大多数情况下使用的都是行锁。



如何查看锁信息？有多种方法可以查看 InnoDB 中锁的情况，例如：

```
select * from information_schema.innodb_locks; #锁的概况
show engine innodb status; #InnoDB整体状态，其中包括锁的情况
```



下面来看一个例子：

```
#在事务A中执行：
start transaction;
update account SET balance = 1000 where id = 1;
在事务B中执行：
start transaction;
update account SET balance = 2000 where id = 1;
```



此时查看锁的情况：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67PROcKHFTEksq3ibJTku4GyoNwWXGYroNia0rObNicqKl7F00SeBibbffjg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

show engine innodb status 查看锁相关的部分：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67okJAXv4Q6jOTQXiawRLHcReicAxw3KOD8SGoRSdicM7icMOSCP5eL8ATiag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过上述命令可以查看事务 24052 和 24053 占用锁的情况；其中 lock_type 为 RECORD，代表锁为行锁(记录锁)；lock_mode 为 X，代表排它锁(写锁)。



除了排它锁(写锁)之外，MySQL 中还有共享锁(读锁)的概念。由于本文重点是 MySQL 事务的实现原理，因此对锁的介绍到此为止。



介绍完写操作之间的相互影响，下面讨论写操作对读操作的影响。



## **脏读、不可重复读和幻读**



首先来看并发情况下，读操作可能存在的三类问题。



**①脏读：**当前事务(A)中可以读到其他事务(B)未提交的数据（脏数据），这种现象是脏读。



举例如下（以账户余额表为例）：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67AOUhzpGNEeHXI3BrhMLarxqPTblTcMc6TPrIP1KiciavLkSNDsy1o27A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**②不可重复读：**在事务 A 中先后两次读取同一个数据，两次读取的结果不一样，这种现象称为不可重复读。



脏读与不可重复读的区别在于：前者读到的是其他事务未提交的数据，后者读到的是其他事务已提交的数据。



举例如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67fE8v1z9cFmFmhzUOxMsPXrpoSZFMIwYtB1gxJKIYia9UCj5kSgAjibrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**③幻读：**在事务 A 中按照某个条件先后两次查询数据库，两次查询结果的条数不同，这种现象称为幻读。



不可重复读与幻读的区别可以通俗的理解为：前者是数据变了，后者是数据的行数变了。



举例如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67vPFQzMib1gSRytXh2TahF3HquJNmufTzKAg3nGh4sAlNeVXiaKZa5UQQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **事务隔离级别**



sql 标准中定义了四种隔离级别，并规定了每种隔离级别下上述几个问题是否存在。



一般来说，隔离级别越低，系统开销越低，可支持的并发越高，但隔离性也越差。



隔离级别与读问题的关系如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67sHlnrszRPdtEaibfDv5rqpEtvArkxyNloGnMPzRWicC0iaRVv0ibmMxjXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在实际应用中，读未提交在并发时会导致很多问题，而性能相对于其他隔离级别提高却很有限，因此使用较少。



可串行化强制事务串行，并发效率很低，只有当对数据一致性要求极高且可以接受没有并发时使用，因此使用也较少。



因此在大多数数据库系统中，默认的隔离级别是读已提交(如 Oracle)或可重复读（后文简称 RR）。



可以通过如下两个命令分别查看全局隔离级别和本次会话的隔离级别：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67horhPicv30ayOmIf5PAAvgBs1IGenItkWScfeoXk6aWZUqRQGdntvOA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67OEotTwtNJXFj55tibbCia0XVTpiasx4cicajsyWP83fAMjwT10fwVFCp7Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

InnoDB 默认的隔离级别是 RR，后文会重点介绍 RR。需要注意的是，在 SQL 标准中，RR 是无法避免幻读问题的，但是 InnoDB 实现的 RR 避免了幻读问题。



## **MVCC**



RR 解决脏读、不可重复读、幻读等问题，使用的是 MVCC：MVCC 全称 Multi-Version Concurrency Control，即多版本的并发控制协议。



**下面的例子很好的体现了 MVCC 的特点：**在同一时刻，不同的事务读取到的数据可能是不同的(即多版本)——在 T5 时刻，事务 A 和事务 C 可以读取到不同版本的数据。

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67iaNcQOCtFnpAmA8GQlI21DxIZUyqLlEYD2HnypgPCnqtjLNKMAk5r5w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

MVCC 最大的优点是读不加锁，因此读写不冲突，并发性能好。InnoDB 实现 MVCC，多个版本的数据可以共存，主要是依靠数据的隐藏列(也可以称之为标记位)和 undo log。



其中数据的隐藏列包括了该行数据的版本号、删除时间、指向 undo log 的指针等等。



当读取数据时，MySQL 可以通过隐藏列判断是否需要回滚并找到回滚需要的 undo log，从而实现 MVCC；隐藏列的详细格式不再展开。



下面结合前文提到的几个问题分别说明。



**①脏读**

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67gPNvC3c5aZxtlAcXxEh6APjR6ibm7AXgtSic6PH8InXViaSCYRjzpcgHA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当事务 A 在 T3 时间节点读取 zhangsan 的余额时，会发现数据已被其他事务修改，且状态为未提交。



此时事务 A 读取最新数据后，根据数据的 undo log 执行回滚操作，得到事务 B 修改前的数据，从而避免了脏读。



**②不可重复读**

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67Xrknz2JU9dd55K7WtRBKfibUV4A0QPeWnqCJpd1ep3D9qjoMOd4oJGg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当事务 A 在 T2 节点第一次读取数据时，会记录该数据的版本号（数据的版本号是以 row 为单位记录的），假设版本号为 1；当事务 B 提交时，该行记录的版本号增加，假设版本号为 2。



当事务 A 在 T5 再一次读取数据时，发现数据的版本号（2）大于第一次读取时记录的版本号（1），因此会根据 undo log 执行回滚操作，得到版本号为 1 时的数据，从而实现了可重复读。



**③幻读**



InnoDB 实现的 RR 通过 next-keylock 机制避免了幻读现象。



next-keylock 是行锁的一种，实现相当于 record lock(记录锁) + gap lock(间隙锁)；其特点是不仅会锁住记录本身(record lock 的功能)，还会锁定一个范围(gap lock 的功能)。



当然，这里我们讨论的是不加锁读：此时的 next-key lock 并不是真的加锁，只是为读取的数据增加了标记（标记内容包括数据的版本号等）；准确起见姑且称之为类 next-key lock 机制。



还是以前面的例子来说明：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67dI5aolicV1Wcoubq4m4112Uxia0uEdKSTqZcPCbpfUbkA8eg3cu6QfZg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当事务 A 在 T2 节点第一次读取 0<id<5 数据时，标记的不只是 id=1 的数据，而是将范围(0,5)进行了标记。



这样当 T5 时刻再次读取 0<id<5 数据时，便可以发现 id=4 的数据比之前标记的版本号更高，此时再结合 undo log 执行回滚操作，避免了幻读。



## **小结：**概括来说，InnoDB 实现的 RR，通过锁机制、数据的隐藏列、undo log 和类 next-key lock，实现了一定程度的隔离性，可以满足大多数场景的需要。



不过需要说明的是，RR 虽然避免了幻读问题，但是毕竟不是 Serializable，不能保证完全的隔离。



下面是一个例子，大家可以自己验证一下：

![img](https://mmbiz.qpic.cn/mmbiz_png/MOwlO0INfQqeEKUDcGJGwYCzz6gFto67z267Y2CdDxicJwRWPOT7rvKRMfoEAwH6NviagCNGuib8jIveH5Sb1NS7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**一致性**



## **基本概念**



一致性是指事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。



数据库的完整性约束包括但不限于：

- **实体完整性（如行的主键存在且唯一）**
- **列完整性（如字段的类型、大小、长度要符合要求）**
- **外键约束**
- **用户自定义完整性（如转账前后，两个账户余额的和应该不变）**

##  

## **实现**



可以说，一致性是事务追求的最终目标：前面提到的原子性、持久性和隔离性，都是为了保证数据库状态的一致性。此外，除了数据库层面的保障，一致性的实现也需要应用层面进行保障。



实现一致性的措施包括：

- 保证原子性、持久性和隔离性，如果这些特性无法保证，事务的一致性也无法保证。
- 数据库本身提供保障，例如不允许向整形列插入字符串值、字符串长度不能超过列的限制等。
- 应用层面进行保障，例如如果转账操作只扣除转账者的余额，而没有增加接收者的余额，无论数据库实现的多么完美，也无法保证状态的一致。

#  

总结



下面总结一下 ACID 特性及其实现原理：

- **原子性：**语句要么全执行，要么全不执行，是事务最核心的特性。事务本身就是以原子性来定义的；实现主要基于 undo log。
- **持久性：**保证事务提交后不会因为宕机等原因导致数据丢失；实现主要基于 redo log。
- **隔离性：**保证事务执行尽可能不受其他事务影响；InnoDB 默认的隔离级别是 RR，RR 的实现主要基于锁机制、数据的隐藏列、undo log 和类 next-key lock 机制。
- **一致性：**事务追求的最终目标，一致性的实现既需要数据库层面的保障，也需要应用层面的保障。

#  

# *参考文献：*

- *《MySQL 技术内幕：InnoDB 存储引擎》*
- *《高性能 MySQL》*
- *《MySQL 运维内参》*
- *https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_acid*
- *https://dev.mysql.com/doc/refman/5.6/en/innodb-next-key-locking.html*
- *http://blog.sina.com.cn/s/blog_499740cb0100ugs7.html*
- *https://mp.weixin.qq.com/s/2dwGBTmu_da2x-HiHlN0vw*
- *http://www.cnblogs.com/chenpingzhao/p/5065316.html*
- *https://juejin.im/entry/5ba0a254e51d450e735e4a1f*
- *http://hedengcheng.com/?p=771*