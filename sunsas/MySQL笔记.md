# 3.4 MySQL笔记

> 作者：SunSAS
>
> **介绍：** 此篇记录平时遇到一些小知识点


## 3.4.1 基础部分


#### 1 整形不建议使用unsigned

首先得知道unsigned，无符号的。mysql 中 INT、SMALLINT 、 TINYINT、MEDIUMINT 和 BIGINT 整型类型

![Cgp9HWCbM_CASy8bAAEA5n3G3Kc663](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Cgp9HWCbM_CASy8bAAEA5n3G3Kc663.png)

有的人就想了，我这个字段没有负数，使用 unsigned 不是挺好的，范围更大？

比如 销售数量，不会出现负数

`sale_count int unsigned DEFAULT NULL `

但是如何对此列做统计，比如相减运算，是**可能出现错误**的

```java 
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(`test`.`s2`.`sale_count` - `test`.`s1`.`sale_count`)'
```

原因是 unsigned 数值相减之后依然为 unsigned，否则就会报错。而这里可能出现负数，就会报超出范围错误。

不过数据库可以设置参数 sql_mode 允许相减结果为signed：

```java
SET sql_mode='NO_UNSIGNED_SUBTRACTION';
```

所以如果要使用 unsigned，确保此参数设置。

#### 2 资金字段类型设置

大家都知道数据库禁止使用 Float 和 Double,事实上要使用高精度的数据存储也禁止这两种类型。这些类型因为不是高精度，也不是 SQL 标准的类型，**所以在真实的生产环境中不推荐使用**，否则在计算时，由于精度类型问题，会导致最终的计算结果出错。MySQL 8 提醒之后会放弃浮点类型。

代替的是Decimal，比如资金：

```java
money DECIMAL(8,2)
```

这样可以精确到分,不过**在海量互联网业务的设计标准中，并不推荐用 DECIMAL 类型，而是BIG INT**，就是把单位从“元”改为“分”

- DECIMAL(8,2) 表示存储最大值为 999999.99，可能不够，有的字段够，有的字段又不够，不好统一。使用BIG INT 所有金额相关字段都是定长字段，占用 8 个字节，存储高效。
- 类型 DECIMAL 是通过二进制实现的一种编码方式，计算效率远不如整型来的高效。

> 为何定长比可变长度性能好（类比CHAR和VARCHAR）？
>
> 从空间来看，可变长度要节省一点空间。但是性能比不上定长字段，一是数据库对定长字段有优化。二是如果修改后长度不同，导致容量不够存储，会导致‘**行迁移**’（Row Migration），造成多余的IO，而且原空间会成为**碎片空间**，无法继续使用，除非人为地进行表空间的碎片整理。

#### 3 自增整形主键

- 自增整型类型做主键，**务必使用类型 BIGINT**，而非 INT，后期表结构调整代价巨大（朝不保夕的小公司可以忽略）；

- 当达到自增整型类型的上限值时，再次自增插入，MySQL 数据库会报重复错误；

- MySQL 8.0 版本前，自增整型会有**回溯问题**。

> 回溯问题：比如,有三条数据 id =1，2， 3。下一个自增id 为 4.但是删除了 id  = 3 的数据后，再重启数据库，自增id 变为了3， 即自增值发生回溯。其原因是 MySQL 8.0 之前自增值没有持久化 。如果MySQL版本低，建议不要使用自增整形做主键。

#### 4 密码存储

禁止使用明文存储，这就不用说了。

##### 1.MD5加密

MD5 算法并不可逆。然而，MD5 加密后的值是固定的，如密码 12345678，它对应的 MD5 固定值即为 25d55ad283aa400af464c76d713c07ad。

因此，可以对 MD5 进行暴力破解，计算出所有可能的字符串对应的 MD5 值。若无法枚举所有的字符串组合，那可以计算一些常见的密码，如111111、12345678 等。

##### 2.加盐MD5

就是密码字符串加上盐存储，比如密码 123456 在数据库的值为：

```java
MD5（‘salt123456’）
```

- 若 salt 值被泄露，还是会暴力破解。
- 若存在相同密码，其中一个泄露，其余相同的都会被泄露。
- 若MD5算法被破解。

