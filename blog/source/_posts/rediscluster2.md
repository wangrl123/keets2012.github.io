---
title: Redis Cluster深入与实践（续）
date: 2018-1-9
categories: 中间件
tags:
- redis 
- mq
---
## 前文回顾

上一篇文章[基于redis的分布式锁实现](http://blueskykong.com/2018/01/06/redislock/)写了基于redis实现的分布式锁。分布式环境下，不会还使用单点的redis，做到高可用和容灾，起码也是redis主从。redis的单线程工作，一台物理机只运行一个redis实例太过浪费，redis单机显然是存在单点故障的隐患。内存资源往往受限，纵向不停扩展内存并不是很实际，因此横向可伸缩扩展，需要多台主机协同提供服务，即分布式下多个Redis实例协同运行。

在之前的文章[Redis Cluster深入与实践](http://blueskykong.com/2017/09/29/rediscluster/)介绍过Redis Cluster的相关内容，之前特地花时间在redis官网看了redis cluster的相关文档和实现。本文是那篇文章的续集，因为笔者最近在调研redis的主从切换到redis 集群的方案，将会讲下redis集群的几种方案选型和redis cluster的实践。

redis集群的几种实现方式如下：

- 客户端分片，如redis的Java客户端jedis也是支持的，使用一致性hash
- 基于代理的分片，如codis和Twemproxy
- 路由查询， redis-cluster

下面我们分别介绍下这几种方案。

## 客户端分片
Redis Sharding是Redis Cluster出来之前，业界普遍使用的多Redis实例集群方法。其主要思想是采用哈希算法将Redis数据的key进行散列，通过hash函数，特定的key会映射到特定的Redis节点上。java redis客户端驱动jedis，支持Redis Sharding功能，即ShardedJedis以及结合缓存池的ShardedJedisPool。

![Sharding](http://ovcjgn2x0.bkt.clouddn.com/redissharding.jpg "Sharding")

Redis Sentinel提供了主备模式下Redis监控、故障转移功能达到系统的高可用性。在主Redis宕机时，备Redis接管过来，上升为主Redis，继续提供服务。主备共同组成一个Redis节点，通过自动故障转移，保证了节点的高可用性。

客户端sharding技术其优势在于非常简单，服务端的Redis实例彼此独立，相互无关联，每个Redis实例像单服务器一样运行，非常容易线性扩展，系统的灵活性很强。

客户端sharding的劣势也是很明显的。由于sharding处理放到客户端，规模进一步扩大时给运维带来挑战。客户端sharding不支持动态增删节点。服务端Redis实例群拓扑结构有变化时，每个客户端都需要更新调整。连接不能共享，当应用规模增大时，资源浪费制约优化。

## 基于代理的分片
客户端发送请求到一个代理组件，代理解析客户端的数据，并将请求转发至正确的节点，最后将结果回复给客户端。

该模式的特性如下：

- 透明接入，业务程序不用关心后端Redis实例，切换成本低。
- Proxy 的逻辑和存储的逻辑是隔离的。
- 代理层多了一次转发，性能有所损耗。

简单的结构图如下：

![proxy](http://ovcjgn2x0.bkt.clouddn.com/redisproxy.jpg "proxy")

主流的组件有：Twemproxy和Codis。
### Twemproxy
> Twemproxy也叫nutcraker，是twtter开源的一个redis和memcache代理服务器程序。redis作为一个高效的缓存服务器，非常具有应用价值。但在用户数据量增大时，需要运行多个redis实例，此时将迫切需要一种工具统一管理多个redis实例，避免在每个客户端管理所有连接带来的不方便和不易维护，Twemproxy即为此目标而生。 

Twemproxy有以下几个特点：

- 快
- 轻量级
- 维持永久的服务端连接
- 支持失败节点自动删除；可以设置重新连接该节点的时间，还可以设置连接多少次之后删除该节点
- 支持设置HashTag；通过HashTag可以自己设定将同一类型的key映射到同一个实例上去。
- 减少与redis的直接连接数，保持与redis的长连接，可设置代理与后台每个redis连接的数目
- 自带一致性hash算法，能够将数据自动分片到后端多个redis实例上；支持多种hash算法，可以设置后端实例的权重，目前redis支持的hash算法有：one_at_a_time、md5、crc16、crc32、fnv1_64、fnv1a_64、fnv1_32、fnv1a_32、hsieh、murmur、jenkins。
- 支持redis pipelining request，将多个连接请求，组成reids pipelining统一向redis请求。
- 支持状态监控；可设置状态监控ip和端口，访问ip和端口可以得到一个json格式的状态信息串；可设置监控信息刷新间隔时间。


TwemProxy 官网介绍了如上的特性。TwemProxy的使用可以像访问redis客户端一样访问TwemProxy。然而Twitter已经很久放弃了更新TwemProxy。Twemproxy最大的痛点在于，无法平滑地扩容/缩容。Twemproxy另一个痛点是，运维不友好，甚至没有控制面板。

### Codis
> Codis是豌豆荚开源的redis集群方案，是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有显著区别 , 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。

Codis当前最新release 版本为 codis-3.2，codis-server 基于 redis-3.2.8。有一下组件组成：

![Codis](http://ovcjgn2x0.bkt.clouddn.com/architecture.png "Codis架构")

- Codis Server：基于 redis-3.2.8 分支开发。增加了额外的数据结构，以支持 slot 有关的操作以及数据迁移指令。
- Codis Proxy：客户端连接的 Redis 代理服务, 实现了 Redis 协议。 除部分命令不支持以外(不支持的命令列表)，表现的和原生的 Redis 没有区别（就像 Twemproxy）。
- Codis Dashboard：集群管理工具，支持 codis-proxy、codis-server 的添加、删除，以及据迁移等操作。在集群状态发生改变时，codis-dashboard 维护集群下所有 codis-proxy 的状态的一致性。
对于同一个业务集群而言，同一个时刻 codis-dashboard 只能有 0个或者1个；所有对集群的修改都必须通过 codis-dashboard 完成。
- Codis Admin：集群管理的命令行工具。
可用于控制 codis-proxy、codis-dashboard 状态以及访问外部存储。
- Codis FE：集群管理界面。
多个集群实例共享可以共享同一个前端展示页面；
通过配置文件管理后端 codis-dashboard 列表，配置文件可自动更新。
- Storage：为集群状态提供外部存储。
提供 Namespace 概念，不同集群的会按照不同 product name 进行组织；目前仅提供了 Zookeeper、Etcd、Fs 三种实现，但是提供了抽象的 interface 可自行扩展。

至于具体的安装与使用，见官网[CodisLabs](https://github.com/CodisLabs/codis)，不在此涉及。

Codis的特性：

- Codis支持的命令更加丰富，基本支持redis的命令。
- 迁移成本低，迁移到codis没这么麻烦，只要使用的redis命令在codis支持的范围之内，只要修改一下配置即可接入。
- Codis提供的运维工具更加友好，提供web图形界面管理集群。
- 支持多核心CPU，twemproxy只能单核
- 支持group划分，组内可以设置一个主多个从，通过sentinel 监控redis主从，当主down了自动将从切换为主


## 路由查询
Redis Cluster是一种服务器Sharding技术，3.0版本开始正式提供。Redis Cluster并没有使用一致性hash，而是采用slot(槽)的概念，一共分成16384个槽。将请求发送到任意节点，接收到请求的节点会将查询请求发送到正确的节点上执行。当客户端操作的key没有分配到该node上时，就像操作单一Redis实例一样，当客户端操作的key没有分配到该node上时，Redis会返回转向指令，指向正确的node，这有点儿像浏览器页面的302 redirect跳转。

Redis集群，要保证16384个槽对应的node都正常工作，如果某个node发生故障，那它负责的slots也就失效，整个集群将不能工作。为了增加集群的可访问性，官方推荐的方案是将node配置成主从结构，即一个master主节点，挂n个slave从节点。这时，如果主节点失效，Redis Cluster会根据选举算法从slave节点中选择一个上升为主节点，整个集群继续对外提供服务。

![Cluster](http://ovcjgn2x0.bkt.clouddn.com/rediscluster.jpg "Redis Cluster")

特点：

- 无中心架构，支持动态扩容，对业务透明
- 具备Sentinel的监控和自动Failover能力
- 客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
- 高性能，客户端直连redis服务，免去了proxy代理的损耗

缺点是运维也很复杂，数据迁移需要人工干预，只能使用0号数据库，不支持批量操作，分布式逻辑和存储模块耦合等。

选型最后确定redis cluster。主要原因是性能高，去中心化支持扩展。运维方面的数据迁移暂时业内也没有特别成熟的方案解决，redis cluster是redis官方提供，我们期待redis官方在后面能够完美支持。
### 安装
官方推荐集群至少需要六个节点，即三主三从。六个节点的配置文件基本相同，只需要修改端口号。

```
port 7000
cluster-enabled yes  #开启集群模式
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

启动后，可以看到如下的日志。

> [82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1

由于没有nodes.conf存在，每个实例启动后都会给自己分配一个ID。为了在集群的环境中有一个唯一的名字，该ID将会被永久使用。每个实例都会保存其他节点使用的ID，而不是通过IP和端口。IP和端口可能会改变，但是唯一的node ID将不会改变直至该node的死亡。

我们现在已经启动了六个redis实例， 需要通过写一些有意义的配置信息到各个节点来创建集群。
redis cluster的命令行工具redis-trib，利用Ruby程序在实例上执行一些特殊的命令，很容易实现创建新的集群、检查或者reshard现有的集群等。

```bash
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```

`--replicas 1 `参数是将每个master带上一个slave。

### 配置JedisClusterConfig

```java
@Configuration
public class JedisClusterConfig {
    private static Logger logger = LoggerFactory.getLogger(JedisClusterConfig.class);

    @Value("${redis.cluster.nodes}")
    private String clusterNodes;

    @Value("${redis.cluster.timeout}")
    private int timeout;

    @Value("${redis.cluster.max-redirects}")
    private int redirects;
    
    @Autowired
    private JedisPoolConfig jedisPoolConfig;

    @Bean
    public RedisClusterConfiguration getClusterConfiguration() {
        Map<String, Object> source = new HashMap();

        source.put("spring.redis.cluster.nodes", clusterNodes);
        logger.info("clusterNodes: {}", clusterNodes);
        source.put("spring.redis.cluster.max-redirects", redirects);
        return new RedisClusterConfiguration(new MapPropertySource("RedisClusterConfiguration", source));
    }

    @Bean
    public JedisConnectionFactory getConnectionFactory() {
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(getClusterConfiguration());
        jedisConnectionFactory.setTimeout(timeout);
        return jedisConnectionFactory;
    }

    @Bean
    public JedisClusterConnection getJedisClusterConnection() {
        return (JedisClusterConnection) getConnectionFactory().getConnection();
    }

    @Bean
    public RedisTemplate getRedisTemplate() {
        RedisTemplate clusterTemplate = new RedisTemplate();
        clusterTemplate.setConnectionFactory(getConnectionFactory());
        clusterTemplate.setKeySerializer(new StringRedisSerializer());
        clusterTemplate.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        return clusterTemplate;

    }

}
```

可以配置密码，cluster对密码支持不太友好，如果对集群设置密码，那么requirepass和masterauth都需要设置，否则发生主从切换时，就会遇到授权问题。

### 配置redis cluster

```yaml
redis:
  cluster:
    enabled: true
    timeout: 2000
    max-redirects: 8
    nodes: 127.0.0.1:7000,127.0.0.1:7001
```
主要配置了redis cluster的节点、超时时间等。
### 使用RedisTemplate

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RedisConfigTest {
    @Autowired
    RedisTemplate redisTemplate;
    
    @Test
    public void clusterTest() {
      redisTemplate.opsForValue().set("foo", "bar");
      System.out.println(redisTemplate.opsForValue().get("foo"));
    }
}
```

用法很简单，注入RedisTemplate即可进行操作，RedisTemplate用法比较丰富，可以自行查阅。

## 总结

本文主要讲了redis集群的选型，主要有三种：客户端分片、基于代理的分片以及路由查询。对于前两种方式，分别进行简单地介绍，最后选择redis官方提供的redis cluster方案，并进行了实践。虽然正式版的推出时间不长，目前成功实践的案例也还不多，但是总体来说，redis cluster的整个设计是比较简单的，大部分操作都可以按照单点的操作流程进行操作。笔者使用的jedis客户端支持JedisCluster也是比较好，用起来也很方便。其实还有个压测的数据，后面再补上吧。


### 参考
1. [Redis集群方案应该怎么做？](https://www.zhihu.com/question/21419897)
2. [cluster-tutorial](https://redis.io/topics/cluster-tutorial)
3. [twemproxy](https://github.com/twitter/twemproxy)
4. [CodisLabs](https://github.com/CodisLabs/codis)

