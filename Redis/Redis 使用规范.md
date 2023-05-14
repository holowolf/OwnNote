第一章节：

（1）Redis介绍
----Redis是一个开源的，基于内存的结构化数据存储媒介，可以作为数据库、缓存服务或消息服务使用。
----Redis支持多种数据结构，包括字符串、哈希表、链表、集合、有序集合、位图、Hyperloglogs等。
----Redis具备LRU淘汰、事务实现、以及不同级别的硬盘持久化等能力，并且支持副本集和通过Redis Sentinel实现的高可用方案，同时还支持通过Redis Cluster实现的数据自动分片能力。
----Redis的主要功能都基于单线程模型实现，也就是说Redis使用一个线程来服务所有的客户端请求，同时Redis采用了非阻塞式IO，并精细地优化各种命令的。
（2）时间复杂度介绍
----常见的时间复杂度有：常数阶O(1),对数阶O(log2n),线性阶O(nlogn)
, 线性对数阶O(nlog2n),平方阶O(n2)，立方阶O(n3),...， k次方阶O(nk),指数阶O(2n)。随着问题规模n的不断增大，上述时间复杂度不断增大，算法的执行效率越低。
----算法时间复杂度，这些信息意味着: 1.Redis是线程安全的（因为只有一个线程），其所有操作都是原子的，不会因并发产生数据异常
2.Redis的速度非常快（因为使用非阻塞式IO，且大部分命令的算法时间复杂度都是O(1))
3.使用高耗时的Redis命令是很危险的，会占用唯一的一个线程的大量处理时间，导致所有的请求都被拖慢。（例如时间复杂度为O(N)的KEYS命令，严格禁止在生产环境中使用）
第二章节：

（1）Redis常用架构
-----redis主从---redis+keepalived
-----redis集群---redis-cluster
-----redis代理---tweeproxy+redis+keepalived
第三章节：

Redis常用数据结构
（1） key
redis采用key-value基本的数据结构，任何二进制序列都可以作为Redis的Key使用（例如普通的字符串或一张JPEG图片），以下是key使用的一些建议：
1.不要使用过长的Key。例如使用一个1024字节的key就不是一个好主意，不仅会消耗更多的内存，还会导致查找的效率降低， Key短到缺失了可读性也是不好的，例如”u1000flw”比起”user:1000:followers”来说，节省了寥寥的存储空间，却引发了可读性和可维护性上的麻烦；最好使用统一的规范来设计Key，比如“GROUP_ACTIVITY_MEMBER_KEY = "G:GROUP:ACTIVITY:MEMBER";”
2.Redis允许的最大Key长度是512MB（对Value的长度限制也是512MB）
（2） string
string是redis的基本数据类型，所有的基本类型在redis中都以string体现。
（3） list
Redis的List是链表型的数据结构，可以使用LPUSH/RPUSH/LPOP/RPOP等命令在List的两端执行插入元素和弹出元素的操作。
虽然List也支持在特定index上插入和读取元素的功能，但其时间复杂度较高（O(N)），应小心使用。
（4） hash
Hash即哈希表，redis的hash和传统的哈希表一样，是一种field-value型的数据结构，可以理解成将HashMap搬入redis。
Hash非常适合用于表现对象类型的数据，用Hash中的field对应对象的field即可。 Hash的优点包括： 1.可以实现二元查找，如”查找ID为1000的用户的年龄”
2.比起将整个对象序列化后作为String存储的方法，Hash能够有效地减少网络传输的消耗
3.当使用Hash维护一个集合时，提供了比List效率高得多的随机访问命令
（5）set集合
集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。
（6）zset有序集合
zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。 不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
zset的成员是唯一的,但分数(score)却可以重复。
Redis Sorted Set是有序的、不可重复的String集合。Sorted Set中的每个元素都需要指派一个分数(score)，Sorted Set会根据score对元素进行升序排序。如果多个member拥有相同的score，则以字典序进行升序排序。
Sorted Set非常适合用于实现排名。
第四章节：