##### 3.动态盐 + 非固定加密算法

其格式为 $盐$算法$密码：

```java
$salt$cryption_algorithm$value
```

- salt ：**动态盐**就是根据不同的业务生成不同的盐，并存储在数据库中。若做得再精细一点，可以动态盐值 + 用户注册日期合并为一个更为动态的盐值。一般来说 盐 不需要保密，这是由于对于字符串的加密只要有一位不同，其散列值也完全不同。即使盐值泄露，暴力破解所有用户密码也是十分困难。
- cryption_algorithm ： 表示加密的算法，比如v1是MD5，v2是AES256等。
- value：就是加密后的字符串了。

在数据库存储的格式就是：

```java
$fgfaef$v1$2198687f6db06c9d1b31a030ba1ef074
```

如果MD5被破解，则升级加密算法即可。

##### 时间类型选择

MySQL中的时间类型有 YEAR、DATE、TIME、DATETIME、TIMESTAMEP。常用的就是DATETIME，TIMESTAMEP，还有人使用int 存储毫秒值表示时间，这还不如用TIMESTAMEP，本质都是自'1970-01-01 00:00:00' 到现在的毫秒数。虽说性能比TIMESTAMEP稍好，但是在现代CPU看来，好处微乎其微，而且大大降低可读性，是在得不偿失。

##### 1.TIMESTAMEP

实际存储的内容为‘1970-01-01 00:00:00’到现在的毫秒数。在 MySQL 中，由于类型 TIMESTAMP 占用 4 个字节，因此其存储的时间上限只能到‘2038-01-19 03:14:07’。

从 MySQL 5.6 版本开始，类型 TIMESTAMP 能支持毫秒。与 DATETIME 不同的是，**若带有毫秒时，类型 TIMESTAMP 占用 7 个字节，而 DATETIME 存储毫秒信息，占用 8 个字节，不存储毫秒则是5个字节（mysql5.6.4，之前是8个字节）**。

类型 TIMESTAMP **最大的优点是可以带有时区属性**，因为它本质上是从毫秒转化而来。如果你的业务需要对应不同的国家时区，那么类型  TIMESTAMP 是一种不错的选择。比如新闻类的业务，通常用户想知道这篇新闻发布时对应的自己国家时间，那么 TIMESTAMP 是一种选择。

如果使用默认的操作系统时区，则每次通过时区计算时间时，要调用操作系统底层系统函数 __tz_convert()，而这个函数需要额外的加锁操作，以确保这时操作系统时区没有修改。所以，当大规模并发访问时，由于热点资源竞争，会产生两个问题：

- **性能不如 DATETIME：** DATETIME 不存在时区转化问题。
- **性能抖动：** 海量并发时，存在性能抖动问题。

##### 2.DATETIME

类型 DATETIME 最终展现的形式为：YYYY-MM-DD HH：MM：SS，**固定占用 8 个字节**。

从 MySQL 5.6 版本开始，DATETIME 类型支持毫秒，DATETIME(N) 中的 N 表示毫秒的精度。例如，DATETIME(6) 表示可以存储 6 位的毫秒值。同时，一些日期函数也支持精确到毫秒，例如常见的函数 NOW、SYSDATE

```mysql
create_time DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
update_time DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
```

上面这两个是多数互联网公司使用的 创建时间，修改时间 建表语句。DEFAULT CURRENT_TIMESTAMP(6) 默认就是当前时间，ON UPDATE CURRENT_TIMESTAMP(6) 表示每次修改都会修改为当前时间。

> 建议每个核心表都应该有 update_time 字段记录修改时间。

##### 3.小结

| 对比       | 优点                   | 缺点                       |
| ---------- | ---------------------- | -------------------------- |
| TIMESTAMEP | 拥有时区，空间占用较少 | 时限有上限，需显示设置时区 |
| DATETIME   | 全能稳定               | 空间占用较大，没有时区     |

尽可能使用类型 DATETIME，对于时区控制可以放到服务或前端。除非确定适合TIMESTAMP，比如公司撑不到2038年。

#### 5 JSON格式

mysql 在 5.7 后支持 json 类型，还能在每个对应的字段上创建索引。之前的话要存这种非关系型只能把json视为字符串，到程序中处理。

