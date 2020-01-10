---
layout:     post
title:      Google Spanner论文解读
subtitle:   
date:       2020-01-10
author:     BY
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - spanner
 
---

​      

​    Spanner是Google搞的一个scalable, multi-version, globally distributed, synchronously-replicated database。它的最大特点就是数据分布在全球范围内，支持外部一致性的分布式事务。在2012年Spanner论文出来的时候，应该还没有一个系统做到这一点。

## **1** **概述**

对于外部来说，Spanner将数据分散到众多Paxos状态机，Paxos成员组甚至可能是分布在全球范围的。Spanner还自动处理failover、数据分片的分裂、迁移等问题。

Spanner关注的核心是跨数据中心的数据副本，先是实现了分布式系统的基础架构，然后在此之上实现了数据库特性。这些数据库特性相比Bigtable/Megastore更加贴近于传统数据库的概念，而做这些特性的出发点是Bigtable/Megastore的功能支持对应用来说，仍然不够。例如，强一致性，schema演化变更。Spanner在Bigtable类似的KV存储之上，将数据版本化以支持MVCC，对外提供类SQL的查询接口。

### **Spanner**的两个特色

第一个特色，副本配置可由应用细粒度地动态控制。

这一点是非常自然的，Spanner底层是一个跨数据中心的分布式系统，Paxos写多数派即同步成功，这两者加一块，自然就会有副本地理分布的考量。不同地域之间的网络延迟是一个重要的考量因素，如何部署才能使得多数派之间的网络延迟可以接受，用户所在地理位置和服务该用户的副本如何减少物理距离（也即如何在用户这个粒度上支持动态控制副本），都是需要考虑的问题。

第二个特色，Spanner支持外部一致性的读写和全局一致性的快照读。这个feature是依赖全局的提交版本号（即提交时间戳，以下不做区分）实现的。它保证了外部一致性external consistency，即linearizability：T1 commit早于T2 start，那么，T1的提交时间戳就应该比T2的提交时间戳小。这个feature对于外部应用客户端来说，是非常有意义的。如果一个应用有如下的逻辑：

 ```
exec T1(x=1);
if T1 success
    exec T2(y=1);
 ```

如果外部一致性保证不了，那么并发的另一个客户端读x和y时，可能看到y生效但是x没有生效。

## **2** **实现**

### **Spanner**的部署方式

Spanner的部署方式，或者说server组织方式如下图。