Redis常用命令以及注意事项
（1）与String相关的常用命令：
1.SET：为一个key设置value，可以配合EX/PX参数指定key的有效期，通过NX/XX参数针对key是否存在的情况进行区别操作，时间复杂度O(1)
2.GET：获取某个key对应的value，时间复杂度O(1)
3.GETSET：为一个key设置value，并返回该key的原value，时间复杂度O(1)
4.MSET：为多个key设置value，时间复杂度O(N)
5.MSETNX：同MSET，如果指定的key中有任意一个已存在，则不进行任何操作，时间复杂度O(N)
6.MGET：获取多个key对应的value，时间复杂度O(N)
（2）与List相关的常用命令：
1.LPUSH：向指定List的左侧（即头部）插入1个或多个元素，返回插入后的List长度。时间复杂度O(N)，N为插入元素的数量
2.RPUSH：同LPUSH，向指定List的右侧（即尾部）插入1或多个元素
3.LPOP：从指定List的左侧（即头部）移除一个元素并返回，时间复杂度O(1)
4.RPOP：同LPOP，从指定List的右侧（即尾部）移除1个元素并返回
5.LPUSHX/RPUSHX：与LPUSH/RPUSH类似，区别在于，LPUSHX/RPUSHX操作的key如果不存在，则不会进行任何操作
6.LLEN：返回指定List的长度，时间复杂度O(1)
7.LRANGE：返回指定List中指定范围的元素（双端包含，即LRANGE key 0 10会返回11个元素），时间复杂度O(N)。应尽可能控制一次获取的元素数量，一次获取过大范围的List元素会导致延迟，同时对长度不可预知的List，避免使用8.LRANGE key 0 -1这样的完整遍历操作。
应谨慎使用的List相关命令： 1.LINDEX：返回指定List指定index上的元素，如果index越界，返回nil。index数值是回环的，即-1代表List最后一个位置，-2代表List倒数第二个位置。时间复杂度O(N)
2.LSET：将指定List指定index上的元素设置为value，如果index越界则返回错误，时间复杂度O(N)，如果操作的是头/尾部的元素，则时间复杂度为O(1)
3.LINSERT：向指定List中指定元素之前/之后插入一个新元素，并返回操作后的List长度。如果指定的元素不存在，返回-1。如果指定key不存在，不会进行任何操作，时间复杂度O(N)
（3）与Hash相关的常用命令：
1.HSET：将key对应的Hash中的field设置为value。如果该Hash不存在，会自动创建一个。时间复杂度O(1)
2.HGET：返回指定Hash中field字段的值，时间复杂度O(1)
3.HMSET/HMGET：同HSET和HGET，可以批量操作同一个key下的多个field，时间复杂度：O(N)，N为一次操作的field数量
4.HSETNX：同HSET，但如field已经存在，HSETNX不会进行任何操作，时间复杂度O(1)
5.HEXISTS：判断指定Hash中field是否存在，存在返回1，不存在返回0，时间复杂度O(1)
6.HDEL：删除指定Hash中的field（1个或多个），时间复杂度：O(N)，N为操作的field数量
7.HINCRBY：同INCRBY命令，对指定Hash中的一个field进行INCRBY，时间复杂度O(1)
应谨慎使用的Hash相关命令：
1.HGETALL：返回指定Hash中所有的field-value对。返回结果为数组，数组中field和value交替出现。时间复杂度O(N)
2.HKEYS/HVALS：返回指定Hash中所有的field/value，时间复杂度O(N)
（4）与Set相关的常用命令：
1.SADD：向指定Set中添加1个或多个member，如果指定Set不存在，会自动创建一个。时间复杂度O(N)，N为添加的member个数
2.SREM：从指定Set中移除1个或多个member，时间复杂度O(N)，N为移除的member个数
3.SRANDMEMBER：从指定Set中随机返回1个或多个member，时间复杂度O(N)，N为返回的member个数
4.SPOP：从指定Set中随机移除并返回count个member，时间复杂度O(N)，N为移除的member个数
5.SCARD：返回指定Set中的member个数，时间复杂度O(1)
6.SISMEMBER：判断指定的value是否存在于指定Set中，时间复杂度O(1)
7.SMOVE：将指定member从一个Set移至另一个Set
慎用的Set相关命令：
1.SMEMBERS：返回指定Hash中所有的member，时间复杂度O(N)
2.SUNION/SUNIONSTORE：计算多个Set的并集并返回/存储至另一个Set中，时间复杂度O(N)，N为参与计算的所有集合的总member数
3.SINTER/SINTERSTORE：计算多个Set的交集并返回/存储至另一个Set中，时间复杂度O(N)，N为参与计算的所有集合的总member数
4.SDIFF/SDIFFSTORE：计算1个Set与1或多个Set的差集并返回/存储至另一个Set中，时间复杂度O(N)，N为参与计算的所有集合的总member数。
（5）Sorted Set的主要命令：
1.ZADD：向指定Sorted Set中添加1个或多个member，时间复杂度O(Mlog(N))，M为添加的member数量，N为Sorted Set中的member数量
2.ZREM：从指定Sorted Set中删除1个或多个member，时间复杂度O(Mlog(N))，M为删除的member数量，N为Sorted Set中的member数量
3.ZCOUNT：返回指定Sorted Set中指定score范围内的member数量，时间复杂度：O(log(N))
4.ZCARD：返回指定Sorted Set中的member数量，时间复杂度O(1)
5.ZSCORE：返回指定Sorted Set中指定member的score，时间复杂度O(1)
6.ZRANK/ZREVRANK：返回指定member在Sorted Set中的排名，ZRANK返回按升序排序的排名，ZREVRANK则返回按降序排序的排名。时间复杂度O(log(N))
7.ZINCRBY：同INCRBY，对指定Sorted Set中的指定member的score进行自增，时间复杂度O(log(N))
慎用的Sorted Set相关命令：
1.ZRANGE/ZREVRANGE：返回指定Sorted Set中指定排名范围内的所有member，ZRANGE为按score升序排序，ZREVRANGE为按score降序排序，时间复杂度O(log(N)+M)，M为本次返回的member数
2.ZRANGEBYSCORE/ZREVRANGEBYSCORE：返回指定Sorted Set中指定score范围内的所有member，返回结果以升序/降序排序，min和max可以指定为-inf和+inf，代表返回所有的member。时间复杂度O(log(N)+M)
3.ZREMRANGEBYRANK/ZREMRANGEBYSCORE：移除Sorted Set中指定排名范围/指定score范围内的所有member。时间复杂度O(log(N)+M)
第五章节：

