# Redis面试题汇总

<!-- vscode-markdown-toc -->
* 1. [Redis是单线程还是多线程?](#Redis)
	* 1.1. [为什么Redis再4.0之前即使使用了单线程，但是仍然那么快？](#Redis4.0)
	* 1.2. [谈谈Redis中的多路复用机制](#Redis-1)
	* 1.3. [Redis线程模型](#Redis-1)
* 2. [Redis存在线程安全的问题吗?](#Redis-1)
* 3. [Redis如何应对缓存穿透？](#Redis-1)
	* 3.1. [Bloom过滤器](#Bloom)
* 4. [Redis如何应对缓存击穿？](#Redis-1)
* 5. [Redis如何应对缓存雪崩?](#Redis-1)
* 6. [Redis如何做缓存预热？](#Redis-1)
* 7. [Redis如何保持缓存一致性?](#Redis-1)
	* 7.1. [Cache Aside](#CacheAside)
	* 7.2. [Read Through](#ReadThrough)
	* 7.3. [Write Through](#WriteThrough)
	* 7.4. [Write Behind Caching](#WriteBehindCaching)
* 8. [Redis如何回收缓存(针对内存空间不足)?](#Redis-1)
	* 8.1. [如何选择合适的策略?](#)
* 9. [Redis过期键有哪些删除策略?](#Redis-1)
	* 9.1. [键的过期精度](#-1)
	* 9.2. [过期和持久](#-1)
	* 9.3. [过期key的删除策略](#key)
	* 9.4. [在复制AOF文件时出现过期key该怎么办?](#AOFkey)
* 10. [简述Redis持久化机制](#Redis-1)
	* 10.1. [RDB的优点](#RDB)
	* 10.2. [RDB的缺点](#RDB-1)
	* 10.3. [AOF的优点](#AOF)
	* 10.4. [AOF缺点](#AOF-1)
	* 10.5. [4.X版本的持久化策略整合](#X)
* 11. [简述Redis主从复制](#Redis-1)
	* 11.1. [主从复制的过程](#-1)
	* 11.2. [主从复制的关注点](#-1)
	* 11.3. [复制的流程](#-1)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

##  1. <a name='Redis'></a>Redis是单线程还是多线程?

* 无论什么redis版本，工作线程[`worker thread`]()只有一个
* [`6.x`]()版本出现了IO多线程
* 目前所说的Redis单线程，指的是"其网络IO和键值对读写是由一个线程完成的"，也就是说，**Redis中只有网络请求模块和数据操作模块是单线程的。而其他的如持久化存储模块、集群支撑模块等是多线程的。** Redis 4.0的时候就已经针对部分命令做了多线程化。主要是体现在大数据的异步删除功能上，例如`unlink key`、`flushdb async`、`flushall async`等。

###  1.1. <a name='Redis4.0'></a>为什么Redis再4.0之前即使使用了单线程，但是仍然那么快？

* 单线程，不存在锁竞争的状态，可以在无锁的情况下完成所有的操作，不存在死锁和线程切换带来的时间开销
* Redis大部分的操作都是在内存中完成的，内存的执行效率非常快，并且采用了高效的数据结构，比如`hashtable`和`skiplist`
* Redis采用IO多路复用机制处理大量客户端的`Socket`请求，**注意，多路复用器仅仅负责调度IO事件，不负责真正的读写操作** ，因为这是非阻塞的IO模型，可以让Redis进行高效的网络通信及IO读写

###  1.2. <a name='Redis-1'></a>谈谈Redis中的多路复用机制

**Linux多路复用技术，就是多个进程的IO可以注册到同一个管道上，这个管道会统一和内核进行交互。当管道中的某一个请求需要的数据准备好之后，进程再把对应的数据拷贝到用户空间中。**也就是说，**通过一个线程来处理多个IO流**。

IO多路复用在Linux下包括了三种，select、poll、epoll三种模式。

其实，Redis的IO多路复用程序的所有功能都是通过包装操作系统的IO多路复用函数库来实现的。每个IO多路复用函数库在Redis源码中都有对应的一个单独的文件。

###  1.3. <a name='Redis-1'></a>Redis线程模型

在Redis 中，文件事件处理器([`file event handler`]())包括套接字、I/O 多路复用程序、文件事件分派器、以及事件处理器。使用 I/O 多路复用程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。每当一个套接字准备好执行连接应答、写入、读取、关闭等操作时，就会产生一个文件事件。因为一个服务器通常会连接多个套接字，所以多个文件事件有可能会并发地出现。

I/O 多路复用程序负责监听多个套接字，并向文件事件分派器传送那些产生了事件的套接字。尽管多个文件事件可能会并发地出现，但 I/O 多路复用程序总是会将所有产生事件的套接字都入队到一个队列里面，然后通过这个队列，以有序、同步、每次一个套接字的方式向文件事件分派器传送套接字：当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕），I/O 多路复用程序才会继续向文件事件分派器传送下一个套接字。如果一个套接字又可读又可写的话，那么服务器将先读套接字，后写套接字。

总的来说，就是**请求Socker—>IO多路复用程序—>file event dispatcher队列—>file event handler文件事件处理器**

**客户端被读取的顺序不能被保障，但是一个Socket中的操作顺序可以被保障**

##  2. <a name='Redis-1'></a>Redis存在线程安全的问题吗?

Redis虽然可以保障内部串行，但是外部使用Redis的时候需要额外的操作来保障线程安全。**因为Redis不能保障每一个Socket请求被桉顺序读取**

##  3. <a name='Redis-1'></a>Redis如何应对缓存穿透？

- 在接口访问层对用户做校验，如接口传参、登陆状态、n秒内访问接口的次数；
- 利用布隆过滤器，将数据库层有的数据key存储在位数组中，以判断访问的key在底层数据库中是否存在；

###  3.1. <a name='Bloom'></a>Bloom过滤器

- 如果Redis内不存在该数据，则通过布隆过滤器判断数据是否在底层数据库内；
- 如果布隆过滤器告诉我们该key在底层库内不存在，则直接返回null给客户端即可，避免了查询底层数据库的动作；
- 如果布隆过滤器告诉我们该key**极有可能**在底层数据库内存在，那么将查询下推到底层数据库即可；

**Bloom过滤器也无法100%告诉我们该数据是否一定在DB中，只是描述可能性的大小** 。

##  4. <a name='Redis-1'></a>Redis如何应对缓存击穿？

缓存击穿和缓存穿透从名词上可能很难区分开来，它们的区别是：穿透表示底层数据库没有数据且缓存内也没有数据，击穿表示底层数据库有数据而缓存内没有数据。当热点数据key从缓存内失效时，大量访问同时请求这个数据，就会将查询下沉到数据库层，此时数据库层的负载压力会骤增，我们称这种现象为"缓存击穿"。

**解决方法如下:**

- 延长热点key的过期时间或者设置永不过期，如排行榜，首页等一定会有高并发的接口；
- 利用互斥锁保证同一时刻只有一个客户端可以查询底层数据库的这个数据，一旦查到数据就缓存至Redis内，避免其他大量请求同时穿过Redis访问底层数据库；

##  5. <a name='Redis-1'></a>Redis如何应对缓存雪崩?

缓存雪崩是缓存击穿的"大面积"版，缓存击穿是数据库缓存到Redis内的热点数据失效导致大量并发查询穿过redis直接击打到底层数据库，而缓存雪崩是指Redis中大量的key几乎同时过期，然后大量并发查询穿过redis击打到底层数据库上，此时数据库层的负载压力会骤增，我们称这种现象为"缓存雪崩"。事实上缓存雪崩相比于缓存击穿更容易发生，对于大多数公司来讲，同时超大并发量访问同一个过时key的场景的确太少见了，而大量key同时过期，大量用户访问这些key的几率相比缓存击穿来说明显更大。

**解决方法如下:**

- 在可接受的时间范围内随机设置key的过期时间，分散key的过期时间，以防止大量的key在同一时刻过期；
- 对于一定要在固定时间让key失效的场景(例如每日12点准时更新所有最新排名)，可以在固定的失效时间时在接口服务端设置随机延时，将请求的时间打散，让一部分查询先将数据缓存起来；
- 延长热点key的过期时间或者设置永不过期，这一点和缓存击穿中的方案一样；

##  6. <a name='Redis-1'></a>Redis如何做缓存预热？

缓存预热如字面意思，当系统上线时，缓存内还没有数据，如果直接提供给用户使用，每个请求都会穿过缓存去访问底层数据库，如果并发大的话，很有可能在上线当天就会宕机，因此我们需要在上线前先将数据库内的热点数据缓存至Redis内再提供出去使用，这种操作就成为"缓存预热"。

缓存预热的实现方式有很多，**比较通用的方式是写个批任务，在启动项目时或定时去触发将底层数据库内的热点数据加载到缓存内。**

##  7. <a name='Redis-1'></a>Redis如何保持缓存一致性?

缓存服务（Redis）和数据服务（底层数据库）是相互独立且异构的系统，在更新缓存或更新数据的时候无法做到原子性的同时更新两边的数据，因此在并发读写或第二步操作异常时会遇到各种数据不一致的问题。如何解决并发场景下更新操作的双写一致是缓存系统的一个重要知识点。

###  7.1. <a name='CacheAside'></a>Cache Aside

查询：先查缓存，缓存没有就查数据库，然后加载至缓存内；更新：先更新数据库，然后让缓存失效；或者先失效缓存然后更新数据库； 

###  7.2. <a name='ReadThrough'></a>Read Through

在查询操作中更新缓存，即当缓存失效时，Cache Aside 模式是由调用方负责把数据加载入缓存，而 Read Through 则用缓存服务自己来加载；

###  7.3. <a name='WriteThrough'></a>Write Through

在更新数据时发生。当有数据更新的时候，如果没有命中缓存，直接更新数据库，然后返回。如果命中了缓存，则更新缓存，然后由**缓存自己更新数据库**； 

###  7.4. <a name='WriteBehindCaching'></a>Write Behind Caching

俗称write back，在更新数据的时候，只更新缓存，不更新数据库，**缓存会异步地定时批量更新数据库**；

##  8. <a name='Redis-1'></a>Redis如何回收缓存(针对内存空间不足)?

|    回收策略     |                             说明                             |
| :-------------: | :----------------------------------------------------------: |
|   noeviction    | 返回错误当内存限制达到并且客户端尝试执行会让更多内存被使用的命令(写入命令) |
|   allkeys-lru   |                   尝试回收最近最少使用的键                   |
|  volatile-lru   |       尝试回收最近最少使用的键，但仅限于在过期集合的键       |
| volatile-random |                    随机回收过期集合中的键                    |
|  volatile-ttl   |            回收过期集合的键，并优先回收TTL短的键             |
|  volatile-lfu   |        从所有配置了过期时间的键中驱逐使用频率最少的键        |
|   allkeys-lfu   |                   回收最近使用频率最少的键                   |
| allkeys-random  |                          随即回收键                          |

**如果没有键满足回收条件，volatile-lru,volatile-random和volatile-ttl和noeviction差不多**

###  8.1. <a name=''></a>如何选择合适的策略?

* 运行时查看缓存命中率和使用情况决定使用何种或者使用何种组合过期策略

##  9. <a name='Redis-1'></a>Redis过期键有哪些删除策略?

###  9.1. <a name='-1'></a>键的过期精度

在redis 2.4版本之前，过期时间可能不是一个精确的数字，有0-1秒的误差。

在redis 2.6版本之后，过期时间误差缩小在0-1ms之间

###  9.2. <a name='-1'></a>过期和持久

keys的过期是按使用Unix时间戳存储(从redis 2.6之后开始使用以ms为单位的过期时间)。这意味着即使redis实例不可用，时间也是在一直流逝的。要想过期工作做得好，redis必须有一个精确且稳定的过期时间。如果想在两个RDB文件在两台时钟不同的计算机之间进行同步，将会出现意想不到的意外。

###  9.3. <a name='key'></a>过期key的删除策略

redis keys过期有两种方式：被动和主动。

* 当有客户端试图访问过期的Keys的时候，过期的key会被发现并被被动的删除。**当然，一些过期的key也可能永远不会被访问，因此只有被动式的删除并不能解决所有的问题。**
* redis还有主动式的删除：**redis会首秀按随机进行20个keys相关的过期检测，然后删除检测到的已经过期的key，如果有多于25%的keys过期，重复第一个步骤。**

###  9.4. <a name='AOFkey'></a>在复制AOF文件时出现过期key该怎么办?

当一个key过期，`DEL`将会随着AOF文字一起合成到所有附件的slave。在master实例中，这种方法是集中的，并且不存在一致性错误的机会。然而，当slave连接到master时，不会独立过期keys(会等到master的`DEL`)，他们可能会在数据集中存在，所以当slave当选为master时淘汰机制将会独立执行，然后成为master。

##  10. <a name='Redis-1'></a>简述Redis持久化机制

Redis提供了不同级别的持久化方式：

* RDB持久化能够在指定的时间间隔对你的数据进行快照存储
* AOF持久化方式记录每次对服务器写操作，当服务器重启的时候会重新执行这些命令来恢复原始数据，AOF命令以redis协议**追加**保存每次写操作到文件末尾。redis还能对AOF文件进行后台重写，使得AOF文件的体积不至于太大。
* 如果只希望运行单机版的redis服务器，可以不适用任何形式的持久化操作。
* redis也支持同时开启两种持久化方式，在这种情况下，当redis重启的时候，会优先载入AOF文件来恢复原始的数据，因为通常情况下AOF文件保存到数据集比RDB文件保存的数据集要更加完整。
* **最重要的是RDB和AOF持久化的方式不同**

###  10.1. <a name='RDB'></a>RDB的优点

* RDB是一个非常紧凑的结构，保存了某个时间点的数据集，非常适用于数据集的备份，比如你可以在每个小时保存过去24小时内的数据，因此每天保存过去30天的数据，这样即使除了问题也可以根据需求恢复到不同版本的数据集。
* RDB是一个紧凑单一的文件，很方便传送到另一个远端数据中心或者亚马逊的S3(可能加密)，非常适用于灾难的恢复。
* RDB在保存RDB文件的时候父进程唯一需要做的就是fork一个子进程，接下来的工作全部交给子进程来做，父进程不需要再做其他IO操作，所以RDB持久化方式可以最大化redis的性能。
* 与AOF相比，在恢复大的数据集的时候，RDB的方式会更加快一点。

###  10.2. <a name='RDB-1'></a>RDB的缺点

* 如果redis意外停止工作，并且要求数据丢失最少的情况下，RDB不适合，因为保存完整数据集是一个非常繁重的任务，通常RDB会每隔几分钟做一个完整的保存，万一redis宕机，有可能会丢失几分钟的数据。
* RDB进场需要fork子进程来保存数据集到硬盘之上，当数据集比较大的时候，fork的过程是非常耗时的，可能会导致redis在一些毫秒级内不能响应客户端的请求，如果数据集巨大并且CPU性能不好的情况下，这种情况会持续1s，AOF也需要fork，但是可以调整重写日志文件的频率来提高数据集的耐久度。

###  10.3. <a name='AOF'></a>AOF的优点

* 使用AOF会让你的redis更加耐久，你可以使用不同的`fsync`策略，无`fsync`， 每秒`fsync`，每次写的时候`fsync`，使用默认的每秒`fsync`策略，redis的性能依然很好，因为`fsync`是由后台线程进行处理的，主线程会尽力处理客户端的请求，一旦出现故障，最多丢失1s的数据。
* AOF文件是一个只进行追加的日志文件，所以不需要写入`seek`，即使由于某些原因(磁盘空间已满，写的过程有宕机策略等等)未执行完成的写入命令，也可以使用`redis-check-aof`工具修复AOF文件。
* redis可以在AOF体积过大的时候，重写AOF文件，重写后的AOF文件包含了恢复当前数据集所需的最小命令集合，整个重写过程是安全的，因为redis在创建新的AOF文件的过程中，会继续将当前命令追加到AOF文件里面，即使重写过程发生了停机，现有的AOF文件也不会丢失。而一旦新的AOF文件创建完毕，redis就会从旧的AOF文件切换到新的AOF文件中，并开始对新的AOF文件进行追加操作。
* AOF文件有序地保存了对数据库执行的所有写入操作，这些写入操作以redis写的形式保存，因此AOF文件的内容非常容易读懂，对文件进行分析也会轻松。导出AOF文件也会非常简单。

###  10.4. <a name='AOF-1'></a>AOF缺点

* 对于相同的数据集，AOF文件的体积通常要大于RDB文件的体积。
* 根据所使用的`fsync`策略，AOF速度可能会慢于RDB。在一般情况下，每秒`fsync`的性能依然非常高，而关闭`fsync`可以让AOF的速度和RDB一样快，即使在高负荷之下也是如此。不过在处理巨大的写入载入时，RDB可以提供更有保证的最大延迟时间。

###  10.5. <a name='X'></a>4.X版本的持久化策略整合

* 对AOF重写策略进行了优化
* 再重写AOF文件的时候，`4.x`版本之前是把内存数据集的操作指令落地，而新的版本把内存中的数据集以rdb的方式落地。**这样重写后的AOF依然时追加日志，但是在恢复的时候先RDB再增量的日志，性能非常优秀**

##  11. <a name='Redis-1'></a>简述Redis主从复制

###  11.1. <a name='-1'></a>主从复制的过程

* 当master和slave正常连接的时候，通过一连串数据流来保证数据的同步
* 当master和slave断开又连接，断开时间不长的情况下，只要恢复少许数据。如果断开时间过长，需要进行全量复制。

###  11.2. <a name='-1'></a>主从复制的关注点

* redis使用异步复制，slave和master之间异步地确认处理地数据量
* 一个master可以有多个slave，slave也可以接受其他slave的连接，除了多个slave可以连接到一个master之外，slave之间也可以像层状结构那样连接到其他的slave。自redis 4.x版本起，所有的sub-slave将从master收到完全一样的数据流
* 复制在master侧时非阻塞的，这意味着master在一个或者多个slave进行初次同步或者部分重同步时，可以继续处理查询请求。
* 复制在slave侧大部分也是非阻塞的，当slave进行初次同步的时候，它可以使用旧的数据处理查询请求，假设你在`redis.conf`中配置了让redis这样做的话。否则，你可以配置如果复制流断开，redis slave会返回一个error给客户端。但是，在初次同步之后，旧的数据集必须被删除，同时加载新的数据集。slave在这个短暂的时间窗口内(如果数据量很大，也可能时间很长)，会阻塞到来的连接请求。自redis 4.0开始，可以配置redis使删除旧数据集的操作在另外一个不同的线程中进行，但是，加载新的数据集的操作依然需要在主线程中进行并且会阻塞slave。
* 复制既可以被用在可伸缩性，以便只读查询可以有多个slave进行
* 可以避免使用复制来避免master将全部数据集写入磁盘造成的开销，一种典型的技术是配置你的master的`redis.conf`以避免对磁盘进行持久化，然后连接一个slave，其配置为不定期保存或者启用AOF，但是这个设置必须小心，因为重新启动的master程序将从一个空的数据集开始。如果一个slave试图和master同步，那么这个slave也会被清空。
* **如果master使用复制功能但是未配置持久化，那么请把自动重启功能关闭** 。

###  11.3. <a name='-1'></a>复制的流程

每个redis master都有一个replication ID: 这是一个较大的伪随机字符串，标记了一个给定的数据集。每个master也持有一个偏移量，master将自己产生的复制流发送给slave的时候，发送了多少个字节的数据，自身的偏移量就会增加多少，目的是当有新的操作修改自己的数据集的时候，它可以以此更新slave的状态。复制偏移量即使在没有一个slave连接到master的时候，也会自增，所以基本每一对给定的`(replication ID, offset)`都会标识一个master数据集的精确版本。

当slave连接到master的时候，它们使用`PSYNC`命令来发送它们记录的旧的master replication ID和它们至今为止处理的偏移量。通过这种方式，master能够仅发送slave所需的增量部分。但是如果master的缓冲区中没有足够的命令积压缓冲记录，或者如果slave引用了不再知道的历史replication ID，则会转而进行一个全量的重型同步：在这种情况下，slave会得到一个完整的数据集副本，从头开始。

**下面是全量同步的工作细节** ：

master开启了一个后台保存进程，以便生产一个RDB文件，同时它开始缓冲所有从客户端接收到的新的写入命令。当后台保存完成的时候，master会将数据集文件传输给slave，slave将之保存在磁盘上，然后加载文件到内存当中。再然后master会发送所有缓冲的命令给slave。这个过程以指令流的形式完成并且和redis协议本身的格式相同。

**下面是无需磁盘参与的复制** ：

正常情况下，一个全量重同步要求在磁盘上创建一个RDB文件，然后将它从磁盘加载进内存中，然后slave以此形式进行数据同步。如果磁盘性能很低，这对master压力很大。redis 2.8.18是第一个支持无磁盘复制的版本。在此设置中，子进程直接发送RDB给slave，无需使用磁盘作为中间存储介质。