[![spanner_deplpoy](http://loopjump.com/wp-content/uploads/2016/09/spanner_deplpoy.png)](http://loopjump.com/wp-content/uploads/2016/09/spanner_deplpoy.png)

一个zone可以理解为一个数据中心，或者直接理解为机房。Zone内部有若干spanserver。Zonemaster负责将数据分配到spanserver，location proxy用于支持客户端查询数据分配情况。一个Spanner包含多个zone、一个universemaster、一个placement driver，总称universe。Universemaster是一个控制台服务器，用于监控和debug。Placement driver用于universe级别的跨zone数据迁移。因为Spanner本身已经是全球规模的部署，每个Spanner已经包含了数量巨大的spanserver，所以基本上也不需要几个universe。从这个部署方式看，Google的存储系统仍然沿用了Master-Slave的大模式，当然，此处master负载并不重。

**Spanserver**软件栈

Spanserver软件栈如图。它描述了从日志持久化到2PC的lock管理部件，内容涉及还比较多。

Tablet的概念与Bigtable差不多，但是key同时绑定一个时间戳，这个时间戳就是MVCC的版本号。Tablet的数据都在B-tree-like的文件 + write ahead log中间，存放于Colossus。Colossus是GFS II代，针对小文件做了优化。

Spanserver的每个tablet上运行一个Paxos状态机。状态机的元数据和Paxos日志都存到tablet中。当前实现下，一次写操作要写盘两次：一次是tablet的write ahead log，一次是Paxos log。论文说写两次，而且Paxos日志写到tablet中，这个还不太清楚是如何做的。Paxos日志要同步到其他副本上，而tablet本身的数据或者日志都是server local的，这个需要做区分。按照replicated state machine这个概念，持久化的数据只要一份就好。状态通过日志做变更或者重启恢复。我们这里也不纠缠这个问题，论文说这个设计是非预期的，正在改为只写一次。

另外，Basic Paxos的性能因为2阶段的RPC导致性能问题，因此，实用的Paxos实现，如Multi Paxos或者Raft，其性能都依赖有一个相对稳定的leader replica。这一点并不难做到，毕竟除了宕机等事件，通常并不需要主动切换leader。这里我们假定相对稳定的leader replica已经做到了。

[![spanserver_software_stack](http://loopjump.com/wp-content/uploads/2016/09/spanserver_software_stack.png)](http://loopjump.com/wp-content/uploads/2016/09/spanserver_software_stack.png)

Leader replica在replica之上，维护了lock table。这个lock是指数据库行锁，lock table主要管理2PC阶段的锁。

说到2PC，这里简单描述一下。2PC的目的是为了支持分布式事务的原子提交，避免只提交事务的一部分。协议本身理解起来非常简单，在若干参与者者中，选择一个作为协调者（理论上可以选择一个跟本事务无关的参与者），所有参与者先执行prepare，prepare成功的参与者就确保可以提交或者回滚，但暂时既不提交也不回滚，协调者收集所有的参与者的prepare结果，如果都成功，就通知所有的参与者提交，否则就通知所有参与者回滚。协议本身很简单，但是具体实现还有不少坑。

Leader上为了支持2PC事务，引入一个transaction manager。按照两阶段和主备，一个事务的副本在事务层的角色分为participant leader、participant slave、coordinator leader、coordinator slave。单tablet上（本质上说，同一个Paxos）上的事务都可以从两阶段优化成一个阶段，因为单个tablet持久化是原子地成功或者失败的，不需要引入2PC来协调事务提交的原子性。

软件栈描述过程中，我们着眼于tablet。下面我们关注下数据模型更细节的层次。

### **Data Model**

Spanner底层存储的是有序的KeyValue集合，在数据模型上，细化了directory这个概念。一个directory是连续的一组KeyValue，这些KeyValue的前缀是一致的。这里没有具体约束前缀是什么，后面可以看到directory的具体应用，举个例子，一个directory存储的都是前缀为user_id=100的相册的数据之类的。之所以这样设计，是为了使directory成为数据放置的单位，只有这样，才能达到前面所述应用能够在user_id粒度上控制数据的placement。Directory可以在Paxos组之间移动，而Directory也是跨Paxos组迁移的最小单位。

Directory和tablet的关系：在Bigtable中，一个tablet覆盖了MinKey => MaxKey的所有row space，在Spanner中，directory是可以类比为bigtable的tablet更细粒度的版本、而Spanner的tablet则单纯地是directory的集合，并不覆盖一段row space所有key-value记录。

这边给的迁移速度是一个direcotry在几秒内迁完。可以猜测的是这个速度限制主要是迁移源端的读取速度。考虑到Spanner目前每个Paxos write都要写两次，key-value部分的数据都是连续存放的，读起来应该不会太慢，而Paxos日志的读取估计会比较费时。也可能是迁移有限流。

因为迁移的耗时比较长，所以迁移本身不适合做成一个事务，因为对普通事务的阻塞太长。迁移时，先在后台迁移数据，等只剩最后一点数据时，原子地迁移最后一点数据和更新Paxos组的Meta信息。

文中还提到，如果一个directory太大，可能会拆分成sub-directory，即fragment。Fragment除了数据时directory的一部分之外，其在Spanner中的地位都是跟directory等价的。

Spanner对应用呈现的是：模式化的半关系型表的数据模型、类SQL的查询语言、通用事务支持。

前面提到的directory的粒度问题，引入了另一个使用上的模式或者叫范式。每个数据库，都需要应用将各个表设计成层级（hierarchy）的样子。各张表的数据是交错（interleave）和级联的（cascade）。这个举个例子就很清楚了。

```
CREATE TABLE Users {
    uid INT64 NOT NULL, email STRING
} PRIMARY KEY (uid), DIRECTORY;
CREATE TABLE Albums {
    uid INT64 NOT NULL, aid INT64 NOT NULL,
    name STRING
} PRIMARY KEY (uid, aid), INTERLEAVE IN PARENT Users ON DELETE CASCADE;
```

[![interleave](http://loopjump.com/wp-content/uploads/2016/09/interleave.png)](http://loopjump.com/wp-content/uploads/2016/09/interleave.png)

这个例子是说，创建两张表，一张是user表，一张是albums表，一个用户有多个相册。那么，就需要将这两张表设计成层级的。即user表是父表，albums表是子表。Key-value数据存放时，子表的数据交错放入父表对应的user_id之后。每个user_id对应的一key-value就是一个directory。

这种数据模型对于应用感知locality很有必要。

 

## **3 True Time**

True Time是Spanner最亮眼的地方。

先看看True Time提供了哪些功能/API：

[![true_time](http://loopjump.com/wp-content/uploads/2016/09/true_time.png)](http://loopjump.com/wp-content/uploads/2016/09/true_time.png)

接口语义很清晰。TT.now()返回的是一个时间戳区间，TrueTime保证调用now函数时，绝对时间一定在这个时间戳区间内。TT.after和TT.before都是对TT.now的封装。

我认为，本质上，TrueTime是个全序时钟，只不过这个时钟不是全局时间点而是全局时间段，这一点也是Spanner实现外部一致性的根本支持。

实现：实现细节论文也没有太多详述，只是描述了大致的原理。每个数据中心都有一个timer master，用以校正全局时钟。每台机器都有一个timeslave daemon来找master poll时间。Master之间为了降低硬件失效，都配了两套时钟硬件：GPS和原子钟。两套硬件失效的因素不同，因此同时失效的概率会比较小。Slave会poll问时间，然后有一套算法来交叉验证。

 

## **4 Concurrency Control**

这节是讲述TrueTime如何应用到并发控制的。

Spanner支持的事务类型：读写事务read-write transactions、只读事务（客户端显示指明是只读）read-only transactions、快照读snapshot isolation。

只读事务不阻塞写，而且只读事务可以在备机上读（前提备机已经有足够新的数据，怎么叫足够新，后面会解释）。

快照读事务是读过去一个时间点的快照，这个时间点可以是用户指定，也可以是指定一个时间戳上界，Spanner选一个合适的时间戳作为快照点。快照读事务同样不阻塞写。

对于只读事务和快照读事务来说，只要选定了时间点，事务总是可以提交的，除非选的快照点太旧，对应的数据被回收了。只读事务对数据库自身的数据状态没有影响，因此即使在某台机器上读了一半的数据时，机器宕机，仍然可以换机器用一样的时间戳重试，结果不会有变化。

### **外部一致性保证**

读写事务使用两阶段锁（two phase locking, 2PL），因此给读写事务指定时间戳的时机可以是在全部锁已申请到锁释放前的任何时刻。Spanner给事务指定的时间戳是Paxos write用的时间戳，这个时间戳是代表了事务提交的时间。

那么，Paxos write用的时间戳是如何指定的呢？这个时间戳就是由leader replica简单地按照递增的顺序指定的。当然，这之外还需要一个约束：在切换leader replica时，仍然保证跨主备切换下的时间戳也是递增的。这个约束论文称之为单调不变性monotonicity invariant。

这个约束如何实现？首先，Spanner的paxos leader lease已经保证leader的lease没有交集，在此条件下，在发生主备切切换时，主想卸任要先等一下。思路很简单，详细的分析在附录A上有，不再赘述。

Spanner还保证了外部一致性约束（external consistency invariant）：如果T2开始前T1已经提交，则T1的提交时间戳早于T2的提交时间戳，即 

### **外部一致性保证**

读写事务使用两阶段锁（two phase locking, 2PL），因此给读写事务指定时间戳的时机可以是在全部锁已申请到锁释放前的任何时刻。Spanner给事务指定的时间戳是Paxos write用的时间戳，这个时间戳是代表了事务提交的时间。

那么，Paxos write用的时间戳是如何指定的呢？这个时间戳就是由leader replica简单地按照递增的顺序指定的。当然，这之外还需要一个约束：在切换leader replica时，仍然保证跨主备切换下的时间戳也是递增的。这个约束论文称之为单调不变性monotonicity invariant。

这个约束如何实现？首先，Spanner的paxos leader lease已经保证leader的lease没有交集，在此条件下，在发生主备切切换时，主想卸任要先等一下。思路很简单，详细的分析在附录A上有，不再赘述。

Spanner还保证了外部一致性约束（external consistency invariant）：如果T2开始前T1已经提交，则T1的提交时间戳早于T2的提交时间戳，即
$$
t_{abs}(e^{commit}_1)<t_{abs}(e^{start}_2)⇒s1<s2
$$



其中，$$t_{abs}(e)$$指事件e的绝对时间戳，$$e^{commit}_1$$是T1的提交事件，$$e^{start}_2$$是T2的开始事件，$$s_1$$和$$s_2$$分别是Spanner给T1、T2指定的提交时间戳。

那么，这个约束又是如何保证的呢？只要保证两条规则：Start规则和Commit Wait规则即可。这里定义$$e^{server}_i$$为事务$$T_i$$的coordinator leader收到提交请求的事件。

**Start规则**：coordinator leader给事务$$T_i$$指定的时间戳$$s_i$$满足 $$s_i$$≥$$TT.now().latestsi≥TT.now().latest $$ 条件，其中TT.now在$$e^{server}_i$$之后调用。

**Commit Wait规则**：$$s_i≤t_{abs}(e^{commit}_i)s_i≤t_{abs}(e^{commit}_i)$$    这个规则是说，提交时间戳要确保比提交事件的绝对时间小，也即$$TT.after(s_i)$$为真。

 这两个规则就保证了外部一致性：

$$s_1<t_{abs}(e_1^{commit})$$   (commit wait)  

$$t_{abs}(e^{commit}_1)<t_{abs}(e_2^{start})$$      (assumption) 

$$t_{abs}(e_2^{start})≤t_abs(e_2^{server})$$   (causality) 

$$t_{abs}(e^{server}_2)≤s_2 $$                 (start)  

$$s_1<s_2 $$                             (transitivity)

### **Spanner**是如何做事务的？

为了搞明白这个并发控制和外部一致性，以及后面读请求的安全快照点，需要先大致了解下Spanner是如何做事务的。

client读请求直接发到对应的replica上，写的数据buffer到client自己这里，等执行结束，要提交时，将buffer的这些数据推到对应的参与者，然后选一个参与者作为协调者执行2PC，各个参与者先申请写锁，然后分配一个prepare的时间戳，然后写日志，所有参与者回应协调者，协调者选择一个时间戳s作为提交时间戳，写commit日志（协调者不写prepare日志），然后按照commit wait规则等待TT.now(s)条件满足后，将commit应用到Paxos状态机，然后通知其他参与者进入2PC Commit阶段，其他参与者明白事务确定要提交，因此写outcome日志，这个日志包含了时间戳s的值，同步该日志成功后，应用到状态机，释放锁。

协调者选时间戳s要满足如下条件：1) s大于等于所有其他参与者的prepare时间戳；2)大于协调者收到client commit请求时$$TT.now().latest$$；3) 大于已经分配的时间戳。

### **安全读取时间戳 tsafetsafe**

我们来看下之前说的读请求在一个副本上可以读到多新的数据？每个副本无论主备，都维护一个$$t_{safe}$$的时间戳，读请求可以读到这个时间戳时间的数据。$$t_{safe}$$计算方式为：

​                                                          $$t_{safe}$$=$$min(t^{Paxos}_{safe},t^{TM}_{safe})$$

$$t^{Paxos}_{safe}$$指已经应用到Paxos状态机的Paxos write的时间戳的安全时间戳，因为Paxos write应用到状态机是顺序应用的，因此只要维护最后一次应用的Paxos write的时间戳就可以了。

$$t^{TM}_{safe}$$是指所有没有提交的事务的提交时间戳下界。那么，没提交的事务的提交时间戳下界怎么找呢？

如果当前活跃的事务都是还未进入2PC的事务，那么，$$t^{TM}_{safe}$$就是 $$∞$$，对$$t^{TM}_{safe}$$没有影响。这是因为读写事务的写数据会先Buffer到client，不到提交时候，不会被其他事务读到。

如果某个事务已经进入2PC的prepare阶段，该参与者上事务最后是不是提交现在还确定不了，那么就需要确保$$t^{TM}_{safe}$$不能超过这个事务将来的提交时间戳。如何保证这一点？TM先给事务的prepare事件指定一个时间戳，并且要求将来提交的时间戳比prepare的时间戳更大，那么prepare事件的时间戳肯定就是这个事务提交事件的时间戳下界。也就是论文中说的，$$t^{TM}_{safe}$$$$=min(s^{prepare}_{i,g})−1$$，其中$$s^{prepare}_{i,g}$$是组$$g$$内所有的participant leader上已prepare未commit的事务TiTi的prepare事件的时间戳。

$$t^{TM}_{safe}$$主备上在具体实现上略有不同，备上没有TM，其信息通过Paxos同步携带，原理一致。

这里需要解释的地方：为什么要取$$t^{Paxos}_{safe}$$, $$t^{TM}_{safe}$$二者的最小值？

因为2PC协议本身的局限，参与者prepare结束后，需要死等协调者的commit or abort结论，这个阶段理论上是可以足够长时间的。这段时间内该参与者replica 上可能一直有无关的事务（修改的是该tablet上的其他row key）执行并提交，足够多的Paxos write已经被应用了，$$t^{Paxos}_{safe}$$可能已经足够大。所以要取二者的最小值。从这个结论也能看出，如果有一个事务卡在2PC过程中，迟迟未决，那么读请求总也读不到比较新的快照点。那么，这里有一个refinement：为了避免一个卡顿的事务影响了该tablet上所有的读请求，即将$$t^{TM}_{safe}$$从tablet粒度细化到row key range粒度，而lock table正适合做这个refinement。