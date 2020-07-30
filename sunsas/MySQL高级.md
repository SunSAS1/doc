# 3.2 MySQL高级

> 作者：SunSAS
>
> **介绍：** SunSAS是SunSAS


## 3.2.1 MySQL调优

### 1. 影响MySQL性能因素

- 业务需求对MySQL的影响(合适合度)
- 存储定位对MySQL的影响
    - 系统各种配置及规则数据
    - 活跃用户的基本信息数据
    - 活跃用户的个性化定制信息数据
    - 准实时的统计信息数据
    - 其他一些访问频繁但变更较少的数据
    - 二进制多媒体数据
    - 流水队列数据
    - 超大文本数据
    - 不适合放进MySQL的数据
    - 需要放进缓存的数据
- Schema设计对系统的性能影响
    - 尽量减少对数据库访问的请求
    - 尽量减少无用数据的查询请求
- 硬件环境对系统性能的影响
    - **CPU**：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候
    - **IO**：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候
    - **服务器硬件的性能瓶颈**：top，free，iostat 和 vmstat来查看系统的性能状态

### 2. 性能分析

在优化MySQL时，通常需要对数据库进行分析，常见的分析手段有慢查询日志，**EXPLAIN 分析查询**，**profiling分析**以及**show命令查询系统状态及系统变量**，通过定位分析性能的瓶颈，才能更好的优化数据库系统的性能。

#### 2.1 show命令

```
-- 显示状态信息（扩展show status like ‘XXX’）
Mysql> show status 
-- 显示系统变量（扩展show variables like ‘XXX’）
Mysql> show variables 
-- 显示InnoDB存储引擎的状态
Mysql> show innodb status 
-- 查看当前SQL执行，包括执行状态、是否锁表等
Mysql> show processlist 
-- 显示系统变量
Shell> mysqladmin variables -u username -p password
-- 显示状态信息
Shell> mysqladmin extended-status -u username -p password
```


#### 2.2 Explain(执行计划)
使用 Explain 关键字可以模拟优化器执行SQL查询语句，从而知道 MySQL 是如何处理你的 SQL 语句的。分析你的查询语句或是表结构的性能瓶颈。

通过执行计划，我们可以知道以下信息：
- 表的读取顺序
- 数据读取操作的操作类型
- 哪些索引可以使用
- 哪些索引被实际使用
- 表之间的引用
- 每张表有多少行被优化器查询

直接在查询语句前加`explain`就可以：
```
explain SELECT * from user where username = "SunSAS"
```
![explain](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/QQ%E5%9B%BE%E7%89%8720200722114805.png)

1. **id**（select 查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序）
    - id相同，执行顺序从上往下
![1](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/20170509234043416.png)
    - id全不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
![2](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/20170510223451835.png)
    - id部分相同，执行顺序是先按照数字大的先执行，然后数字相同的按照从上往下的顺序执行
![3](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/20170510224901726.png)
2. **select_type**（查询类型，用于区别普通查询、联合查询、子查询等复杂查询）
    - **SIMPLE** ：简单的select查询，查询中不包含子查询或UNION
    - **PRIMARY**：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
    - **SUBQUERY**：在select或where列表中包含了子查询
    - **DERIVED**：在from列表中包含的子查询被标记为DERIVED，MySQL会递归执行这些子查询，把结果放在临时表里
    - **UNION**：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
    - **UNION RESULT**：从UNION表获取结果的select
3. **table**（显示这一行的数据是关于哪张表的）
4. ==**type**==（显示查询使用了那种类型，从最好到最差依次排列 **system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL** ）
    - **system**：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现
    - **const**：表示通过索引一次就找到了，const 用于比较 primary key 或 unique 索引，因为只要匹配一行数据，所以很快，如将主键置于 where 列表中，mysql 就能将该查询转换为一个常量
    - **eq_ref**：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
    - **ref**：非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体
    - **range**：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引
    - **index**：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）
    - **ALL**：Full Table Scan，将遍历全表找到匹配的行
    > 一般来说，得保证查询至少达到range级别，最好到达ref
5. **possible_keys**（显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用）
6. **==key==**:    实际使用的索引，如果为NULL，则没有使用索引.  
    查询中若使用了覆盖索引，则该索引和查询的 select 字段重叠，仅出现在key列表中
