# 3.3 Redis

> 作者：SunSAS
>
> **介绍：** SunSAS是SunSAS

## 3.3.1 redis 基础

### 1. 简介
Redis 是 C 语言开发的一个开源的（遵从 BSD 协议）高性能键值对（key-value）的**内存数据库**，可以用作数据库、缓存、消息中间件等。

它是一种 **NoSQL**（not-only sql，泛指非关系型数据库）的数据库。
Redis 作为一个内存数据库：
性能优秀，数据在内存中，读写速度非常快，支持并发 10W QPS。单进程单线程，是线程安全的，采用 IO 多路复用机制。丰富的数据类型，支持字符串（strings）、散列（hashes）、列表（lists）、集合（sets）、有序集合（sorted sets）等。支持数据持久化。可以将内存中数据保存在磁盘中，重启时加载。主从复制，哨兵，高可用。可以用作分布式锁。可以作为消息中间件使用，支持发布订阅。

### 2. redis数据类型
![数据](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/023b5bb5c9ea15ce598be46bdbb071f53b87b2a5.jpeg)

Redis 内部使用一个 `redisObject` 对象来表示所有的 key 和 value。

`redisObject` 最主要的信息如上图所示：type 表示一个 value 对象具体是何种数据类型，encoding 是不同数据类型在 Redis 内部的存储方式。

#### 2.1 String（字符串）
String 是 Redis 最基本的类型，可以理解成与 Memcached一模一样的类型，一个 Key 对应一个 Value。Value 不仅是 String，也可以是数字。  
String 类型是二进制安全的，意思是 Redis 的 String 类型可以包含任何数据，比如 jpg 图片或者序列化的对象。String 类型的值最大能存储 512M。
#### 2.2 Hash（字典）
Hash是一个键值（key-value）的集合。Redis 的 Hash 是一个 String 的 Key 和 Value 的映射表，Hash 特别适合存储对象。常用命令：hget，hset，hgetall 等。
#### 2.3 List（列表）
List 列表是简单的字符串列表，按照插入顺序排序。可以添加一个元素到列表的头部（左边）或者尾部（右边） 常用命令：lpush、rpush、lpop、rpop、lrange（获取列表片段）等。

> **应用场景**：List 应用场景非常多，也是 Redis 最重要的数据结构之一，比如 Twitter 的关注列表，粉丝列表都可以用 List 结构来实现。  
> **实现方式**:Redis List 的是实现是一个双向链表,可以用来当消息队列用。Redis 提供了 List 的 Push 和 Pop 操作，还提供了操作某一段的 API，可以直接查询或者删除某一段的元素。不过带来了额外的内存开销。

#### 2.4 Set（集合）
Set 是 String 类型的无序集合。集合是通过 hashtable 实现的。Set 中的元素是没有顺序的，而且是没有重复的。常用命令：sdd、spop、smembers、sunion 等。
> **应用场景**：Redis Set 对外提供的功能和 List 一样是一个列表，特殊之处在于 Set 是自动去重的，而且 Set 提供了判断某个成员是否在一个 Set 集合中。  
> 在微博应⽤中，可以将⼀个⽤户所有的关注⼈存在⼀个集合中，将其所有粉丝存在⼀个集合。
Redis可以⾮常⽅便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程，
具体命令如下:
    ```
        sinterstore key1 key2 key3 将交集存在key1内
    ```

#### 2.5 Zset（有序集合）
Zset(Sorted Set) 和 Set 一样是 String 类型元素的集合，且不允许重复的元素。常用命令：zadd、zrange、zrem、zcard 等。  
和 Set 相比，Sorted Set关联了一个 Double 类型权重的参数 Score，使得集合中的元素能够按照 **Score** 进行有序排列，Redis 正是通过分数来为集合中的成员进行从小到大的排序。
> **实现方式**：Redis Sorted Set 的内部使用 HashMap 和跳跃表（skipList）来保证数据的存储和有序，HashMap 里放的是成员到 Score 的映射。使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。  
>**应用场景**：在直播系统中，实时排⾏信息包含直播间在线⽤户列表，各种礼物排⾏榜，弹幕消息（可以理
解为按消息维度的消息排⾏榜）等信息，适合使⽤ Redis 中的 Sorted Set 结构进⾏存储。

**zset底层实现原理**：

有序集合对象的编码可以是ziplist或者skiplist。

当数据较少时，sorted set是由一个ziplist来实现的。  
当数据多的时候，sorted set是由一个dict + 一个skiplist来实现的。简单来讲，dict用来查询数据到分数的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）。

同时满足以下条件时使用ziplist编码：
- 元素数量小于128个
- 所有member的长度都小于64字节

![调表](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/skiplist_insertions.png)