而 8.0 版本解决了更新 JSON 的日志性能瓶颈。如果要在生产环境中使用 JSON 数据类型，强烈推荐使用 MySQL 8.0 版本。下面就 MySQL 如何操作 JSON 做介绍。

##### 创建

```mysql
CREATE TABLE UserLogin (
    userId BIGINT NOT NULL,
    loginInfo JSON,
    PRIMARY KEY(userId)
);
```

##### 新增数据

```mysql
INSERT INTO UserLogin VALUES (1,'{
	"cellphone" : "13918888888",
	"wxchat" : "破产码农",
    "QQ" : "82946772"
}
');
INSERT INTO UserLogin VALUES (2,'{
	"cellphone" : "15026888888"
}
');
```

##### 查询数据

```mysql
SELECT userid , 
JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.cellphone")) cellphone,
JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.wxchat")) wxchat
FROM userlogin
-- MySQL 还提供了 ->> 表达式,上面的简化写法
SELECT userid , 
loginInfo->>"$.cellphone" cellphone,
loginInfo->>"$.wxchat" wxchat
FROM userlogin
```

##### 创建索引

根据JSON某个字段创建列，然后在列上加索引：

```mysql
ALTER TABLE UserLogin ADD COLUMN cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone");
ALTER TABLE userlogin ADD UNIQUE INDEX idx_cellphone(cellphone);
```

```mysql
explain select * from userlogin where cellphone = '13918888888'

possible_keys: idx_cellphone
          key: idx_cellphone
```

也可以在建表时就加上索引：

```mysql
CREATE TABLE UserLogin (
    userId BIGINT,
    loginInfo JSON,
    cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone"),
    PRIMARY KEY(userId),
    UNIQUE KEY uk_idx_cellphone(cellphone)
);
```

##### 用户画像设计应用

比如很多平台会对用户打上各种标签，每个用户的标签都不同，一种存储方式是，每个标签中间使用分好隔开。

```mysql
|用户    |标签                                   |
+-------+---------------------------------------+
|David  |80后 ； 高学历 ； 小资 ； 有房 ；常看电影   |
|Tom    |90后 ；常看电影 ； 爱外卖                 |       
```

这样做的缺点是： **不好搜索特定标签**的用户，另外分隔符也是一种自我约定，在数据库中其实可以任意存储其他数据，最终产生脏数据。

用JSON类型就可以解决。

首先建一个 Tag 表，存的是各种标签：

```mysql
CREATE TABLE Tags (
    tagId bigint auto_increment,
    tagName varchar(255) NOT NULL,
    primary key(tagId)
);
```

用户表：

```mysql
CREATE TABLE UserTag (
    userId bigint NOT NULL,
    userTags JSON,
    PRIMARY KEY (userId)
);
```

**插入数据：**

```mysql
INSERT INTO UserTag VALUES (1,'[2,6,8,10]');
INSERT INTO UserTag VALUES (2,'[3,10,12]');
```

**添加索引：**

MySQL 8.0.17 版本开始支持 Multi-Valued Indexes，用于在 JSON 数组上创建索引，并通过函数 member  of、json_contains、json_overlaps 来快速检索索引数据。所以你可以在表 UserTag 上创建  Multi-Valued Indexes：

```mysql
ALTER TABLE UserTag
ADD INDEX idx_user_tags ((cast((userTags->"$") as unsigned array)));
```

如果想要查询用户画像为常看电影（tag 包含10）的用户，可以使用函数 **MEMBER OF**：

```mysql
EXPLAIN SELECT * FROM UserTag 
WHERE 10 MEMBER OF(userTags->"$")

possible_keys: idx_user_tags
          key: idx_user_tags
```

如果想要查询用户画像为为 80 后，且常看电影（tag 包含 2 and tag 包含 10）的用户，可以使用函数 **JSON_CONTAINS**：

```mysql
EXPLAIN SELECT * FROM UserTag 
WHERE JSON_CONTAINS(userTags->"$", '[2,10]')

possible_keys: idx_user_tags
          key: idx_user_tags
```

#### 6 JOIN 查询

##### 1 JOIN连接算法

MySQL 8.0 版本支持两种 JOIN 算法用于表之间的关联：

- Nested Loop Join；
- Hash Join。