7. **key_len**    表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好。  
key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。
8. **ref**:显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值。
9. **rows**（根据表统计信息及索引选用情况，大致估算找到所需的记录所需要读取的行数）
10. ==**Extra**==：不适合在其他字段中显示，但是十分重要的额外信息。
    - **using filesort**：说明mysql会对数据使用一个外部的索引排序，不是按照表内的索引顺序进行读取。mysql中无法利用索引完成的排序操作称为“文件排序”。常见于order by和group by语句中
    - **Using temporary**：使用了临时表保存中间结果，mysql在对查询结果排序时使用临时表。常见于排序order by和分组查询group by。
    - **using index**：表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找；否则索引被用来读取数据而非执行查找操作。
    - **using where**：使用了where过滤
    - **using join buffer**：使用了连接缓存
    - **impossible where**：where子句的值总是false，不能用来获取任何元祖
    - **select tables optimized away**：在没有group by子句的情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化
    - **distinct**：优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作
> 元组是关系数据库中的基本概念，关系是一张表，表中的每行（即数据库中的每条记录）就是一个元组，每列就是一个属性。 在二维表里，元组也称为记录。

**案例**：
![case](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/case.jpg)
执行顺序(id全不同，id越大优先执行):
1. （id = 4）、`select id, name from t2`：select_type 为union，说明id=4的select是union里面的第二个select。
2. （id = 3）、`select id, name from t1 where address = ‘11`：因为是在from语句中包含的子查询所以被标记为DERIVED（衍生），where address = ‘11’ 通过复合索引idx_name_email_address就能检索到，所以type为index。
3. （id = 2）、`select id from t3`：因为是在select中包含的子查询所以被标记为SUBQUERY。
4. （id = 1）、`select d1.name, … d2 from … d1`：select_type为PRIMARY表示该查询为最外层查询，table列被标记为 “derived3”表示查询结果来自于一个衍生表（id = 3 的select结果）。
5. （id = NULL）、`… union … `：代表从union的临时表中读取行的阶段，table列的 “union 1, 4”表示用id=1 和 id=4 的select结果进行union操作。

#### 2.3 Show Profile 分析查询
通过慢日志查询可以知道哪些 SQL 语句执行效率低下，通过 explain 我们可以得知 SQL 语句的具体执行情况，索引使用等，还可以结合Show Profile命令查看执行状态。
默认该功能是关闭的,因为开启很耗性能，使用前需开启。
```
-- 查看是否开启profile
Show  variables like 'profiling';
-- 开启profile（会话级别的，关闭当前会话就会恢复原来的关闭状态）,下面两个都可以。
set profiling=1; 
set profiling=ON;
-- 关闭profile,下面两个都可以。
set profiling=0; 
set profiling=OFF;
-- 查看结果
show profiles;
-- 显示当前查询语句执行的时间和系统资源消耗
/*Query_ID为show profiles列表中的Query_ID*/
show profile cpu,block io for query Query_ID;
```

> **日常开发需注意的结论**

1. **converting  HEAP to MyISAM**：查询结果太大，内存不够，数据往磁盘上搬了。
2. **Creating tmp table**：创建临时表。先拷贝数据到临时表，用完后再删除临时表。
3. **Copying to tmp table on disk**：把内存中临时表复制到磁盘上，危险！！！
4. **locked**。

**如果在show profile诊断结果中出现了以上4条结果中的任何一条，则sql语句需要优化。**




### 3. 慢查询
MySQL 的慢查询日志是 MySQL 提供的一种日志记录，它用来记录在 MySQL 中响应时间超过阈值的语句，具体指运行时间超过 `long_query_time` 值的 SQL，则会被记录到慢查询日志中。
> long_query_time 的默认值为10，意思是运行10秒以上的语句.

默认情况下，MySQL数据库没有开启慢查询日志，**需要手动设置参数开启**

```
-- 查看开启状态
SHOW VARIABLES LIKE '%slow_query_log%'
-- 开启慢查询日志(临时配置)
set global slow_query_log='ON';
-- set文件位置，系统会默认给一个缺省文件host_name-slow.log
set global slow_query_log_file='/var/lib/mysql/hostname-slow.log';
set global long_query_time=2;
```
set操作开启慢查询日志**只对当前数据库生效**，如果MySQL重启则会失效。

> 需要的时候才开启，因为很耗性能，建议使用即时性的.

**永久配置**