Redis性能调优
（1）确保没有让redis执行耗时长的命令
-------不要把List当做列表使用，仅当做队列来使用
-------严格控制Hash、Set、Sorted Set的大小
-------绝对禁止使用KEYS命令
-------避免一次性遍历集合类型的所有成员，而应使用SCAN类的命令进行分批的，游标式的遍历
-------通过slowlog get来获取慢日志
（2）网络引发的延迟
-------尽量使用长连接或者连接池
（3）数据持久化引发的延迟
-------AOF + fsync always的设置虽然能够绝对确保数据安全，但每个操作都会触发一次fsync，会对Redis的性能有比较明显的影响
-------AOF + fsync every second是比较好的折中方案，每秒fsync一次
-------AOF + fsync never会提供AOF持久化方案下的最优性能
-------使用RDB持久化通常会提供比使用AOF更高的性能
（4）引入读写分离机制
-------主节点接收所有写请求，并将数据同步给多个从节点。在这一基础上，我们可以让从节点提供对实时性要求不高的读请求服务，以减小主节点的压力。 尤其是针对一些使用了长耗时命令的统计类任务，完全可以指定在一个或多个从节点上执行，避免这些长耗时命令影响其他请求的响应。这个还是根据架构来
（5）设置key的失效时间
-------expire key 30（http://doc.redisfans.com/key/expire.html---作为参考，分析的很好）
（6）Hashes是非常不错的选择
（7）修改系统内核参数vm.overcommit_memory为1
Redis内存调优
（1）不要开启redis的vm（虚拟内存），确保vm-enabled参数值为no
（2）设置maxmemory的时候，可以根据系统业务的需要设置特定的可分配内存大小
（3）如果key的数据类型使用hashmap的话，可以通过以下参数节省内存空间：
hash-max-zipmap-entries 64
hash-max-zipmap-value 512
hash-max-zipmap-entries
--------含义是当value这个Map内部不超过多少个成员时会采用线性紧凑格式存储，默认是64,即value内部有64个以下的成员就是使用线性紧凑存储，超过该值自动转成真正的HashMap。参数的设置不宜过大，不然适得其反！
--------同样的参数配置还有：
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
（5） 内存的优化还是要看key-value的存储的方式，根据业务来进行调优
（6） 对能设置失效key的进行失效设置EXPIRE
Redis主从复制
（1）主从复制原理
slave发送PSYNC
master执行bgsave生成RDB快照文件，同时将这之后新的写命令记入缓冲区
master向slave发送快照文件，并继续记录写命令
slave接收并保存快照
slave将快照文件载入内存
slave开始接收master中缓冲区的命令完成同步
（2）主从复制注意事项
1.由于bgsave是走内存，会损耗当前系统的内存，当物理内存不足时，swap会随之出现不足现象导致系统假死，所以这里需要关注当前的物理内存free
2.强烈要求开发对可进行失效时间的key做失效时间设置！
第六章节：

Redis监控：redis-live
第七章节：

Redis数据备份以及导入
（1）通过redis-dump工具，该工具基于ruby语言进行编写，需要安装ruby、rubygem、以及相关的插件。通过命令redis-dump和redis-load进行导出和插入下面是导出来的格式
（2）通过redis-cli，进入后台执行命令bgsave，生成一个rdb文件
第八章节：

Redis-FAQ
一：咪咕善跑重构社区使用redis案列分析
-----场景
咪咕善跑社区有个最美路线的活动，该活动提供用户点赞功能，每次用户点赞均缓存在redis中
-----问题
运营侧每上线一个最美路线，现网的redis主从均会进行切换，通过日志查看到是redis响应延时，导致ping命令长时间不返回PONG，引起主从切换
-----问题分析、
（1）通过查看slowlog get获取当前的慢日志，查看到当前某个key最慢
（2）通过命令hget获取该key值，发现key值获取时间有几十秒
（3）查看该类key的value值，存储值有30万+
（4）有大量的hgetall命令，该命令时间复杂度O(N)
（5）删除掉该key后，redis恢复正常
------问题总结、
（1）与开发沟通，修改key的存储方式
（2）梳理redis不合理的数据类型和命令
-------查询命令介绍、
（1）INFO commandstats 查看使用命令的次数、耗时、平均耗时
（2）redis-cli -h IP -p PORT –bigkeys 统计key的分布以及占用内存
（3）slowlog get 查询慢日志
二：大内存key存储导致碎片率高
-----场景
redis碎片率高，使用内存占比大
-----问题分析、
（1）通过redis-dump工具将排名前3000的大key导出
（2）查看大key是否是热点key
（3）每个key占用内存太大，key value不对应出现内存碎片
------问题总结、
（1）与开发沟通，将大key进行压缩
（2）设置内存淘汰策略，将不是常用的非热点key进行淘汰