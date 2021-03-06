# 一、数据分布

分布式数据库首先要解决的是要把整个数据集按照分区规则映射到多个节点的问题，即把数据集划分到多个节点上，每个节点负责整体数据的一个子集。常见的分区规则有哈希分区和顺序分区两种。Redis采用的是哈希分区

哈希分区一般分为以下几种：

1、节点取余分区：

使用特定的数据，如redis的键或用户ID，再根据节点数量N使用公式$hash(key)$%$N$计算出哈希值，用来决定数据映射到哪一个节点上。

缺点：当节点数量变化时，如扩容或收缩节点，数据节点映射关系需要重新计算，会导致数据的重新迁移

优点：简单，常用于数据库的分库分表规则，一般采用预分区的方式，提前根据数据量规划好分区数

2、一致性哈希

一致性哈希分区实现思路是为系统中每个节点分配一个token，范围一般在0～2^32^，这些token构成一个哈希环。数据读写执行节点查找操作时，现根据key计算哈希值，然后顺时针找到第一个大于等于该哈希值的token节点

优点：这宗方式相比节点取余最大的好处在于加入和删除节点只影响哈希环中相邻的节点，对其它节点无影响

缺点： 

- 加减节点会造成哈希环中部分数据无法命中，需要手动处理或者忽略这部分数据，因此一致性哈希常用于缓存场景
- 当使用少量节点时，节点变化将大范围影响哈希环中的数据映射，因此这种方式不适合少量数据节点的分布式方案
- 普通的一致性哈希分区在增减节点时需要增加一倍或减去一半节点才能保证数据和负载的均衡

3、虚拟槽分区

 虚拟槽分区使用分散度良好的哈希函数把所有数据映射到一个固定范围的整数集合中，整数定义为槽。这个范围一般远大于节点数，比如redis Cluster槽范围为0～16383.槽时集群内数据管理和迁移的基本单位。采用大范围槽主要目的是问了方便数据拆分和集群扩展

redis数据分区： 

redis cluster采用虚拟槽分区，所有的键根据哈希函数映射到0～16383整数槽内，计算公式：$slot=CRC(key)$&$16383$。每一个节点负责维护一部分槽以及槽所映射的键值数据

redis虚拟槽分区的特点：

- 解耦数据和节点之间的关系，简化了节点扩容和收缩的难度
- 节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区元数据
- 支持节点、槽、键之间的映射查询，用于数据路由、在线伸缩等场景 

集群功能的一些限制：

1、key批量操作支持有限，只支持具有相同slot值的key执行批量操作

2、只支持同一个节点上的多个key的事物操作

3、key作为数据分区的最小力度，因此不能将一个大的键值对象如hash、list等映射到不同的节点

4、不支持多数据库空间，单机下redis可以支持16个数据库，集群模式下只能支持一个数据库空间，也就是db 0

5、复制结构只能支持一层，从节点只能复制主节点，不支持嵌套梳妆复制结构

# 二、集群配置

## 1、节点握手

节点握手是指一批运行在集群模式下的节点通过Gossip协议彼此通信，达到感知对方的过程

我们只需要在集群内任意节点上执行`cluster meet`命令加入新节点，握手状态会通过消息在集群内传播，这样其它节点会自动发现新节点并发起握手流程

节点建立握手后集群还不能正常工作，这时集群处于下线状态，所有的数据读写都被禁止,可以通过`cluster info`命令获取集群的当前状态,因为目前所没有槽被分配到节点，因此集群无法完成槽到节点的映射。只有当16384个槽全部分配给节点之后，集群才进入到在线状态

## 2、节点握手

redis集群把所有的数据映射到16384个槽中。每个key会映射为一个固定的槽，只有当前节点分配了槽，才能响应和这些槽关联的键命令。通过`cluster addslots`命令可以为节点分配槽