修改配置文件my.cnf或my.ini，在[mysqld]一行下面加入两个配置参数
```
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/hostname-slow.log
long_query_time = 3
```
> log_file 参数为慢查询日志存放的位置，一般这个目录要有 MySQL 的运行帐号的可写权限，一般都将这个目录设置为 MySQL 的数据存放目录；  
> long_query_time=2 中的 2 表示查询超过两秒才记录；  
> 在my.cnf或者 my.ini 中添加 log-queries-not-using-indexes 参数，表示记录下那些没有使用索引的查询。

#### 慢查询日志分析工具Mysqldumpslow
在生产环境中，如果手工分析日志，查找、分析SQL，还是比较费劲的，所以MySQL提供了日志分析工具**mysqldumpslow**。

其功能是, 统计不同慢sql的出现次数(Count)，执行最长时间(Time)，累计总耗费时间(Time)，等待锁的时间(Lock)，发送给客户端的行总数(Rows)，扫描的行总数(Rows)

通过 `mysqldumpslow --help`(windows下需要安装perl环境) 查看操作帮助信息
![help](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/6748468-9c42638eedf72222.png)

- -s，是order的顺序，主要有c（按query次数排序）、t（按查询时间排序）、l（按lock的时间排序）、r （按返回的记录数排序）和 at、al、ar，前面加了a的代表平均数
- -t，是top n的意思，即为返回前面多少条的数据
- -g，后边可以写一个正则匹配模式，大小写不敏感的
- -r：倒序



```
-- 取出耗时最长的两条sql
mysqldumpslow -s t -t 2 /var/lib/mysql/hostname-slow.log
-- 取出查询次数最多，且使用了in关键字的1条sql
mysqldumpslow -s c -t 1 -g 'in' /var/lib/mysql/hostname-slow.log
-- 得到返回记录集最多的10个SQL
mysqldumpslow -s r -t 10 /var/lib/mysql/hostname-slow.log
-- 得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/hostname-slow.log
-- 得到按照时间排序的前10条里面含有左连接的查询语句
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/hostname-slow.log
```
![case](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200722/6748468-c98580c5fd4f39dd.png)
- 出现次数(Count),
- 执行最长时间(Time),
- 累计总耗费时间(Time),
- 等待锁的时间(Lock),
- 发送给客户端的行总数(Rows),
- 扫描的行总数(Rows),
- 用户以及sql语句本身(抽象了一下格式, 比如 limit 1, 20 用 limit N,N 表示).


### 4. 性能优化
#### 4.1 索引优化
1. **全值匹配**我最爱，组合索引中全部条件都有就是全值匹配。
2. **最佳左前缀法则**，比如建立了一个联合索引(a,b,c)，那么其实我们可利用的索引就有(a), (a,b), (a,b,c)
3. **不在索引列上做任何操作**（计算、函数、(自动or手动)类型转换），会导致索引失效而转向全表扫描。
4. **存储引擎不能使用索引中范围条件右边的列**，也就是where a>1 and b=2,后面的条件是不会用到索引的。
5. **尽量使用覆盖索引**(只访问索引的查询(索引列和查询列一致))，减少select回表。
6. is null ,is not null 也无法使用索引（**有可能会使用索引**，取决于数据null值个数，如果较少或较多，那么相对于全表扫描，还是会走索引的）
7. **like "xxxx%"** 是可以用到索引的，like "%xxxx" 则不行(like "%xxx%" 同理)。like以通配符开头('%abc...')索引失效会变成全表扫描的操作.
8. 字符串不加单引号索引失效
9. 少用**or**，用它来连接时会索引失效
    ```
    select id from t where num=10 or num=20    
    -- 可以这样查询：    
    select id from t where num=10    
    union all    
    select id from t where num=20 
    ```
10. <，<=，=，>，>=，BETWEEN，IN 可用到索引，**<>，not in ，!=** 则不行，会导致全表扫描
11. 合理的创建及使用索引（考虑数据的增删情况,索引不是越多越好(**单张表索引不超过 5 个**)，当索引列有大量数据重复时，SQL查询可能不会去利用索引）。
    > 索引可以增加查询效率，但同样也会降低插入和更新的效率，甚至有些情况下会降低查询效率。
    因为 MySQL 优化器在选择如何优化查询时，会根据统一信息，对每一个可以用到的索引来进行评估，以生成出一个最好的执行计划，如果同时有很多个索引都可以用于查询，就会增加 MySQL 优化器生成执行计划的时间，同样会降低查询性能。