通常认为，在 OLTP 业务中，因为**查询数据量较小**、语句相对简单，大多使用索引连接表之间的数据。这种情况下，优化器大多会用 Nested  Loop Join 算法；而 OLAP 业务中的查询数据量较大，关联表的数量非常多，所以用 Hash Join 算法，直接扫描全表效率会更高。

#### Nested Loop Join

Nested Loop Join 之间的表关联是使用索引进行匹配的，表 R 中通过 WHERE 条件过滤出的数据会在表 S 对应的索引上进行一一查询。如果驱动表 R 的数据量不大，上述算法非常高效。这也是我们常说的小表驱动大表。

对于 Left Join 来说，驱动表就是左表 R；Right Join中，驱动表就是右表 S。这是 JOIN 类型**决定左表或右表的数据一定要进行查询**。但对于 INNER JOIN，驱动表可能是表 R，也可能是表 S。**谁需要查询的数据量越少，谁就是驱动表。**

为了理解**优化器驱动表的选择**，先来看下面这条 SQL：

```java
    SELECT COUNT(1) 
    FROM orders
    INNER JOIN lineitem
      ON orders.o_orderkey = lineitem.l_orderkey 
        WHERE orders.o_orderdate >= '1994-02-01' 
          AND  orders.o_orderdate < '1994-03-01'
        
EXPLAIN: -> Aggregate: count(1)
     -> Nested loop inner join  (cost=115366.81 rows=549152)
         -> Filter: ((orders.O_ORDERDATE >= DATE'1994-02-01') and (orders.O_ORDERDATE < DATE'1994-03-01'))  (cost=26837.49 rows=133612)
             -> Index range scan on orders using idx_orderdate  (cost=26837.49 rows=133612)
         -> Index lookup on lineitem using PRIMARY (l_orderkey=orders.o_orderkey)  (cost=0.25 rows=4)

```

驱动表是 过滤时间后的数据，预估记录数13万条。

但若执行的是下面这条 SQL，则执行计划就有了改变：

```mysql
    EXPLAIN FORMAT=tree
    SELECT COUNT(1) 
    FROM orders
    INNER JOIN lineitem
      ON orders.o_orderkey = lineitem.l_orderkey 
        WHERE orders.o_orderdate >= '1994-02-01' 
          AND  orders.o_orderdate < '1994-03-01'
          AND lineitem.l_partkey = 620758

    EXPLAIN: -> Aggregate: count(1)
    -> Nested loop inner join  (cost=17.37 rows=2)
        -> Index lookup on lineitem using lineitem_fk2 (L_PARTKEY=620758)  (cost=4.07 rows=38)
        -> Filter: ((orders.O_ORDERDATE >= DATE'1994-02-01') and (orders.O_ORDERDATE < DATE'1994-03-01'))  (cost=0.25 rows=0)
            -> Single-row index lookup on orders using PRIMARY (o_orderkey=lineitem.l_orderkey)  (cost=0.25 rows=1)
```

新增了一个条件 lineitem.l_partkey =620758,找到的记录大约只有 38 条,因此这时优化器选择表 lineitem 为驱动表。

#### Hash Join

MySQL 中的第二种 JOIN 算法是 Hash Join，用于两张表之间连接条件没有索引的情况。

有同学会提问，没有连接，那创建索引不就可以了吗？或许可以，但：

1. 如果有些列是低选择度的索引，那么创建索引在导入数据时要对数据排序，影响导入性能；
2. 二级索引会有回表问题，若筛选的数据量比较大，则直接全表扫描会更快。

对于 OLAP 业务查询来说，Hash Join 是必不可少的功能，**MySQL 8.0 版本开始支持 Hash Join 算法**，加强了对于 OLAP 业务的支持。

所以，如果你的查询数据量不是特别大，对于查询的响应时间要求为分钟级别，完全可以使用单个实例 MySQL 8.0 来完成大数据的查询工作。

Hash Join会扫描关联的两张表：

- 首先会在扫描驱动表的过程中创建一张哈希表；
- 接着扫描第二张表时，会在哈希表中搜索每条关联的记录，如果找到就返回记录。