从上面skiplist的创建和插入过程可以看出，每一个节点的层数（level）是随机出来的（算法得出），而且新插入一个节点不会影响其它节点的层数。因此，插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。这就降低了插入操作的复杂度。
具体参考 [redis zset底层实现原理](https://www.cnblogs.com/yuanfang0903/p/12165394.html)


类型 | 简介 | 特性 | 场景
---|---|---|---
String | 二进制安全 | 可以包含任何数据，比如 jpg 图片或者序列化的对象。| ...
Hash | 键值对集合，类似map | 适合存储对象，并且可以像数据库update操作只修改其中一个属性值 | 存储读取修改用户属性
List | 双向链表 | 增删快，提供了操作某一元素的api | 最新消息排行，消息队列
Set | hash表实现，元素不重复 | 添加删除查找的操作都是O（1），提供了求交集并集差集的操作 | 共同好友；利用唯一性，统计访问网站的所有ip
Zset | set元素添加了score，实现有序排列 | 数据插入集合时，已经进行了天然排序 | 排行榜；带权重的消息队列

### 3. 整合Spring Boot

#### 3.1 Spring Cache

当Spring Boot 结合Redis来作为缓存使用时，最简单的方式就是使用Spring Cache了，使用它我们无需知道Spring中对Redis的各种操作，仅仅通过它提供的`@Cacheable` 、`@CachePut` 、`@CacheEvict` 、`@EnableCaching`等注解就可以实现缓存功能。

**@EnableCaching**  
开启缓存功能，一般放在启动类上。

**@Cacheable**  
使用该注解的方法当缓存存在时，会从缓存中获取数据而不执行方法，当缓存不存在时，会执行方法并把返回结果存入缓存中。**一般使用在查询方法上**，可以设置如下属性：

- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- unless：条件符合则不缓存；
- condition：条件符合则缓存。

**@CachePut**  
使用该注解的方法每次执行时都会把返回结果存入缓存中。一般使用在**新增**方法上，可以设置如下属性：

- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- unless：条件符合则不缓存；
- condition：条件符合则缓存。
    
**@CacheEvict**  
使用该注解的方法执行时会清空指定的缓存。一般使用在**更新或删除**方法上，可以设置如下属性：

- value：缓存名称（必填），指定缓存的命名空间；
- key：用于设置在命名空间中的缓存key值，可以使用SpEL表达式定义；
- condition：条件符合则缓存。


##### Demo
添加依赖：

```java
<!--redis依赖配置-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
> 在 Spring Boot 2.x 以后底层不再使用 Jedis，而是换成了 Lettuce
>
> Jedis在实现上是直连Redis服务，多线程环境下非线程安全
>
> Lettuce是一种可伸缩，线程安全，完全非阻塞的Redis客户端，多个线程可以共享一个RedisConnection，它利用Netty NIO框架来高效地管理多个连接，从而提供了异步和同步数据访问方式，用于构建非阻塞的反应性应用程序。

如果要使用连接池：需要引入依赖：

```
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

配置redis：

```
spring:
  redis:
    host: 127.0.0.1 # Redis服务器地址
    database: 0 # Redis数据库索引（默认为0）
    port: 6379 # Redis服务器连接端口
    password: # Redis服务器连接密码（默认为空）
    timeout: 1000ms # 连接超时时间
    lettuce: # 连接池的配置，不用则去除掉
      pool:
        max-active: 8 # 连接池最大连接数
        max-idle: 8 # 连接池最大空闲连接数
        min-idle: 0 # 连接池最小空闲连接数
        max-wait: -1ms # 连接池最大阻塞等待时间，负值表示没有限制
```
在启动类上添加@EnableCaching注解启动缓存功能：

```java
@EnableCaching
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```
使用缓存：

```java
@CacheEvict(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id")
@Override
public int update(Long id, PmsBrand brand) {
    brand.setId(id);
    return brandMapper.updateByPrimaryKeySelective(brand);
}

@CacheEvict(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id")
@Override
public int delete(Long id) {
    return brandMapper.deleteByPrimaryKey(id);
}

@Cacheable(value = RedisConfig.REDIS_KEY_DATABASE, key = "'pms:brand:'+#id", unless = "#result==null")
@Override
public PmsBrand getItem(Long id) {
    return brandMapper.selectByPrimaryKey(id);
}

```

#### 3.2 RedisTemplate
Spring Cache 给我们提供了操作Redis缓存的便捷方法，但是也有很多局限性。接下来我们来讲下如何通过RedisTemplate来自由操作Redis中的缓存。


RedisServiceImpl：使用RedisTemplate来自由操作Redis中的缓存数据(部分常用)。这种api随便搜搜就好了，
给一个参考[Java 操作Redis封装RedisTemplate工具类](https://www.cnblogs.com/smartsmile/p/11633844.html)
```java

/**
 * redis操作Service的实现类
 */
@Service
public class RedisServiceImpl implements RedisService {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public void set(String key, String value) {
        stringRedisTemplate.opsForValue().set(key, value);
    }

    @Override
    public String get(String key) {
        return stringRedisTemplate.opsForValue().get(key);
    }

    @Override
    public boolean expire(String key, long expire) {
        return stringRedisTemplate.expire(key, expire, TimeUnit.SECONDS);
    }

    @Override
    public void remove(String key) {
        stringRedisTemplate.delete(key);
    }

    @Override
    public Long increment(String key, long delta) {
        return stringRedisTemplate.opsForValue().increment(key,delta);
    }
}
```



> 参考
>
> [Spring Data Redis 最佳实践！](https://mp.weixin.qq.com/s/9j3exBtZ9FWWlyZkxlWaOA)




---




## 3.3.2 redis高级
基础掌握基本够日常的使用，但是面试可一般不会问的这么浅。

### 1.为什么要⽤缓存？
主要从“⾼性能”和“⾼并发”这两点来看待这个问题。

**⾼性能**：
假如⽤户第⼀次访问数据库中的某些数据。这个过程会⽐᫾慢，因为是从硬盘上读取的。将该⽤户访问
的数据存在缓存中，这样下⼀次再访问这些数据的时候就可以直接从缓存中获取了。操作缓存就是直接
操作内存，所以速度相当快。如果数据库中的对应数据改变的之后，同步改变缓存中相应的数据即可！

**⾼并发**：
直接操作缓存能够承受的请求是远远⼤于直接访问数据库的，所以我们可以考虑把数据库中的部分数据
转移到缓存中去，这样⽤户的⼀部分请求会直接到缓存这⾥⽽不⽤经过数据库。

### 2.为什么要⽤ redis ⽽不⽤ map/guava 做缓存?
- Redis 可以用几十 G 内存来做缓存,Map 不行,一般 JVM 也就分几个 G 数据就够大了
- Redis 的缓存可以持久化,Map 是内存对象,程序一重启数据就没了
- Redis 可以实现分布式的缓存,Map 只能存在创建它的程序里
- Redis 可以处理每秒百万级的并发,是专业的缓存服务,Map 只是一个普通的对象
- Redis 缓存有过期机制,Map 本身无此功能
- Redis 有丰富的 API,Map 就简单太多了


### 3. Redis为什么这么快？
Redis采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写，官方提供的数据是可以达到**100000**+的QPS（每秒内查询次数）。这个数据不比采用单进程多线程的同样基于内存的 KV 数据库 Memcached 差
1. 完全基于**内存**，绝大部分请求是纯粹的内存操作，非常快速。数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
2. **数据结构简单**，对数据操作也简单，Redis中的数据结构是专门进行设计的；
3. **采用单线程**，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
4. 使用**多路I/O复用模型**，非阻塞IO；
5. 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

> 多路I/O复用模型是利用 select、poll、epoll 可以同时监察多个流的 I/O 事件的能力，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有 I/O 事件时，就从阻塞态中唤醒，于是程序就会轮询一遍所有的流（epoll 是只轮询那些真正发出了事件的流），并且只依次顺序的处理就绪的流，这种做法就避免了大量的无用操作。
>
> 这里“多路”指的是**多个网络连接**，“复用”指的是**复用同一个线程**。采用多路 I/O 复用技术可以让单个线程高效的处理多个连接请求（尽量减少网络 IO 的时间消耗），且 Redis 在内存中操作数据的速度非常快，也就是说内存内的操作不会成为影响Redis性能的瓶颈，主要由以上几点造就了 Redis 具有很高的吞吐量。

### 4. 为什么Redis是单线程的?
Redis 在处理客户端的请求时，包括获取（Socket 读）、解析、执行、内容返回（Socket 写）等都由一个顺序串行的主线程处理，这就是所谓的“单线程”。  
**注意redis6.0支持多线程！**

但如果严格来讲从 Redis 4.0 之后并不是单线程，除了主线程外，它也有后台线程在处理一些较为缓慢的操作，例如清理脏数据、无用连接的释放、大 Key 的删除等等。

官方曾做过类似问题的回复：使用 Redis 时，几乎不存在 CPU 成为瓶颈的情况， Redis 主要受限于内存和网络。

例如在一个普通的 Linux 系统上，Redis 通过使用 Pipelining 每秒可以处理 100 万个请求，所以如果应用程序主要使用 O(N) 或 O(log(N)) 的命令，它几乎不会占用太多 CPU。

使用了单线程后，可维护性高。多线程模型虽然在某些方面表现优异，但是它却引入了程序执行顺序的不确定性，带来了并发读写的一系列问题，增加了系统复杂度、同时可能存在线程切换、甚至加锁解锁、死锁造成的性能损耗。

Redis 通过 AE 事件模型以及 IO 多路复用等技术，处理性能非常高，因此没有必要使用多线程。

单线程机制使得 Redis 内部实现的复杂度大大降低，Hash 的惰性 Rehash、Lpush 等等 “线程不安全” 的命令都可以无锁进行。

**Redis的瓶颈不是cpu的运行速度，而往往是网络带宽和机器的内存大小**。再说了，单线程切换开销小，容易实现既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了。
> 如果无法发挥多核CPU 性能，可以通过在单机开多个Redis 实例来完善。
>
> 不过**redis6.0开始支持多线程**了，参考[支持多线程的Redis 6.0终于发布了](https://mp.weixin.qq.com/s/_MWT5W8OjhdIrX6DS9rQcA)

> **单线程可以处理高并发请求吗？**
> 
> 当然可以了，Redis都实现了。有一点概念需要澄清，并发并不是并行。
    （相关概念：并发性I/O流，意味着能够让一个计算单元来处理来自多个客户端的流请求。并行性，意味着服务器能够同时执行几个事情，具有多个计算单元）

### 5. redis 的线程模型
redis 内部使⽤⽂件事件处理器 `file event handler` ，这个⽂件事件处理器是单线程的，所以
redis 才叫做单线程的模型。它采⽤ IO 多路复⽤机制同时监听多个 socket，根据 socket 上的事件
来选择对应的事件处理器进⾏处理。
⽂件事件处理器的结构包含 4 个部分：
- 多个 socket
- IO 多路复⽤程序
- ⽂件事件分派器
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个 socket 可能会并发产⽣不同的操作，每个操作对应不同的⽂件事件，但是 IO 多路复⽤程序会监
听多个 socket，会将 socket 产⽣的事件放⼊队列中排队，事件分派器每次从队列中取出⼀个事件，
把该事件交给对应的事件处理器进⾏处理。


### 6. redis 和 memcached 的区别
对于 redis 和 memcached 我总结了下⾯四点。现在公司⼀般都是⽤ redis 来实现缓存，⽽且 redis
⾃身也越来越强⼤了！
1. redis**⽀持更丰富的数据类型**（⽀持更复杂的应⽤场景）：Redis不仅仅⽀持简单的k/v类型的数
据，同时还提供list，set，zset，hash等数据结构的存储。memcache⽀持简单的数据类型，
String。
2. Redis**⽀持数据的持久化**，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进⾏使
⽤,⽽Memecache把数据全部存在内存之中。
3. **集群模式**：memcached没有原⽣的集群模式，需要依靠客户端来实现往集群中分⽚写⼊数据；但
是 redis ⽬前是原⽣⽀持 cluster 模式的.
4. Memcached是多线程，⾮阻塞IO复⽤的⽹络模型；Redis使⽤单线程的**多路 IO 复⽤模型**。

![对比](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/QQ%E5%9B%BE%E7%89%8720200727153855.png)

### 7. redis 设置过期时间
Redis中有个设置时间过期的功能，即对存储在 redis 数据库中的值可以设置⼀个过期时间。作为⼀个
缓存数据库，这是⾮常实⽤的。如我们⼀般项⽬中的 token 或者⼀些登录信息，尤其是短信验证码都
是有时间限制的，按照传统的数据库处理⽅式，⼀般都是⾃⼰判断过期，这样⽆疑会严重影响项⽬性能。  

我们 set key 的时候，都可以给⼀个 expire time，就是过期时间，通过过期时间我们可以指定这个
key 可以存活的时间。 

如果假设你设置了⼀批 key 只能存活1个⼩时，那么接下来1⼩时后，redis是怎么对这批key进⾏删除
的？  
**定期删除+惰性删除**。

- **定期删除**：redis默认是每隔 100ms 就**随机抽取**⼀些设置了过期时间的key，检查其是否过期，
如果过期就删除。注意这⾥是随机抽取的。为什么要随机呢？你想⼀想假如 redis 存了⼏⼗万
个 key ，每隔100ms就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很⼤的负载！
- **惰性删除** ：定期删除可能会导致很多过期 key 到了时间并没有被删除掉。所以就有了惰性删
除。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存⾥，除⾮你的系统去查⼀下那
个 key，才会被redis给删除掉。这就是所谓的惰性删除。

但是仅仅通过设置过期时间还是有问题的。我们想⼀下：如果定期删除漏掉了很多过期 key，然后你也
没及时去查，也就没⾛惰性删除，此时会怎么样？如果⼤量过期key堆积在内存⾥，导致redis内存块耗
尽了。怎么解决这个问题呢？ **redis 内存淘汰机制**。

### 8. redis 内存淘汰机制
redis 提供 6种数据淘汰策略：
1. **volatile-lru**：从已设置过期时间的数据集（server.db[i].expires）中挑选最久没有使用的数
据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘
汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru**：当内存不⾜以容纳新写⼊数据时，在键空间中，移除最近最少使⽤的key（由于存expire需要额外内存，所以一般不设置过期时间，这个是
最常⽤的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁⽌驱逐数据，也就是说当内存不⾜以容纳新写⼊数据时，新写⼊操作会报错。
这个应该没⼈使⽤吧！

4.0版本后增加以下两种：

7. **volatile-lfu**：从已设置过期时间的数据集(server.db[i].expires)中挑选使用频率最少的数据
淘汰
8. **allkeys-lfu**：当内存不⾜以容纳新写⼊数据时，在键空间中，移除使用频率最少的的key

> 1. 版本不同，默认的策略不同  
在2.8.13的版本里，默认是noeviction，在3.2.3版本里默认是volatile-lru
> 2. volatile这几种只对设置了过期时间的key有效，不会淘汰没有设置过期时间的key
> 3. 与是否设置持久化无关
> 4. 测试volatile-lru测量，在插入有过期时间的key时，如果达到了maxmemory值，就会对老的key进行淘汰，插入新值
> 5. 如果设置为no-eviction或者key没有过期时间，当达到最大内存值时就会直接报错

想要深入理解，参考[Redis系列--内存淘汰机制（含单机版内存优化建议）](https://www.cnblogs.com/alsf/p/9399009.html)

### 9.  redis 持久化机制
很多时候我们需要持久化数据也就是将内存中的数据写⼊到硬盘⾥⾯，⼤部分原因是为了之后重⽤数据
（⽐如重启机器、机器故障之后恢复数据），或者是为了防⽌系统故障⽽将数据备份到⼀个远程位置。

Redis不同于Memcached的很重⼀点就是，**Redis⽀持持久化**，⽽且⽀持两种不同的持久化操作。Redis的
⼀种持久化⽅式叫**快照**（snapshotting，RDB），另⼀种⽅式是**只追加⽂件**（append-only
file,AOF）。

#### 9.1 RDB
Redis可以通过创建快照来获得存储在内存⾥⾯的数据在某个时间点上的副本。Redis创建快照之后，可
以对快照进⾏备份，可以将快照复制到其他服务器从⽽创建具有相同数据的服务器副本（Redis主从结
构，主要⽤来提⾼Redis性能），还可以将快照留在原地以便重启服务器的时候使⽤。

**快照持久化是Redis默认采⽤的持久化⽅式**，在redis.conf配置⽂件中默认有此下配置：
```
save 900 1 #在900秒(15分钟)之后，如果⾄少有1个key发⽣变化，
Redis就会⾃动触发BGSAVE命令创建快照。
save 300 10 #在300秒(5分钟)之后，如果⾄少有10个key发⽣变化，
Redis就会⾃动触发BGSAVE命令创建快照。
save 60 10000 #在60秒(1分钟)之后，如果⾄少有10000个key发⽣变化，
Redis就会⾃动触发BGSAVE命令创建快照。
```
##### 优点
1. RDB文件紧凑，全量备份，非常适合用于进行备份和灾难恢复。
2. 生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作。
3. RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

##### 缺点
RDB快照是一次全量备份，存储的是内存数据的二进制序列化形式，存储上非常紧凑。当进行快照持久化时，会开启一个子进程专门负责快照持久化，子进程会拥有父进程的内存数据，父进程修改内存子进程不会反应出来，所以在快照持久化期间修改的数据不会被保存，可能丢失数据。

#### 9.2 AOF
与快照持久化相⽐，AOF持久化 的实时性更好，因此已成为主流的持久化⽅案。默认情况下Redis没有
开启AOF（append only file）⽅式的持久化，可以通过appendonly参数开启：
```
appendonly yes
```
开启AOF持久化后每执⾏⼀条会更改Redis中的数据的命令，Redis就会将该命令写⼊硬盘中的AOF⽂件。
AOF⽂件的保存位置和RDB⽂件的位置相同，都是通过dir参数设置的，默认的⽂件名是
`appendonly.aof`。  
在Redis的配置⽂件中存在三种不同的 AOF 持久化⽅式，它们分别是：
```
#每次有数据修改发⽣时都会写⼊AOF⽂件,这样会严重降低Redis的速度
appendfsync always 
#每秒钟同步⼀次，显示地将多个写命令同步到硬盘
appendfsync everysec 
#让操作系统决定何时进⾏同步
appendfsync no 
```

命令 | always | everysec | no
---|---|---|---
优点 | 不丢失数据 | 每秒一次fsync，性能影响小 | 不用管
缺点 | IO开销较大，一般的sata盘只有几百TPS | 会丢失一秒数据 | 不可控


为了兼顾数据和写⼊性能，⽤户可以考虑 `appendfsync everysec`选项 ，让Redis每秒同步⼀次AOF⽂
件，Redis性能⼏乎没受到任何影响。⽽且这样即使出现系统崩溃，⽤户最多只会**丢失⼀秒之内**产⽣的
数据。当硬盘忙于执⾏写⼊操作的时候，Redis还会优雅的放慢⾃⼰的速度以便适应硬盘的最⼤写⼊速
度。

##### 优点
1. AOF可以更好的保护数据不丢失，一般AOF会每隔1秒，通过一个后台线程执行一次fsync操作，最多丢失1秒钟的数据。
2. AOF日志文件没有任何磁盘寻址的开销，写入性能非常高，文件不容易破损。
3. AOF日志文件即使过大的时候，出现**后台重写**操作，也不会影响客户端的读写。
4. AOF日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用flushall命令清空了所有数据，只要这个时候后台rewrite还没有发生，那么就可以立即拷贝AOF文件，将最后一条flushall命令给删了，然后再将该AOF文件放回去，就可以通过恢复机制，自动恢复所有数据

##### 缺点
1. 对于同一份数据来说，AOF日志文件通常比RDB数据快照文件更大
2. AOF开启后，支持的写QPS会比RDB支持的写QPS低，因为AOF一般会配置成每秒fsync一次日志文件，当然，每秒一次fsync，性能也还是很高的
3. 以前AOF发生过bug，就是通过AOF记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。

> **AOF 重写**  
>
> AOF重写可以产⽣⼀个新的AOF⽂件，这个新的AOF⽂件和原有的AOF⽂件所保存的数据库状态⼀样，但体
积更⼩。  
AOF重写是⼀个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序⽆须对现有AOF⽂件
进⾏任何读⼊、分析或者写⼊操作。  
在执⾏ BGREWRITEAOF 命令时，Redis 服务器会维护⼀个 AOF 重写缓冲区，该缓冲区会在⼦进程创建
新AOF⽂件期间，记录服务器执⾏的所有写命令。当⼦进程完成创建新AOF⽂件的⼯作之后，服务器会将
重写缓冲区中的所有内容追加到新AOF⽂件的末尾，使得新旧两个AOF⽂件所保存的数据库状态⼀致。最
后，服务器⽤新的AOF⽂件替换旧的AOF⽂件，以此来完成AOF⽂件重写操作

#### 9.3  RDB 和 AOF 的混合持久化
Redis 4.0 开始⽀持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 aof-use-rdb-preamble 开启）。  
如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF ⽂件开头。这样做的好处是可
以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF ⾥⾯的
RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

### 10. redis 事务
Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务(transaction)功能。事务提供了⼀种将多个命令
请求打包，然后⼀次性、按顺序地执⾏多个命令的机制，并且在事务执⾏期间，服务器不会中断事务⽽
改去执⾏其他客户端的命令请求，它会将事务中的所有命令都执⾏完毕，然后才去处理其他客户端的命
令请求。
在传统的关系式数据库中，常常⽤ ACID 性质来检验事务功能的可靠性和安全性。在 Redis 中，事务
总是具有原⼦性（Atomicity）、⼀致性（Consistency）和隔离性（Isolation），并且当 Redis 运⾏
在某种特定的持久化模式下时，事务也具有持久性（Durability）。
> redis同⼀个事务中如果有⼀条命令执⾏失败，其后的命令仍然会被执⾏，没有回滚。

### 11. 缓存雪崩
缓存同一时间大面积的失效，所以，后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉。

举个例子：目前电商首页以及热点数据都会去做缓存，一般缓存都是定时任务去刷新，或者查不到之后去更新缓存的，定时任务刷新就有一个问题。  
如果首页所有 Key 的失效时间都是 12 小时，中午 12 点刷新的，我零点有个大促活动大量用户涌入，假设每秒 6000 个请求，本来缓存可以抗住每秒 5000 个请求，但是缓存中所有 Key 都失效了。  
此时 6000 个/秒的请求全部落在了数据库上，数据库必然扛不住，真实情况可能 DBA 都没反应过来直接挂了。
> ⼀般 3000 个并发请求就能打死⼤部
分数据库了。

#### 解决办法
- 在批量往 Redis 存数据的时候，把每个 Key 的失效时间都加个随机值就好了，这样可以保证数据不会再同一时间大面积失效。
    ```
    setRedis（key, value, time+Math.random()*10000）;
    ```
- 设置热点数据永不过期,有更新操作就同步更新缓存就好了
- 如果 Redis 是集群部署，将热点数据均匀分布在不同的 Redis 库中也能避免全部失效。
- 事前：尽量保证整个 redis 集群的⾼可⽤性，发现机器宕机尽快补上。选择合适的内存淘汰策
略。
- 事中：本地ehcache缓存 + hystrix限流&降级，避免MySQL崩掉
- 事后：利⽤ redis 持久化机制保存的数据尽快恢复缓存
![缓存雪崩](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/6078367.png)

### 12. 缓存穿透
缓存穿透说简单点就是⼤量请求的 key 根本不存在于缓存中，导致请求直接到了数据库上，根本没有
经过缓存这⼀层。举个例⼦：某个⿊客故意制造我们缓存中**不存在的** key 发起⼤量请求，导致⼤量请
求落到数据库。（如果存在那么就直接缓存了）

#### 解决办法
1. ⾸先**做好参数校验**，⼀些不合法的参数请求直接抛出异常信息返回给客户端。⽐如查询的
数据库 id 不能⼩于 0、传⼊的邮箱格式不对的时候直接返回错误消息给客户端等等。
2. **缓存⽆效 key** : 如果缓存和数据库都查不到某个 key 的数据就写⼀个到 redis 中去并设置过期时
间，具体命令如下： SET key value EX 10086 。这种⽅式可以解决请求的 key 变化不频繁的情
况，如果⿊客恶意攻击，每次构建不同的请求key，会导致 redis 中缓存⼤量⽆效的 key 。很明显，
这种⽅案并不能从根本上解决此问题。如果⾮要⽤这种⽅式来解决穿透问题的话，尽量将⽆效的 key
的过期时间设置短⼀点⽐如 1 分钟。  
⼀般情况下我们是这样设计 key 的： 表名:列名:主键名:主键值 。
    ```java
    public Object getObjectInclNullById(Integer id) {
        // 从缓存中获取数据
        Object cacheValue = cache.get(id);
        // 缓存为空
        if (cacheValue == null) {
            // 从数据库中获取
            Object storageValue = storage.get(key);
            // 缓存空对象
            cache.set(key, storageValue);
            // 如果存储数据为空，需要设置一个过期时间(300秒)
            if (storageValue == null) {
                // 必须设置过期时间，否则有被攻击的风险
                cache.expire(key, 60 * 5);
            }
            return storageValue;
        }
        return cacheValue;
    }
    ```

3. **布隆过滤器**  
布隆过滤器是⼀个⾮常神奇的数据结构，通过它我们可以⾮常⽅便地判断⼀个给定数
据是否存在与海量数据中。我们需要的就是判断 key 是否合法。  
    当一个元素加入布隆过滤器中的时候，会进行如下操作：
    
    - 使用布隆过滤器中的哈希函数对元素值进行计算，得到哈希值（有几个哈希函数得到几个哈希值）。
    - 根据得到的哈希值，在位数组中把对应下标的值置为 1。
    
    当我们需要判断一个元素是否存在于布隆过滤器的时候，会进行如下操作：
    
    - 对给定元素再次进行相同的哈希计算；
    - 得到值之后判断位数组中的每个元素是否都为 1，如果值都为 1，那么说明这个值在布隆过滤器中，如果存在一个值不为 1，说明该元素不在布隆过滤器中。  

具体原理参考[详解布隆过滤器的原理、使用场景和注意事项](https://www.jianshu.com/p/2104d11ee0a2)

如果我们需要体验 Redis 中的布隆过滤器非常简单，通过 Docker 就可以了：

```
➜  ~ docker run -p 6379:6379 --name redis-redisbloom redislabs/rebloom:latest
➜  ~ docker exec -it redis-redisbloom bash
root@21396d02c252:/data# redis-cli
127.0.0.1:6379> 
```
常用命令
```
BF.ADD ：将元素添加到布隆过滤器中，如果该过滤器尚不存在，则创建该过滤器。格式：BF.ADD {key} {item}。
BF.ADD myFilter java
BF.MADD : 将一个或多个元素添加到“布隆过滤器”中，并创建一个尚不存在的过滤器。该命令的操作方式BF.ADD与之相同，只不过它允许多个输入并返回多个值。格式：BF.MADD {key} {item} [item ...] 。
**BF.EXISTS ** : 确定元素是否在布隆过滤器中存在。格式：BF.EXISTS {key} {item}。
BF.MEXISTS ： 确定一个或者多个元素是否在布隆过滤器中存在格式：BF.MEXISTS {key} {item} [item ...]。

```



### 13. 缓存击穿
跟缓存雪崩有点像，但是又有一点不一样，缓存雪崩是因为大面积的缓存失效，打崩了 DB。

而缓存击穿不同的是缓存击穿是指一个 Key 非常热点，在不停地扛着大量的请求，大并发集中对这一个点进行访问，当这个 Key 在失效的瞬间，持续的大并发直接落到了数据库上，就在这个 Key 的点上击穿了缓存。

#### 解决办法
- 设置热点数据永不过期，
- 加上互斥锁。
    ```
    public static String getData(String key)throws InterruptedException {
        //从Redis查询数据 
        String result = getDataByKV(key);
        //参数校验
        if (StringUtils.isBlank(result)) {
            try {
                //获得锁
                if (reenLock.tryLock()) {
                    //去数据库查询 
                    result = getDataByDB(key);
                    //校验
                    if (StringUtils.isNotBlank(result)) {
                        //插进缓存 
                        setDataToKV(key, result); 
                    } 
                } else {
                    //睡一会再拿 
                    Thread.sleep(100L); 
                    result = getData(key); 
                } 
            } finally {
                //释放锁 
                reenLock.unlock(); 
            } 
        }
        return result; 
    }
    ```
缓存中有数据，直接返回。  
没有数据时，第一个进入的线程，获取锁去数据库取数据，没释放锁时，其它线程等待100ms，再重新去缓存取数据。防止都去数据库请求数据，重复向缓存更新数据。

### 14. 双写一致性
参考[如何保证缓存与数据库的双写一致性？](https://www.jianshu.com/p/2936a5c65e6b)

最经典的缓存+数据库读写的模式，就是 Cache Aside Pattern。

- 读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。
- 更新的时候，先更新数据库，然后再删除缓存。

关于更新，也有先让缓存失效，再更新数据库这种，不管哪种，在高并发下都可能导致双写不一致，只不过后者出现的case几率更小。

⼀般来说，就是如果你的系统不是严格要求缓存+数据库必须⼀致性的话，缓存可以稍微的跟数据库偶
尔有不⼀致的情况，最好不要做这个⽅案**：读请求和写请求串⾏化**，串到⼀个内存队列⾥去，这样就可
以保证⼀定不会出现不⼀致的情况。

串⾏化之后，就会导致系统的吞吐量会⼤幅度的降低，⽤⽐正常情况下多⼏倍的机器去⽀撑线上的⼀个
请求。


### 15. 主从复制及哨兵机制
此部分属于运维知识，了解即可。

Redis 单节点存在单点故障问题，为了解决单点问题，一般都需要对 Redis 配置从节点，然后使用哨兵来监听主节点的存活状态，如果主节点挂掉，从节点能继续提供缓存功能。  
从节点仅提供读操作，主节点提供写操作。对于读多写少的状况，可给主节点配置多个从节点，从而提高响应效率。

- 为数据提供多个副本，实现高可用
- 实现读写分离（主节点负责写数据，从节点负责读数据，主节点定期把数据同步到从节点保证数据的一致性）


#### 15.1 主从复制的方式
主从刚刚连接的时候，进行全量同步；全同步结束后，进行增量同步。当然，如果有需要，slave 在任何时候都可以发起全量同步。redis 策略是，无论如何，首先会尝试进行增量同步，如不成功，要求从机进行全量同步。
##### 15.1.2 全量复制
用于初次复制或其它无法进行部分复制的情况，将主节点中的所有数据都发送给从节点，是一个非常重型的操作，当数据量较大时，会对主从节点和网络造成很大的开销:
- 主节点需要bgsave
- RDB文件网络传输占用网络io
- 从节点要清空数据
- 从节点加载RDB
- 全量复制会触发从节点AOF重写

![全量复制](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/7368936-365044ea1c884c96.png)
**全量复制过程：**
1. Redis内部会发出一个同步命令，刚开始是Psync命令，Psync ? -1表示要求master主机同步数据
2. 主机会向从机发送run_id和offset，因为slave并没有对应的 offset，所以是全量复制
3. 从机slave会保存主机master的基本信息
4. 主节点收到全量复制的命令后，执行bgsave（异步执行），在后台生成RDB文件（快照），并使用一个缓冲区（称为复制缓冲区）记录从现在开始执行的所有写命令
5. 主机发送RDB文件给从机
6. 发送缓冲区数据
7. 刷新旧的数据。从节点在载入主节点的数据之前要先将老数据清除
8. 加载RDB文件将数据库状态更新至主节点执行bgsave时的数据库状态和缓冲区数据的加载。

##### 15.1.2 部分复制
部分复制是Redis 2.8以后出现的，用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当从节点再次连上主节点后，如果条件允许，主节点会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销，需要注意的是，如果网络中断时间过长，造成主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制.

![部分复制](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/7368936-ee71cb2a90e7f4b3.png)
**部分复制过程：**

1. 如果网络抖动（连接断开 connection lost）
2. 主机master 还是会写 repl_back_buffer（复制缓冲区）
3. 从机slave 会继续尝试连接主机
4. 从机slave 会把自己当前 run_id 和偏移量传输给主机 master，并且执行 pysnc 命令同步
5. 如果master发现你的偏移量是在缓冲区的范围内，就会返回 continue命令
6. 同步了offset的部分数据，所以部分复制的基础就是偏移量 offset。

#### 15.2 哨兵机制
一旦主节点宕机，从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令所有从节点去复制新的主节点，整个过程需要人工干预。主节点的写能力受到单机的限制。主节点的存储能力受到单机的限制。原生复制的弊端在早期的版本中也会比较突出，比如：Redis 复制中断后，从节点会发起 psync。此时如果同步不成功，则会进行全量同步，主库执行全量备份的同时，可能会造成毫秒或秒级的卡顿。

这种故障转移的方式对于很多应用场景是不能容忍的。正式由于这个问题，Redis 提供了 **Sentinel**(哨兵) 架构来解决这个问题。

![Redis Sentinel（哨兵）的架构](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/a2cc7cd98d1001e963a8484cd5be30ea55e79717.jpeg)
Redis Sentinel（哨兵）主要功能包括主节点存活检测、主从运行情况检测、自动故障转移、主从切换。

Redis Sentinel 最小配置是一主一从。Redis 的 Sentinel 系统可以用来管理多个 Redis 服务器。

该系统可以执行以下四个任务：

- **监控**：不断检查主服务器和从服务器是否正常运行。
- **通知**：当被监控的某个 Redis 服务器出现问题，Sentinel 通过 API 脚本向管理员或者其他应用程序发出通知。
- **自动故障转移**：当主节点不能正常工作时，Sentinel 会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点，并且将其他的从节点指向新的主节点，这样人工干预就可以免了。
- **配置提供者**：在 Redis Sentinel 模式下，客户端应用在初始化时连接的是 Sentinel 节点集合，从中获取主节点的信息。

##### 哨兵的工作原理
![哨兵的工作原理](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/4e4a20a4462309f7774c05db1fbe47f5d6cad60b.jpeg)
1. 每个 Sentinel 节点都需要定期执行以下任务：每个 Sentinel 以每秒一次的频率，向它所知的主服务器、从服务器以及其他的 Sentinel 实例发送一个 PING 命令。（如上图）
2. 如果一个实例距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 所指定的值，那么这个实例会被 Sentinel 标记为**主观下线**。
3. 如果一个主服务器被标记为主观下线，那么正在监视这个服务器的所有 Sentinel 节点，要以每秒一次的频率确认主服务器的确进入了主观下线状态。
4. 如果一个主服务器被标记为主观下线，并且有足够数量的 Sentinel（至少要达到配置文件指定的数量）在指定的时间范围内同意这一判断，那么这个主服务器被标记为**客观下线**。
5. 一般情况下，每个 Sentinel 会以每 10 秒一次的频率向它已知的所有主服务器和从服务器发送 INFO 命令。**当一个主服务器被标记为客观下线时**，Sentinel 向下线主服务器的所有从服务器发送 INFO 命令的频率，会从 10 秒一次改为每秒一次。
6. Sentinel 和其他 Sentinel 协商客观下线的主节点的状态，如果处于 SDOWN 状态，则投票自动选出新的主节点，将剩余从节点指向新的主节点进行数据复制。
7. 当没有足够数量的 Sentinel 同意主服务器下线时，主服务器的客观下线状态就会被移除。  
当主服务器重新向 Sentinel 的 PING 命令返回有效回复时，主服务器的主观下线状态就会被移除。

### 16. Redis-Cluster
edis最开始使用主从模式做集群，若master宕机需要手动配置slave转为master；后来为了高可用提出来哨兵模式，该模式下有一个哨兵监视master和slave，若master宕机可自动将slave转为master，但它也有一个问题，就是不能动态扩充；所以在3.x提出cluster集群模式。

![Redis-Cluster架构](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/12185313-0f55e1cc574cae70.png)

Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接

其结构特点：
1. 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
2. 节点的fail是通过集群中超过半数的节点检测失效时才生效。
3. 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。
4. redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配）,cluster 负责维护node<->slot<->value。
5. Redis集群预分好16384个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

**缺陷：**

- **不能自动发现**：无Auto Discovery功能。集群建立时以及运行中新增结点时，都要通过手动执行MEET命令或redis-trib.rb脚本添加到集群中
- **不能自动Resharding**：不仅不自动，连Resharding算法都没有，要自己计算从哪些结点上迁移多少Slot，然后还是得通过redis-trib.rb操作
- **严重依赖外部redis-trib**：如上所述，像集群健康状况检查、结点加入、Resharding等等功能全都抽离到一个Ruby脚本中了。还不清楚上面提到的缺失功能未来是要继续加到这个脚本里还是会集成到集群结点中？redis-trib也许要变成Codis中Dashboard的角色
- **无监控管理UI**：即便未来加了UI，像迁移进度这种信息在无中心化设计中很难得到
- **只保证最终一致性**：写Master成功后立即返回，如需强一致性，自行通过WAIT命令实现。但对于“脑裂”问题，目前Redis没提供网络恢复后的Merge功能，“脑裂”期间的更新可能丢失

参考[全面剖析Redis Cluster原理和应用](https://www.cnblogs.com/xiaomaohai/p/6157597.html)

### 17 Redis分布式锁

首先说结论，单体Redis加锁，解锁：

```linux
 # 加锁
 SET resource_name my_random_value NX PX 30000
 # 解锁（以下通过Lua脚本实现）
 if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
 else
    return 0
 end
```

其实官方给的示例就是如此。

> [Redis官方](http://redis.cn/topics/distlock.html)

#### 加锁

首先我们需要满足最基本的需求，锁互斥，我拿到了锁，你就不能拿到。可以使用`SETNX key value`命令

这个命令来自于`SET if Not eXists`的缩写，意思是：如果 `key` 不存在，则设置 `value` 给这个`key`，否则啥都不做。返回1则设置成功，0表示不成功

#### 解锁

很简单，使用 `DEL` 删除这个 `key` 就行。

但是存在异常情况无法执行这个 DEL 命令，比如程序异常了，客户端挂了，那么这个锁就一直无法释放了，所以我们需要加入过期时间。

#### 超时设置锁

加上设置过期时间命令：

```mysql
> SETNX lock:168 1  // 获取锁
(integer) 1
> EXPIRE lock:168 60 // 设置超时时间 60s
```

但是这样存在一个问题，这两个操作不是原子操作，所以可能还是没法设置超时时间。Redis 2.6.X 之后，官方拓展了 `SET` 命令的参数，满足了当 key 不存在则设置 value，同时设置超时时间的语义，并且满足原子性。

```undefined
SET resource_name random_value NX PX 30000
```

- NX：表示只有 `resource_name` 不存在的时候才能 `SET` 成功，从而保证只有一个客户端可以获得锁；
- PX 30000：表示这个锁有一个 30 秒自动过期时间

释放了不是自己的锁

使用了超时锁后，假如过期时间不够，线程1还没执行完，但锁过期被释放，然后线程2获得锁，此时线程1调用释放锁命令，把线程2的拿到的锁释放了，这就释放了不是自己锁。

#### 加标识

这个标识就是客户端的标识，直接存在 `value` 中即可。在解锁时判断取出来的值与客户端标识是否一致，如果一致再解锁，伪代码如下：

```csharp
// 比对 value 与 唯一标识
if (redis.get("lock:168").equals(random_value)){
   redis.del("lock:168"); //比对成功则删除
 }
```

但这个操作会存在原子性问题，所以解锁使用 lua 命令：

```kotlin
// 获取锁的 value 与 ARGV[1] 是否匹配，匹配则执行 del
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

> 关于 lua 语法，参考 [Redis 使用lua脚本最全教程](https://blog.csdn.net/le_17_4_6/article/details/117588021?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-4-117588021.pc_agg_new_rank&utm_term=ARGV%E5%92%8Ckey+lua&spm=1000.2123.3001.4430)

#### 锁续期

加标识可以让各客户端释放自己的锁，但是上面还有一个问题需要解决：**如果业务还未执行完，但锁过期了怎么办？**

我们把过期时间延长，自然没有上面的问题，但是如果宕机了，锁没有被释放，这段时间都无法获得锁，除非自己去手动删除。设置的过长，自然没有上面的问题，但是如果宕机了，锁没有被释放，这段时间都无法获得锁，除非自己去手动删除。

我们需要设置合理的过期时间，**一般进行压测后，设置为业务的两倍**。所以上面的问题可能存在，对于这个问题的解决办法就是锁续期。

思路是开启一个定时器，每隔一段时间就去检查客户端是否持有锁（当然需要判断是否是自己加的锁），如果有则续上时间。

```java
private static final String RENEW_LOCK_SCRIPT =
"local lockClientId = redis.call('GET', KEYS[1])\n" +
"if lockClientId == ARGV[1] then\n" +
" redis.call('PEXPIRE', KEYS[1], ARGV[2])\n" +
" return true\n" +
"end\n" +
"return false";
```

定时任务，定时执行续锁代码：

```java
redisTemplate.execute(renewLockScript,
Collections.singletonList(lockKey), clientId,
String.valueOf(expireAfter));
```

> 对于上面的 lua 脚本，key[1] = Collections.singletonList(lockKey), argv[1] = clientId, argv[2] = String.valueOf(expireAfter)

#### 哨兵

**如果客户端宕机了，这个锁还会自动续期么？**

并不会，因为续期的定时器是在这个客户端本身执行，如果客户端宕机，也不会再进行续期，其余客户端想获取锁等到过期自动释放即可。

**如果客户端宕机，想立即释放这个锁怎么办？**

这就需要用到**哨兵**了，这个不是redis的集群哨兵，而是自己写的额外的一个服务，哨兵来维护所有redis客户端的列表。哨兵定时监控客户端是否宕机，一旦发现 client1 宕机，立即删除这个客户端的锁。

![image-20220303172151147](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220303172151147.png)

#### redisson 加锁

上面我们手动写一个定时任务来进行锁续期，但其实使用 Redision 就自带此功能。

Redisson 提供了 watch dog 自动延时机制，提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。也就是说，如果一个拿到锁的线程一直没有完成逻辑，那么看门狗会帮助线程不断的延长锁超时时间，锁不会因为超时而被释放。默认情况下，看门狗的续期时间是30s，也可以通过修改Config.lockWatchdogTimeout来另行指定。

### 18 Redisson

> [Redisson](https://redisson.org/)是架设在[Redis](http://redis.cn/)基础上的一个Java驻内存数据网格（In-Memory Data Grid）。充分的利用了Redis键值数据库提供的一系列优势，基于Java实用工具包中常用接口，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。

#### 使用

```java
public static void main(String[] args) {

    Config config = new Config();
    config.useSingleServer().setAddress("redis://127.0.0.1:6379");
    config.useSingleServer().setPassword("redis1234");
    
    final RedissonClient client = Redisson.create(config);  
    RLock lock = client.getLock("lock1");
    
    try{
        lock.lock();
    }finally{
        lock.unlock();
    }
}
```

#### getLock

获取锁实例

```java
public RLock getLock(String name) {
    return new RedissonLock(connectionManager.getCommandExecutor(), name);
}

public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {
    super(commandExecutor, name);
    //命令执行器
    this.commandExecutor = commandExecutor;
    //UUID字符串
    this.id = commandExecutor.getConnectionManager().getId();
    //内部锁过期时间
    this.internalLockLeaseTime = commandExecutor.
                getConnectionManager().getCfg().getLockWatchdogTimeout();
    this.entryName = id + ":" + name;
}
```

#### lock()

```java
public void lock(long leaseTime, TimeUnit unit) {
    try {
        this.lockInterruptibly(leaseTime, unit);
    } catch (InterruptedException var5) {
        Thread.currentThread().interrupt();
    }

}

public void lockInterruptibly() throws InterruptedException {
    this.lockInterruptibly(-1L, (TimeUnit)null);
}
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
    
    //当前线程ID
    long threadId = Thread.currentThread().getId();
    //尝试获取锁
    Long ttl = tryAcquire(leaseTime, unit, threadId);
    // 如果ttl为空，则证明获取锁成功
    if (ttl == null) {
        return;
    }
    //如果获取锁失败，则订阅到对应这个锁的channel
    RFuture<RedissonLockEntry> future = subscribe(threadId);
    commandExecutor.syncSubscription(future);
    try {
        while (true) {
            //再次尝试获取锁
            ttl = tryAcquire(leaseTime, unit, threadId);
            //ttl为空，说明成功获取锁，返回
            if (ttl == null) {
                break;
            }
            //ttl大于0 则等待ttl时间后继续尝试获取
            if (ttl >= 0) {
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                getEntry(threadId).getLatch().acquire();
            }
        }
    } finally {
        //取消对channel的订阅
        unsubscribe(future, threadId);
    }
}
```

先调用`tryAcquire`来获取锁，如果返回值ttl为空，则证明加锁成功，返回；如果不为空，则证明加锁失败。这时候，它会订阅这个锁的Channel，等待锁释放的消息，然后重新尝试获取锁。流程如下：

![image-20220303194651967](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220303194651967.png)

#### tryAcquire

```java
private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
    return (Long)this.get(this.tryAcquireAsync(leaseTime, unit, threadId));
}

private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, final long threadId) {

    //如果带有过期时间，则按照普通方式获取锁
    if (leaseTime != -1) {
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
    }

    //先按照30秒的过期时间来执行获取锁的方法
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(
        commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
        TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        
    //如果还持有这个锁，则开启定时任务不断刷新该锁的过期时间
    ttlRemainingFuture.addListener(new FutureListener<Long>() {
        @Override
        public void operationComplete(Future<Long> future) throws Exception {
            if (!future.isSuccess()) {
                return;
            }

            Long ttlRemaining = future.getNow();
            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        }
    });
    return ttlRemainingFuture;
}
```

#### tryLockInnerAsync

`tryLockInnerAsync`方法是真正执行获取锁的逻辑，它是一段LUA脚本代码。在这里，它使用的是hash数据结构。

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit,     
                            long threadId, RedisStrictCommand<T> command) {

        //过期时间
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  //如果锁不存在，则通过hset设置它的值，并设置过期时间
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  //如果锁已存在，并且锁的是当前线程，则通过hincrby给数值递增1
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  //如果锁已存在，但并非本线程，则返回过期时间ttl
                  "return redis.call('pttl', KEYS[1]);",
        Collections.<Object>singletonList(getName()), 
                internalLockLeaseTime, getLockName(threadId));
    }
```

1. 通过exists判断，如果锁不存在，则设置值和过期时间，加锁成功
2. 通过hexists判断，如果锁已存在，并且锁的是当前线程，则证明是重入锁，加锁成功
3. 如果锁已存在，但锁的不是当前线程，则证明有其他线程持有锁。返回当前锁的过期时间，加锁失败

![image-20220303200215032](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220303200215032.png)



加锁成功后，在redis的内存数据中，就有一条hash结构的数据。Key为锁的名称；field为随机字符串+线程ID；值为1。如果同一线程多次调用`lock`方法，值递增1。

#### unlock

```java
public void unlock() {
        try {
            get(unlockAsync(Thread.currentThread().getId()));
        } catch (RedisException e) {
            if (e.getCause() instanceof IllegalMonitorStateException) {
                throw (IllegalMonitorStateException)e.getCause();
            } else {
                throw e;
            }
        }
    }

public RFuture<Void> unlockAsync(final long threadId) {
    final RPromise<Void> result = new RedissonPromise<Void>();
    
    //解锁方法
    RFuture<Boolean> future = unlockInnerAsync(threadId);

    future.addListener(new FutureListener<Boolean>() {
        @Override
        public void operationComplete(Future<Boolean> future) throws Exception {
            if (!future.isSuccess()) {
                cancelExpirationRenewal(threadId);
                result.tryFailure(future.cause());
                return;
            }
            //获取返回值
            Boolean opStatus = future.getNow();
            //如果返回空，则证明解锁的线程和当前锁不是同一个线程，抛出异常
            if (opStatus == null) {
                IllegalMonitorStateException cause = 
                    new IllegalMonitorStateException("
                        attempt to unlock lock, not locked by current thread by node id: "
                        + id + " thread-id: " + threadId);
                result.tryFailure(cause);
                return;
            }
            //解锁成功，取消刷新过期时间的那个定时任务
            if (opStatus) {
                cancelExpirationRenewal(null);
            }
            result.trySuccess(null);
        }
    });

    return result;
}
```

#### unlockInnerAsync

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, EVAL,
    
            //如果锁已经不存在， 发布锁释放的消息
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            //如果释放锁的线程和已存在锁的线程不是同一个线程，返回null
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                "return nil;" +
            "end; " +
            //通过hincrby递减1的方式，释放一次锁
            //若剩余次数大于0 ，则刷新过期时间
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            "if (counter > 0) then " +
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "return 0; " +
            //否则证明锁已经释放，删除key并发布锁释放的消息
            "else " +
                "redis.call('del', KEYS[1]); " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; "+
            "end; " +
            "return nil;",
    Arrays.<Object>asList(getName(), getChannelName()), 
        LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

}
```

1. 如果锁已经不存在，通过publish发布锁释放的消息，解锁成功
2. 如果解锁的线程和当前锁的线程不是同一个，解锁失败，抛出异常
3. 通过hincrby递减1，先释放一次锁。若剩余次数还大于0，则证明当前锁是重入锁，刷新过期时间；若剩余次数小于0，删除key并发布锁释放的消息，解锁成功

![image-20220303201757853](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/picgo2022image-20220303201757853.png)

可以看到 Redisson 对于加锁解锁实现了 可重入，和开启定时器延期锁。

> 参考
>
> [分布式锁之Redis实现](https://www.jianshu.com/p/47fd7f86c848)  
>
> [Redis 分布式锁的正确实现](https://www.jianshu.com/p/73996715a38b)
>
> [Redis 分布式锁](https://www.jianshu.com/p/47fd7f86c848)
>
> [Redis 分布式锁过期了，但业务还没有执行完，怎么办](https://zhuanlan.zhihu.com/p/421843030)

### 19. 扩展

[redis6.0](https://mp.weixin.qq.com/s/_MWT5W8OjhdIrX6DS9rQcA)

[Redis 性能优化的 13 条军规！](https://mp.weixin.qq.com/s/ZOYS_UP19VYTDFVwJ7x8Pw)




> 参考
>
> [为什么说Redis是单线程的以及Redis为什么这么快](https://blog.csdn.net/u010870518/article/details/79470556?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)  
> [详解Redis中两种持久化机制RDB和AOF](https://baijiahao.baidu.com/s?id=1654694618189745916&wfr=spider&for=pc)  
> [Redis主从复制的原理](https://www.jianshu.com/p/4aa9591c3153)  
> [Redis主从复制原理总结](https://www.cnblogs.com/daofaziran/p/10978628.html)  
> [搞懂这些Redis知识点，吊打面试官！](https://baijiahao.baidu.com/s?id=1660009541007805174&wfr=spider&for=pc)