> 对于单键索引，尽量选择针对当前query过滤性更好的索引  
> 在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前越好。  
> 在选择组合索引的时候，尽量选择可以能够包含当前query中的where字句中更多字段的索引(**覆盖索引**，减少回表)  
> 尽可能通过分析统计信息和调整query的写法来达到选择合适索引的目的
> 少用Hint强制索引


#### 4.2 查询优化
**永远小表驱动大表（小的数据集驱动大的数据集）**

1. **exists，in的优化**  
详见上节的“[MySQL查询](https://sunsas.gitee.io/doc/#/sunsas/MySQL?id=_314-mysql%e6%9f%a5%e8%af%a2)”

2. **ORDER BY关键字优化**  
- MySQL 支持两种方式的排序，FileSort 和 Index，Index效率高，它指 MySQL 扫描索引本身完成排序，FileSort 效率较低；
- ORDER BY子句，尽量使用 Index 方式排序，避免使用 FileSort 方式排序
- ORDER BY 满足两种情况，会使用Index方式排序；**①ORDER BY语句使用索引最左前列** **②使用where子句与ORDER BY子句条件列组合满足索引最左前列**
- 尽可能在索引列上完成排序操作，遵照索引建的最佳最前缀
- 如果不在索引列上，filesort 有两种算法，mysql就要启动双路排序和单路排序
    - 双路排序：MySQL 4.1之前是使用双路排序,字面意思就是两次扫描磁盘，最终得到数据
    - 单路排序：从磁盘读取查询需要的所有列，按照order by 列在 buffer对它们进行排序，然后扫描排序后的列表进行输出，效率高于双路排序
    

3. **GROUP BY关键字优化**
- group by实质是先排序后进行分组，遵照索引建的最佳左前缀
- 当无法使用索引列，增大 `max_length_for_sort_data` 参数的设置，增大`sort_buffer_size`参数的设置
- where高于having，能写在where限定的条件就不要去having限定了

4. **select语句**  
select语句中尽量不要使用*、count(*),可参考[为什么大家都说SELECT * 效率低？ ](https://mp.weixin.qq.com/s/FD4BsUH0WdjJGuLAWdPK7A)
    1. **不需要的列会增加数据传输时间和网络开销**
        - 用“SELECT * ”数据库需要解析更多的对象、字段、权限、属性等相关内容，在 SQL 语句复杂，硬解析较多的情况下，会对数据库造成沉重的负担。
        - 增大网络开销；* 有时会误带上如log、IconMD5之类的无用且大文本字段，数据传输size会几何增涨。如果DB和应用程序不在同一台机器，这种开销非常明显
        - 即使 mysql 服务器和客户端是在同一台机器上，使用的协议还是 tcp，通信也是需要额外的时间。
    2. **对于无用的大字段，如 varchar、blob、text，会增加 io 操作**  
    准确来说，长度超过 728 字节的时候，会先把超出的数据序列化到另外一个地方，因此读取这条记录会增加一次 io 操作。（MySQL InnoDB）
    3. **失去MySQL优化器“覆盖索引”策略优化的可能性**
    覆盖索引我就不多说了，select * 了基本就不可能用到覆盖索引。
5. **limit优化**
    1. **如果知道查询结果只有一条或者只要最大/最小一条记录，建议用limit 1**。  
        加上limit 1后,只要找到了对应的一条记录,就不会继续向下    扫描了,效率将会大大提高。
        当然，如果name是唯一索引的话，是不必要加上limit 1了，因为limit的存在主要就是为了防止全表扫描，从而提高性能,如果一个语句本身可以预知不用全表扫描，有没有limit ，性能的差别并不大。
    2. **优化limit分页**  
    ```
    -- 当偏移量特别大的时候，查询效率就变得低下。
    --因为Mysql并非是跳过偏移量直接去取后面的数据，
    -- 而是先把偏移量+要取的条数，然后再把前面偏移量这一段的数据抛弃掉再返回的。
    select id，name，age from employee limit 10000，10
    
    -- 方案一 ：返回上次查询的最大记录(偏移量)
    -- 这样可以跳过偏移量，效率提升不少。
    select id，name from employee where id>10000 limit 10.
    
    -- 方案二：order by + 索引，也可以提高查询效率
    select id，name from employee order by id  limit 10000，10
    
    -- 方案三：在业务允许的情况下限制页数：
    ```
6. **优先使用Inner join**，如果是left join，左边表结果尽量小。  
优先使用内连接，因为MySQL对内连接有优化。


#### 4.3 数据类型优化
MySQL 支持的数据类型非常多，选择正确的数据类型对于获取高性能至关重要。
- 更小的通常更好：一般情况下，应该尽量使用可以正确存储数据的最小数据类型。
    - 将字符串转换成数字类型存储,如:将 IP 地址转换成整形数据
        ```
        MySQL 提供了两个方法来处理 ip 地址  
        inet_aton 把 ip 转为无符号整型 (4-8 位)  
        inet_ntoa 把整型的 ip 转为地址 
        ```   
    - 对于非负型的数据 (如自增 ID,整型 IP) 来说,要优先使用无符号整型来存储  
    无符号相对于有符号可以多出一倍的存储空间
        ```
        SIGNED INT -2147483648~2147483647
        UNSIGNED INT 0~4294967295
        ```
- 简单就好：简单的数据类型通常需要更少的CPU周期。例如，整数比字符操作代价更低，因为字符集和校对规则（排序规则）使字符比较比整型比较复杂。
- **尽量避免NULL：通常情况下最好指定列为NOT NULL**
索引 NULL 列需要额外的空间来保存，所以要占用更多的空间。  
进行比较和计算时要对 NULL 值做特别的处理
- 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。    
这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。
- 尽可能的使用 varchar 代替 char ，因为首先变长字段存储空间小，可以节省存储空间，    
其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。
- **避免使用 TEXT,BLOB 数据类型**，最常见的 TEXT 类型可以存储 64k 的数据
    - <font color=red >建议把 BLOB 或是 TEXT 列分离到单独的扩展表中</font>,MySQL 内存临时表不支持 TEXT、BLOB 这样的大数据类型，如果查询中包含这样的数据，在排序等操作时，就不能使用内存临时表，必须使用磁盘临时表进行。而且对于这种数据，MySQL 还是要进行二次查询，会使 sql 性能变得很差，但是不是说一定不能使用这样的数据类型。  
    如果一定要使用，建议把 BLOB 或是 TEXT 列分离到单独的扩展表中，查询时一定不要使用 select * 而只需要取出必要的列，不需要 TEXT 列的数据时不要对该列进行查询。
    - <font color=red >TEXT 或 BLOB 类型只能使用前缀索引</font>,因为MySQL[1] 对索引字段长度是有限制的，所以 TEXT 类型只能使用前缀索引，并且 TEXT 列上是不能有默认值的
- **避免使用 ENUM 类型**  
修改 ENUM 值需要使用 ALTER 语句  
ENUM 类型的 ORDER BY 操作效率低，需要额外操作  
禁止使用数值作为 ENUM 的枚举值  
- **使用 TIMESTAMP(4 个字节) 或 DATETIME 类型 (8 个字节) 存储时间**
TIMESTAMP 存储的时间范围 1970-01-01 00:00:01 ~ 2038-01-19-03:14:07  
TIMESTAMP 占用 4 字节和 INT 相同，但比 INT 可读性高  
**超出 TIMESTAMP 取值范围的使用 DATETIME 类型存储**  
<font color=red>不要用字符串存储日期型的数据</font>
    - 无法用日期函数进行计算和比较
    - 用字符串存储日期要占用更多的空间

> 参考
>
> [MySQL高级 之 explain执行计划详解](https://blog.csdn.net/wuseyukui/article/details/71512793?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)  
> [MySQL性能优化（七）-- 慢查询](https://www.jianshu.com/p/13311c49bc97)  
> [MySQL高级知识（十一）——Show Profile](https://www.cnblogs.com/developer_chan/p/9231761.html)  
> [为什么大家都说SELECT * 效率低？ ](https://mp.weixin.qq.com/s/FD4BsUH0WdjJGuLAWdPK7A)  
> [一手好 SQL 是如何炼成的？](https://mp.weixin.qq.com/s/BPGe8BNZEDgxbJ2dVXzdXg)  
> [MySQL高性能优化规范建议](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485117&idx=1&sn=92361755b7c3de488b415ec4c5f46d73&chksm=cea24976f9d5c060babe50c3747616cce63df5d50947903a262704988143c2eeb4069ae45420&token=79317275&lang=zh_CN%23rd)  
> [后端程序员必备：书写高质量SQL的30条建议 ](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486461&idx=1&sn=60a22279196d084cc398936fe3b37772&chksm=cea24436f9d5cd20a4fa0e907590f3e700d7378b3f608d7b33bb52cfb96f503b7ccb65a1deed&token=1987003517&lang=zh_CN%23rd)



---


## 3.2.2 MySQL分区，分表，分库


### 1. 分区
一般情况下我们创建的表对应一组存储文件，使用MyISAM存储引擎时是一个.MYI和.MYD文件，使用Innodb存储引擎时是一个.ibd和.frm（表结构）文件。

当数据量较大时（一般千万条记录级别以上），MySQL的性能就会开始下降，这时我们就需要将数据分散到多组存储文件，保证其单个文件的执行效率。
- 逻辑数据分割
- 提高单一的写和读应用速度
- 提高分区范围读查询的速度
- 分割数据能够有多个不同的物理文件路径
- 高效的保存历史数据


```
-- 查看当前数据库是否支持分区
-- MySQL5.6以及之前版本：
SHOW VARIABLES LIKE '%partition%';
-- MySQL5.6（在其中找到partition）：
show plugins;
```

#### 1.1 分区类型
- **RANGE分区**：基于属于一个给定连续区间的列值，把多行分配给分区。mysql将会根据指定的拆分策略，,把数据放在不同的表文件上。相当于在文件上,被拆成了小块.但是,对外给客户的感觉还是一张表，透明的。

    按照 range 来分，就是每个库一段连续的数据，这个一般是按比如时间范围来的，比如交易表啊，销售表啊等，可以根据年月来存放数据。可能会产生热点问题，**大量的流量都打在最新的数据上了**。好处是扩容的时候很简单。
- **LIST分区**：类似于按RANGE分区，每个分区必须明确定义。它们的主要区别在于，LIST分区中每个分区的定义和选择是基于某列的值从属于一个值列表集中的一个值，而RANGE分区是从属于一个连续区间值的集合。
- **HASH分区**：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。
- **KEY分区**：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。


**为什么大部分互联网还是更多的选择自己分库分表来水平扩展，而不是分区？**
- 分区表，分区键设计不太灵活，如果不走分区键，很容易出现全表锁
- 一旦数据并发量上来，如果在分区表实施关联，就是一个灾难
- 自己分库分表，自己掌控业务场景与访问模式，可控。分区表，研发写了一个sql，都不确定mysql是怎么玩的，不太可控

### 2. MySQL分表

#### 2.1 垂直拆分
垂直分表，通常是按照业务功能的使用频次，把主要的、热门的**字段**放在一起做为主要表。然后把不常用的，按照各自的业务属性进行聚集，拆分到不同的次要表中；主要表和次要表的关系一般都是一对一的。

#### 水平拆分(数据分片)
单表的容量不超过500W，否则建议水平拆分。是把一个表复制成同样表结构的不同表，然后把数据按照一定的规则划分，分别存储到这些表中，从而保证单表的容量不会太大，提升性能；当然这些结构一样的表，可以放在一个或多个数据库中。
> 数据分片指按照某个维度将存放在单一数据库中的数据分散地存放至多个数据库或表中。数据分片的有效手段就是对关系型数据库进行分库和分表。
>
> 区别于分区的是，分区一般都是放在单机里的，用的比较多的是时间范围分区，方便归档。只不过分库分表需要代码实现，分区则是mysql内部实现。分库分表和分区并不冲突，可以结合使用。

**水平分割的几种方法**：
- **使用MD5哈希**，做法是对UID进行md5加密，然后取前几位（我们这里取前两位），然后就可以将不同的UID哈希到不同的用户表（user_xx）中了。
- 还可根据**时间**放入不同的表，比如：article_201601，article_201602。
- 按热度拆分，高点击率的词条生成各自的一张表，低热度的词条都放在一张大表里，待低热度的词条达到一定的贴数后，再把低热度的表单独拆分成一张表。
- 根据ID的值放入对应的表，第一个表user_0000，第二个100万的用户数据放在第二 个表user_0001中，随用户增加，直接添加用户表就行了。
- **取模**。这种方法先计算Key的哈希值，再对设备数量取模（整型的Key也可直接用Key取模），假设有N台设备，编号为0~N-1，通过Hash(Key)%N就可以确定数据所在的设备编号。这种方法实现也非常简单，数据分布和负载也会比较均匀，可以新增任何数量的设备来扩容。主要的问题是扩容的时候，会产生大量的数据迁移，比如从N台设备扩容到N+1台，绝大部分的数据都要在设备间进行迁移。


#### 3. MySQL分库
分表能够解决单表数据量过大带来的查询效率下降的问题，但是，却无法给数据库的并发处理能力带来质的提升。面对高并发的读写访问，当数据库master服务器无法承载**写**操作压力时，不管如何扩展slave服务器，此时都没有意义了。因此，我们必须换一种思路，**对数据库进行拆分**，从而提高数据库写入能力，这就是所谓的分库。


一个库里表太多了，导致了海量数据，系统性能下降，把原本存储于一个库的表拆分存储到多个库上， 通常是将表**按照功能模块、关系密切程度**划分出来，部署到不同库上。

**优点：**
- 减少增量数据写入时的锁对查询的影响
- 由于单表数量下降，常见的查询操作由于减少了需要扫描的记录，使得单表单次查询所需的检索行数变少，减少了磁盘IO，时延变短

**缺点**：无法解决单表数据量太大的问题


### 4. 分库分表后的难题
- **事务问题**  
在执行分库分表之后，由于数据存储到了不同的库上，数据库事务管理出现了困难。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价；如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。
- **跨库跨表的join问题**  
在执行了分库分表之后，难以避免会将原本逻辑关联性很强的数据划分到不同的表、不同的库上，这时，表的关联操作将受到限制，我们无法join位于不同分库的表，也无法join分表粒度不同的表，结果原本一次查询能够完成的业务，可能需要多次查询才能完成。
- **额外的数据管理负担和数据运算压力**  
额外的数据管理负担，最显而易见的就是数据的定位问题和数据的增删改查的重复执行问题，这些都可以通过应用程序解决，但必然引起额外的逻辑运算，例如，对于一个记录用户成绩的用户数据表userTable，业务要求查出成绩最好的100位，在进行分表之前，只需一个order by语句就可以搞定，但是在进行分表之后，将需要n个order by语句，分别查出每一个分表的前100名用户数据，然后再对这些数据进行合并计算，才能得出结果。

> 参考 
> 
> [MySql从一窍不通到入门（五）Sharding：分表、分库、分片和分区](https://blog.csdn.net/KingCat666/article/details/78324678?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)  
> [分库分表带来的完整性和一致性问题](https://www.cnblogs.com/aigongsi/archive/2013/01/25/2875731.html)



--- 


## 3.2.3 MySQL主从  
### 1. 简介
mysql的主从复制，是用来建立一个和主数据库完全一样的数据库环境，称为从数据库，主数据库一般是实时的业务数据操作，从数据库常用的读取为主。

优点：

1. 可以作为备用数据库进行操作，当主数据库出现故障之后，从数据库可以替代主数据库继续工作，不影响业务流程
2. **读写分离**，将读和写应用在不同的数据库与服务器上。一般读写的数据库环境配置为，一个写入的数据库，一个或多个读的数据库，各个数据库分别位于不同的服务器上，充分利用服务器性能和数据库性能；当然，其中会涉及到如何保证读写数据库的数据一致，这个就可以利用主从复制技术来完成。
3. **吞吐量较大**，业务的查询较多，并发与负载较大。

### 2. 复制步骤
![MySQL主从复制步骤](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200724/MySQL%E5%A4%8D%E5%88%B6.png)
1. master将改变记录到二进制日志（binary log）。这些记录过程叫做二进制日志事件，binary log events；
2. salve 将 master 的 binary log events 拷贝到它的中继日志（relay log）;
3. slave 重做中继日志中的事件，将改变应用到自己的数据库中。MySQL 复制是异步且是串行化的。

### 3. 搭建主从复制
略，和开发没啥关系，如果真要搭建可以参考[mysql主从复制的理解和搭建](https://blog.csdn.net/li_lening/article/details/81878163?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)


> 参考
>
> [MySQL主备、主从、读写分离详解](https://blog.csdn.net/qq_40378034/article/details/91125768)  
> [mysql主从复制的理解和搭建](https://blog.csdn.net/li_lening/article/details/81878163?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)