Hash Join 选择驱动表和 Nested Loop Join 算法大致一样，都是较小的表作为驱动表。如果驱动表比较大，创建的哈希表超过了内存的大小，MySQL 会**自动把结果转储到磁盘**。

##### 2 JOIN 能不能写？

很多人会纠结能不能写JOIN查询，认为JOIN查询会更慢。这并不对，如果join的列都有索引，那连接查询效率会比多次查询更快，但这需要良好的索引设计。

|      | join                                                         | 多次查询                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 优点 | 书写简单，避免多余代码。索引设计良好情况下，效率高。         | sql不做逻辑控制，改为serivec层。sql易复用，逻辑修改灵活，易维护。 |
| 缺点 | sql不能或难以复用，sql逻辑复杂，不易修改。错误使用join，过多join会导致效率降低。join语句不好做分库分表。 | 书写麻烦，service层需要依赖stream做很多处理。可读性不强。    |

关于join的使用，还要结合业务场景来决定，比如**OLTP**的系统，海量并发，要求响应非常及时，在毫秒级别返回结果。使用join需要小心，确保其索引设计合理。

```mysql
SELECT o_custkey, o_orderdate, o_totalprice, p_name FROM orders,lineitem, part
WHERE o_orderkey = l_orderkey
  AND l_partkey = p_partkey
  AND o_custkey = ?
ORDER BY o_orderdate DESC
LIMIT 30;
```

```mysql
--执行计划
EXPLAIN: -> Limit: 30 row(s)  (cost=27.76 rows=30)

    -> Nested loop inner join  (cost=27.76 rows=44)

        -> Nested loop inner join  (cost=12.45 rows=44)

            -> Index lookup on orders using idx_custkey_orderdate (O_CUSTKEY=1; iterate backwards)  (cost=3.85 rows=11)

            -> Index lookup on lineitem using PRIMARY (l_orderkey=orders.o_orderkey)  (cost=0.42 rows=4)

        -> Single-row index lookup on part using PRIMARY (p_partkey=lineitem.L_PARTKEY)  (cost=0.25 rows=1)
```

而OLAP(Online Analytical Processing)系统，数据读取非常多，而且也不要求执行时间。就可以用join查询。

还有现在微服务流行，对于数据库可能不在一个服务，不在一个库，就不能使用join了。这时候就要确保join的使用是在一个模块下的，比如用户系统中去联合查询用户信息。这种一般也是放在一个库，可以使用join。

#### 7 子查询

MySQL 8.0 版本中，子查询的优化得到大幅提升。但之前的版本，应当慎用子查询。

其实很多的子查询都可以用join去替代，只不过用子查询要好理解很多。比如下面这段sql：

```mysql
    SELECT
        COUNT(c_custkey) cnt
    FROM
        customer
    WHERE
        c_custkey NOT IN (
            SELECT
                o_custkey
            FROM
                orders
            WHERE
                o_orderdate >=  '1993-01-01'
                AND o_orderdate <  '1994-01-01'
    	);
-- 表 customer 存在，在表 orders 不存在,改用left join写法
    SELECT
        COUNT(c_custkey) cnt
    FROM
        customer
            LEFT JOIN
        orders ON
                customer.c_custkey = orders.o_custkey
                AND o_orderdate >= '1993-01-01'
                AND o_orderdate < '1994-01-01'
    WHERE
        o_custkey IS NULL;
```

**LEFT JOIN 是一个代数关系，而子查询更偏向于人类的思维角度进行理解。**

在 MySQL 8.0 中，优化器会自动地将 IN 子查询优化，优化为最佳的 JOIN 执行计划，这样一来，会显著的提升性能。所以你可以用子查询，更有利于理解sql。

关于in 和 exists，也做了优化，所以要看哪个性能更好，需要关注sql执行计划，可能他们的执行计划是一样的。

##### **依赖子查询的优化**

在 MySQL 8.0 版本之前，MySQL 对于子查询的优化并不充分。所以在子查询的执行计划中会看到 **DEPENDENT SUBQUERY** 的提示，这表示是一个依赖子查询，**子查询需要依赖外部表的关联**。