作为一个完整的集群，每个负责处理槽的节点应该具有从节点，保证当它出现故障时可以自动进行故障转移。集群模式下，redis节点角色分为主节点和从节点。首次启动的节点和被分配槽的节点都是主节点，从节点负责复制主节点槽信息和相关数据。适用`cluster replication [nodeId]`命令让一个节点成为从节点。其中命令执行必须在对应的节点上执行，nodeId时要复制主节点的节点ID

## 3、集群的完整性检查

集群完整性是指所有的槽都分配到存活的主节点上，只要16384个槽中有一个没有分配给节点则表示集群不完整

# 三、节点通信

**Gossip协议：**Gossip协议的工作原理就是节点彼此不断通信交换信息，一段时间后所有的节点都会知道集群完整的信息

## 1、通信流程

1）集群中的每个节点都会单独开辟一个TCP通道，用于节点之间彼此通信，通信端口号在基础端口上加10000

2）每个节点在固定周期内通过特定规则选择几个节点发送ping消息

3）收到ping消息的节点用pong消息作为响应

## 2、Gossip消息

Gossip消息的主要职责就是信息交换。信息交换的载体就是节点彼此发送的Gossip消息，常用的Gossip消息可分为：ping消息、pong消息、meet消息、fail消息等

1）meet消息：用于通知新节点加入集群

2）ping消息：集群内交换最频繁的消息，集群内每个节点每秒向多个其它节点发送ping消息，用于检测节点是否在线和交换彼此状态信息

3）pong消息：当收到ping、meet消息时，作为响应消息回复给发送方确认消息正常通信。pong消息封装了自身的状态数据。节点也可以向集群内广播自身的pong消息来通知整个集群对自身状态进行更新

4）fail消息：当节点判定集群内另一节点下线时，会向集群内广播一个fail消息，其它节点接收到fail消息之后会把对应节点更新为下线状态

 所有的消息格式划分为：消息头和消息题。消息头包含发送节点自身状态数据，接收节点根据消息头就可以获取到发送节点的相关数据

**解析消息头过程：**

消息头包含了发送节点的信息，如果发送节点时新节点且消息是meet类型，则加入到本地节点列表；如果是已知节点，则尝试更新发送节点的状态，如槽映射关系主从角色等状态

**解析消息体过程：**

如果消息体的clusterMsgDataGossip数组包含的节点是新节点，则尝试发起与新节点的meet握手流程；如果是已知节点，则根据clusterMsgDataGossip中的flags字段判断该节点是否下线，用于故障转移

# 四、请求路由

## 1、请求重定向

在集群模式下，redis接收任何键相关命令时首先计算键对应的槽，再根据槽找出所对应的节点，如果节点是自身，则处理键命令；否则回复MOVED重定向错误，通知客户端请求真确的节点。这个过程称为重定向

**节点对于不属于它的键命令至回复重定向响应，并不负责转发**

键命令执行的步骤主要分两步：计算槽，查找槽所对应的节点 

**计算槽：**redis首先需要计算键所对应的槽。根据键的有小部分（若有大括号，就是大括号包围的部分）使用CRC16函数计算出散列值，再取对16383的余数，使每个键都可以映射到0～16383的槽范围内，键内部使用大括号包含的内容叫做hash tag，它提供不同的键可以具备相同slot的功能

**槽节点查找：**Jedis的槽定向

# 五、ASK重定向

ASK与MOVED虽然都是对客户端的重定向控制，但是有着本质的区别。ASK重定向说明集群正在进行slot数据迁移，客户端无法知道什么时候迁移完成，因此只能是临时性的重定向，客户端不会更新slots缓存。但是MOVED重定向说明键对应的槽已经明确指定到新的节点，因此需要更新slots缓存

# 六、故障转移

**主观下线：**指某个节点认为另一个节点不可用，即下线状态，这个状态并不是最终的故障判定，只能代表一个节点的意见，可能存在误判的情况

**客观下线：**指标记一个节点真正的下线，集群内多个节点都认为该节点不可用，从而达成共识的结果。如果是持有槽的主节点故障，需要为该节点进行故障转移