> 依赖子查询又被称为 相关子查询，与独立子查询区别参考 [相关子查询和嵌套子查询](https://www.cnblogs.com/Ryan_j/archive/2010/10/20/1857026.html)

```java
    SELECT
        *
    FROM
        orders
    WHERE
        (o_clerk , o_orderdate) IN (
            SELECT
                o_clerk, MAX(o_orderdate)
            FROM
                orders
            GROUP BY o_clerk);
```

老版本的 MySQL 数据库，它的执行计划将会是依赖子查询，执行计划如下所示

![image-20210624173553248](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/image-20210624173553248.png)

先进行外表扫描，接着做依赖子查询的判断。**所以，子查询执行了500多万次，而不是1次**，可想而知其效率有多慢了。

那么对于上面的这条 SQL ，可将其重写为：

```mysql
SELECT * FROM orders o1,
(
    SELECT
        o_clerk, MAX(o_orderdate)
    FROM
        orders
    GROUP BY o_clerk
) o2
WHERE
    o1.o_clerk = o2.o_clerk
    AND o1.o_orderdate = o2.orderdate;
```

总结来看：

1. 子查询相比 JOIN 更易于人类理解，所以受众更广，使用更多；
2. 当前 MySQL 8.0 版本可以“毫无顾忌”地写子查询，对于子查询的优化已经相当完备；
3. 对于老版本的 MySQL，**请 Review 所有子查询的SQL执行计划，** 对于出现 DEPENDENT SUBQUERY 的提示，请务必即使进行优化，否则对业务将造成重大的性能影响；
4. DEPENDENT SUBQUERY 的优化，一般是重写为派生表进行表连接。

## 3.4.2 索引部分

关于索引可以先看 [3.1.3 MySQL索引](https://sunsas.gitee.io/doc/#/sunsas/MySQL?id=_313-mysql索引)

### 1 B+树索引

关于B+ 树索引具体就不再介绍了，为啥要使用B+ 树，而不是二叉树，哈希索引、红黑树、SkipList。因为它是目前为止排序最有效率的数据结构，当然这是基于磁盘存储海量数据，如果是内存中，一般使用后者。

B+树索引的特点是： **基于磁盘的平衡树**，但树非常矮，通常为 3~4 层，能存放千万到上亿的排序数据。树矮意味着访问效率高，从千万或上亿数据里查询一条数据，只用 3、4 次 I/O。

在 MySQL InnoDB 存储引擎中，一个页的大小为 16K，在上面的表 User 中，键值 userId 是BIGINT 类型，则：

```java
根节点能最多存放以下多个键值对 = 16K / 键值对大小(8+6) ≈ 1100
```

再假设表 User 中，每条记录的大小为 500 字节，则：

    叶子节点能存放的最多记录为 = 16K / 每条记录大小 ≈ 32

综上所述，树高度为 2 的 B+ 树索引，最多能存放的记录数为：

    总记录数 = 1100 * 32 =  35,200

同理，树高度为 3 的 B+ 树索引，最多能存放的记录数为：

    总记录数 = 1100（根节点） * 1100（中间节点） * 32 =  38,720,000

不过，在真实环境中，每个页其实利用率并没有这么高，还会存在一些碎片的情况，我们假设每个页的使用率为60%，则：

![image-20210723104048913](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/image-20210723104048913.png)

### 2 索引开销

索引在插入时，需要进行排序，但是排序开销并不大，真正的开销是插入索引，也就是索引的维护。

- **数据顺序**（或逆序）插入： B+ 树索引的维护代价非常小，叶子节点都是从左往右进行插入，比较典型的是自增 ID 的插入、时间的插入（若在自增 ID 上创建索引，时间列上创建索引，则 B+ 树插入通常是比较快的）。
- **数据无序**插入： B+ 树为了维护排序，需要对页进行分裂、旋转等开销较大的操作，另外，即便对于固态硬盘，随机写的性能也不如顺序写，所以磁盘性能也会收到较大影响。比较典型的是用户昵称，每个用户注册时，昵称是随意取的，若在昵称上创建索引，插入是无序的，索引维护需要的开销会比较大。

但我们也不可能要求所有顺序都有序，不然也不用索引了。只不过要求**数据库主键索引为有序**，比如使用自增，或使用函数 UUID_TO_BIN 排序的 UUID，而不用无序值做主键。

#### 查询哪些索引未被使用过

在 MySQL 数据库中，可以通过查询表 `sys.schema_unused_indexes`，查看有哪些索引一直未被使用过，可以被废弃。

而 MySQL 8.0 版本推出了索引不可见（Invisible）功能。在删除废弃索引前，用户可以将索引设置为对优化器不可见，然后观察业务是否有影响。若无，DBA 可以更安心地删除这些索引：

```mysql
ALTER TABLE t1 
ALTER INDEX idx_name INVISIBLE/VISIBLE;
```

### 3 索引组织表

数据存储有 堆表和索引组织表两种方式，其实也就是非聚簇索引和聚簇索引。

堆表中的数据无序存放， 数据的排序完全依赖于索引（Oracle、Microsoft SQL Server、PostgreSQL 早期默认支持的数据存储都是堆表结构）。当堆表的数据发生改变，且位置发生了变更，所有索引中的地址都要更新，这非常影响性能，特别是对于 OLTP 业务。

#### 二级索引

二级索引又叫辅助索引，他很像非聚簇索引，这两个不同之处在于 **非聚簇索引叶子结点存的是数据对应的位置**，**二级索引叶子结点存的是 主键索引对应的主键值**，这样好处是，除非主键发生了更改，大部分情况都不需要维护二级索引。 当然插入了数据还是得维护的。

> 通过主键值再次查询主键索引的操作被称为 **回表**。

上面我们说过，只要求主键索引数据递增，但二级索引上可能有很多不会是有序递增，所以二级索引的维护性能开销相对较大，比如用户昵称，这种可以要求单个用户每天、甚至是每年昵称更新的次数，比如每天更新一次，每年更新三次。

在实际核心业务中，开发同学还有很大可能会设计带有业务属性的主键，但请牢记以下两点设计原则：

- 要比较顺序，对聚集索引性能友好；
- 尽可能紧凑，对二级索引的性能和存储友好；

#### 函数索引

之前我们说过 数据库规范要求查询条件中函数写在等式右边，而不能写在左边 ，不要在查询时使用 函数表达式，会使索引失效。

如果确实需要加函数，或者代码已经如此写了，线上查询慢，最快的解决办法就是新建**函数索引**。

```mysql
ALTER TABLE User 
ADD INDEX 
idx_func_register_date((DATE_FORMAT(register_date,'%Y-%m')));
```

暂时解决线上查询慢，下个版本想办法修复掉，没必要用函数索引就不用。

第二种是**结合虚拟列**使用：

```mysql
CREATE TABLE UserLogin (
    userId BIGINT,
    loginInfo JSON,
    cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone"),
    PRIMARY KEY(userId),
    UNIQUE KEY idx_cellphone(cellphone)
);
```

cellphone 九十一个虚拟列，它是由后面的表达式计算而成，本身这个列不占用任何的存储空间，而索引 idx_cellphone 实质是一个函数索引。这样做得好处是在写 SQL 时可以直接使用这个虚拟列，而不用写冗长的函数：

```mysql
-- 不用虚拟列
SELECT  *  FROM UserLogin
WHERE loginInfo->>"$.cellphone" = '13918888888'

-- 使用虚拟列
SELECT  *  FROM UserLogin 
WHERE cellphone = '13918888888'
```

对于爬虫类的业务，我们会从网上先爬取很多数据，其中有些是我们关心的数据，有些是不关心的数据。通过虚拟列技术，可以展示我们想要的那部分数据，再通过虚拟列上创建索引，就是对爬取的数据进行快速的访问和搜索。

### 4 组合索引

首先我们注意一个点：

假如有组合索引  (a,b)，那么下面sql都能用到组合索引：

```mysql
SELECT * FROM table WHERE a = ?
SELECT * FROM table WHERE a = ? AND b = ?
SELECT * FROM table WHERE b = ? AND a = ?
SELECT * FROM table WHERE a = ? ORDER BY b DESC
SELECT * FROM table WHERE a = ? AND b > ?
```

关于第三个sql，其实mysql解析就优化了，所以不论你写的位置是如何，都可以用到索引。

关于第四个sql，由于我们索引已经是（a,b）排序，所以可以用到索引，这也是一个**可以优化的点**。

以下sql则不能使用组合索引：

```mysql
SELECT * FROM table WHERE b = ? ORDER BY a DESC
SELECT * FROM table WHERE a > ? AND b = ? 
```

### 5 组合索引优化

**排序优化**：

刚才我们说过，`SELECT * FROM table WHERE a = ? ORDER BY b DESC` 是可以用到索引的，对于下面sql：

```mysql
SELECT * FROM orders 
WHERE o_custkey = 147601 ORDER BY o_orderdate DESC
```

如果只有一个索引 o_custkey，那么还需要额外的一次时间排序才能得到结果。

为此，我们在表 orders 上创建新的组合索引 idx_custkey_orderdate，对字段（o_custkey，o_orderdate）进行索引，这样就不需要额外的一次排序了。

**覆盖索引**：

之前我说过回表，如果查询二级索引有的字段没有，就得去通过主键查询数据了，但如果要回表的数据很多，那其实很影响性能，所以最好是二级索引上包含所查询的数据。

### 6 MySQL索引优化器

MySQL 数据库由 Server 层和 Engine 层组成：

- Server 层有 SQL 分析器、SQL优化器、SQL 执行器，用于负责 SQL 语句的具体执行过程；
- Engine 层负责存储具体的数据，如最常使用的 InnoDB 存储引擎，还有用于在内存中存储临时结果集的 TempTable 引擎。

SQL 优化器会分析所有可能的执行计划，选择成本最低的执行，这种优化器称之为：**CBO**（Cost-based Optimizer，基于成本的优化器）。

在 MySQL中，**一条 SQL 的计算成本计算如下所示：**

```java
Cost  = Server Cost + Engine Cost
      = CPU Cost + IO Cost
```

数据库 mysql 下的表 server_cost、engine_cost 则记录了对于各种成本的计算，可以自行查看。这些值自己也是可以改的

#### MySQL索引出错案例

**案例1：未能使用创建的索引**

```mysql
EXPLAIN FORMAT=tree 
SELECT * FROM orders 
WHERE o_orderdate > '1994-01-01' 
AND o_orderdate < '1994-12-31'\G
*************************** 1. row ***************************
EXPLAIN: -> Filter: ((orders.O_ORDERDATE > DATE'1994-01-01') and (orders.O_ORDERDATE < DATE'1994-12-31'))  (cost=592267.11 rows=1876082)
    -> Table scan on orders  (cost=592267.11 rows=5799601)

EXPLAIN FORMAT=tree 
SELECT * FROM orders FORCE INDEX(idx_orderdate)
WHERE o_orderdate > '1994-01-01' 
AND o_orderdate < '1994-12-31'\G
*************************** 1. row ***************************
EXPLAIN: -> Index range scan on orders using idx_orderdate, with index condition: ((orders.O_ORDERDATE > DATE'1994-01-01') and (orders.O_ORDERDATE < DATE'1994-12-31'))  (cost=844351.87 rows=1876082)
```

o_orderdate 是一个二级索引，但是执行上面sql语句却没有使用该索引，这是因为走索引成本更高，下面强制使用索引的 cost 为 844351.87，MySQL认为全表扫描低于使用二级索引。故，MySQL 优化器没有使用二级索引 idx_orderdate。二级索引是回表的数据过多导致不如全表扫描，如果回表数据没那么多，就会使用二级索引了。

**案例2：索引创建在有限状态上**

我们通常说有限状态的列不应该建立索引，不过这种是基于状态分布均匀的列，比如性别，比例大致为1：1。但有的列会有很大的**数据倾斜**，比如订单状态，有已支付，未支付，取消等几个状态。但是大部分的是已支付和取消。有时我们想查询有哪些订单是未支付的，就在订单状态列 o_orderstatus 添加了索引。

然而查询结果看，并没有使用到这个索引，优化器会认为订单状态为 未支付 的订单占用 1/3 的数据，使用全表扫描，避免二级索引回表的效率会更高。

这种情况下，我们可以利用 MySQL 8.0 的直方图功能，创建一个**直方图**，让优化器知道数据的分布，从而更好地选择执行计划。直方图的创建命令如下所示：

```mysql
ANALYZE TABLE orders 
UPDATE HISTOGRAM ON o_orderstatus;
```


