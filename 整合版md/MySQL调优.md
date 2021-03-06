

## MySQL调优之索引：索引的失效与优化

#### **索引条件使用函数操作或运算**

看到这个标题有的哥们就说了：这个我知道，给索引条件加函数Mysql不会走索引。那么是为什么呢？我们来分析一下，将第五章的建表语句稍微改动下：

```text
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16)  NOT NULL,
  `name` varchar(16)  NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128)  DEFAULT NULL,
  `update_time` varchar(20)  DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `update_index` (`update_time`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```

> 添加update_time字段 ，并给此字段添加索引

现在我们要统计12月的记录一共有多少条记录SQL语句如下：

```text
select count(*) from t where month(update_time)=12;
```

![img](https://picb.zhimg.com/v2-339b6e65b18b3788e63a5e2745b7ad09_b.png)

看运行结果竟然用到了update_index这个索引，惊奇不惊奇，意外不意外？然后我们再细分析一下，实际上这个函数已经破坏了索引的原本的结构，在这种情况下SQL优化器可以选择唯一索引，也可以选择索引update_index，然鹅优化器在经过一系列运算后发现还是用update_index划算，那么他就还是走了update_index。但是我们看一下执行计划的rows是10004条（本表一共有10000条记录，满足条件的有9000多条），也就是说虽然走了update_index索引，却走了全表扫描(这就论证了，业内所传的函数条件导致不走索引的结论不是很严谨，哈哈哈...）。再看看我们使用这样一条SQL的执行计划：

```text
EXPLAIN select * from t where month(update_time)=12;
```

![img](https://pic2.zhimg.com/v2-19d9ad21f5491cc949f1119bb2809afc_b.png)

当需要查询非索引字段时Mysql果断放弃了update_index而使用了全表扫描。

说完函数我们说说索引条件的运算，这个就比较好理解的了，举个例子：

```text
select * from t where id + 1 = 10
```

这个条件并不会破坏索引原有结构的有序性，但是Mysql还是不能通过索引一次性找到这条记录，需要扫描全表

![img](https://picb.zhimg.com/v2-f12abcd20fd15d8421c345a4520f72c3_b.jpg)

#### **隐式转换**

我们这里抛出一个sql大家猜猜看看是什么结果就知道了：

```text
select “10” > 9
```

我这里就不截图了，它查询结果是1，也就是说MySQL中，字符串和数字做比较的话，是将字符串转换成数字。那么它的执行SQL是这样的：

```text
select CAST("10" AS signed int) > 9
```

假如我们将这种使用方式在where条件中，就会破坏到当前条件索引，平时在写sql时注意：一般都是宽类型向窄类型转换，尽量将需要转换的类型放在表达式后面。

#### **隐式字符编码转换**

这里大家需要注意的是：多表关联查询时，被驱动表的字符集与其他表的不一致时，被驱动的表的关联索引字段会加上字符转换函数CONVERT()，又会导致全表扫描。所以在数据库表设计和创建之初需要有统一规范的建模流程，避免这样的惨剧发生。

- 都有哪些调优方法
  - 覆盖索引优化查询
    - 从辅助索引中查询得到记录，而不需要通过聚族索引查询获得，MySQL 中将其称为覆盖索引
    - SELECT COUNT(*) 时，如果不存在辅助索引，此时会通过查询聚族索引来统计行数，如果此时正好存在一个辅助索引，则会通过查询辅助索引来统计行数，减少 I/O 操作。
  - 自增字段作主键优化查询
    - 同一个叶子节点内的各个数据是按主键顺序存放的
  - 前缀索引优化
    - 前缀索引是有一定的局限性的，例如 order by 就无法使用前缀索引，无法把前缀索引用作覆盖索引。
  - 防止索引失效
    - 最左匹配原则
    - 如果查询条件中使用 or，且 or 的前后条件中有一个列没有索引，那么涉及的索引都不会被使用到。

#### 索引查询失效的几个情况：

1、like 以%开头，索引无效；当like前缀没有%，后缀有%时，索引有效。

2、or语句前后没有同时使用索引。当or左右查询字段只有一个是索引，该索引失效，只有当or左右查询字段均为索引时，才会生效

3、组合索引，不是使用第一列索引，索引失效。

4、数据类型出现隐式转化。如varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。

5、在索引列上使用 IS NULL 或 IS NOT NULL操作。索引是不索引空值的，所以这样的操作不能使用索引，可以用其他的办法处理，例如：数字类型，判断大于0，字符串类型设置一个默认值，判断是否等于默认值即可。

6、在索引字段上使用not，<>，!=。不等于操作符是永远不会用到索引的，因此对它的处理只会产生全表扫描。 优化方法： key<>0 改为 key>0 or key<0。

7、对索引字段进行计算操作、字段上使用函数。（索引为 emp(ename,empno,sal)）

8、当全表扫描速度比索引速度快时，mysql会使用全表扫描，此时索引失效。



索引失效分析工具：

可以使用explain命令加在要分析的sql语句前面，在执行结果中查看key这一列的值，如果为NULL，说明没有使用索引。







## SQL语句逻辑相同，性能却差异巨大（索引失效等等）

- 条件字段函数操作

  - > mysql> select count(*) from tradelog where month(t_modified)=7;
    >
    > 如果对字段做了函数计算，就用不上索引了，这是 MySQL 的规定。
    >
    > key="t_modified"表示的是，使用了 t_modified 这个索引
    > rows=100335，说明这条语句扫描了整个索引的所有值；
    > Extra 字段的 Using index，表示的是使用了覆盖索引
    > 也就是说，由于在 t_modified 字段加了 month() 函数操作，导致了全索引扫描
    >
    > 
    >
    > 如果你的 SQL 语句条件用的是 where t_modified='2018-7-1’的话，引擎就会快速定位到 t_modified='2018-7-1’需要的结果。
    > 	实际上，B+ 树提供的这个快速定位能力，来源于同一层兄弟节点的有序性。
    > 	也就是说，对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。
    >
    > 
    >
    > 对于 select * from tradelog where id + 1 = 10000 这个 SQL 语句，这个加 1 操作并不会改变有序性，但是 MySQL 优化器还是不能用 id 索引快速定位到9999 这一行。所以，需要你在写 SQL 语句的时候，手动改写成 where id = 10000 -1 才可以。

- 隐式类型转换

  - > mysql> select * from tradelog where tradeid=110717;
    > tradeid 的字段类型是 varchar(32)，而输入的参数却是整型，所以需要做类型转换。
    >
    > 
    >
    > 在 MySQL 中，字符串和数字做比较的话，是将字符串转换成数字。

- 隐式字符编码转换

  - 两个表的字符集不同，一个是 utf8，一个是 utf8mb4，所以做表连接查询的时候用不上关联字段的索引。这个回答，也是通常你搜索这个问题时会得到的答案。
  - 为什么字符集不同就用不上索引呢？
    - 字符集 utf8mb4 是 utf8 的超集，所以当这两个类型的字符串在做比较的时候，MySQL 内部的操作是，先把 utf8 字符串转成 utf8mb4 字符集，再做比较。
      - CONVERT() 函数，在这里的意思是把输入的字符串转成 utf8mb4 字符集。
      - 这就再次触发了我们上面说到的原则：对索引字段做函数操作，优化器会放弃走树搜索功能。
  - 字符集不同只是条件之一，连接过程中要求在被驱动表的索引字段上加函数操作，是直接导致对被驱动表做全表扫描的原因。









## 数据库参数设置优化，失之毫厘差之千里

- InnoDB 中的数据和索引缓存，如果设置过大，就会引发 SWAP 页交换。还有数据写入到磁盘也不是越快越好，我们期望的是在高并发时，数据能均匀地写入到磁盘中，从而避免 I/O 性能瓶颈。

- 在执行查询 SQL 语句时，会涉及到两个缓存

  - Query Cache  它缓存的是 SQL 语句和对应的结果集

    - 我们可以通过设置合适的 query_cache_min_res_unit 来减少碎片

    - 通过以下公式计算所得

      ```
      （query_cache_size - Qcache_free_memory）/ Qcache_queries_in_cache
      
      Qcache_free_memory 和 Qcache_queries_in_cache 的值可以通过以下命令查询：show status like 'Qcache%'
      ```

    - Query Cache 虽然可以优化查询操作，但也仅限于不常修改的数据，如果一张表数据经常进行新增、更新和删除操作，则会造成 Query Cache 的失效率非常高，从而导致频繁地清除 Cache 中的数据，给系统增加额外的性能开销。

    - Qcache_hits，该值表示缓存命中率。如果缓存命中率特别低的话，我们还可以通过 query_cache_size = 0或者 query_cache_type 来关闭查询缓存。

  - InnoDB 存储引擎参数设置调优

    - 经过了 Query Cache 缓存之后，还会使用到存储引擎中的 Buffer 缓存。不同的存储引擎，使用的 Buffer 也是不一样的。
    - InnoDB Buffer Pool（简称 IBP）是 InnoDB 存储引擎的一个缓冲池
    - InnoDB 表空间缓存越多，MySQL访问物理磁盘的频率就越低
    - innodb_buffer_pool_size
      - MySQL 推荐配置 IBP 的大小为服务器物理内存的 80%。
      - 但如果我们将 IBP 的大小设置为物理内存的80% 以后，发现命中率还是很低，此时我们就应该考虑扩充内存来增加 IBP 的大小。
    - innodb_buffer_pool_instances
      - InnoDB 中的 IBP 缓冲池被划分为了多个实例，
      - 将缓冲池划分为单独的实例可以减少不同线程读取和写入缓存页面时的争用，从而提高系统的并发性
      - 建议 innodb_buffer_pool_instances 的大小不超过innodb_read_io_threads +innodb_write_io_threads 之和
    - innodb_read_io_threads / innodb_write_io_threads
      - 默认情况下，MySQL 后台线程包括了主线程、IO 线程、锁线程以及监控线程等
    - innodb_log_file_size
      - MySQL的InnoDB 存储引擎使用一个指定大小的Redo log空间（一个环形的数据结构）。Redo log的空间通过innodb_log_file_size和innodb_log_files_in_group（默认2）参数来调节
      - 理论上来说，innodb_log_file_size 设置得越大，缓冲池中需要的检查点刷新活动就越少，从而节省磁盘 I/O
      - 在大多数情况下，我们将日志文件大小设置为 1GB 就足够了。
    - innodb_log_buffer_size
      - 这个参数决定了 InnoDB 重做日志缓冲池的大小，默认值为 8MB
      - 我们可以通过增大该参数来减少写入磁盘操作，从而提高并发时的事务性能。
    - innodb_flush_log_at_trx_commit
      - 这个参数可以控制重做日志从缓存写入文件刷新到磁盘中的策略
        - 当设置该参数为 0 时，InnoDB 每秒种就会触发一次缓存日志写入到文件中并刷新到磁盘的操作
        - 当设置该参数为 1 时，则表示每次事务的 redo log 都会直接持久化到磁盘中
        - 当设置该参数为 2 时，每次事务的 redo log 都会直接写入到文件中，再将文件刷新到磁盘。
      - 在一些对数据安全性要求比较高的场景中，显然该值需要设置为 1；而在一些可以容忍数据库崩溃时丢失 1s 数据的场景中，我们可以将该值设置为 0 或 2，这样可以明显地减少日志同步到磁盘的 I/O 操作。







## 只查一行的语句，也执行这么慢

- 第一类：查询长时间不返回

  - 等 MDL 锁

    - MySQL 5.7版本修改了 MDL 的加锁策略，所以就不能复现这个场景了。

    - > mysql> select * from t where id=1;
      > 	一般碰到这种情况的话，大概率是表 t 被锁住了。接下来分析原因的时候，一般都是首先执行一下 show processlist 命令，看看当前语句处于什么状态。

    - 通过查询 sys.schema_table_lock_waits 这张表，我们就可以直接找出造成阻塞的process id，把这个连接用 kill 命令断开即可。

  - 等 flush

    - > MySQL 里面对表做 flush操作的用法，一般有以下两个：
      > 	flush tables t with read lock;
      > flush tables with read lock;
      > 	这两个 flush 语句，如果指定表 t 的话，代表的是只关闭表 t；如果没有指定具体的表名，则表示关闭 MySQL 里所有打开的表。但是正常这两个语句执行起来都很快，除非它们也被别的线程堵住了。

  - 等行锁

    - > mysql> select * from t where id=1 lock in share mode;
      > 由于访问 id=1 这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的 select 语句就会被堵住。

    - 如果你用的是 MySQL 5.7 版本，可以通过 sys.innodb_lock_waits 表查到。

    - > mysql> select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G

    - 4 号线程是造成堵塞的罪魁祸首

      - 不应该显示“KILL QUERY 4”。这个命令表示停止 4 号线程当前正在执行的语句，而这个方法其实是没有用的。因为占有行锁的是 update 语句，这个语句已经是之前执行完成了的，现在执行 KILL QUERY，无法让这个事务去掉 id=1 上的行锁。
      - 实际上，KILL 4 才有效，也就是说直接断开这个连接。这里隐含的一个逻辑就是，连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了 id=1 上的行锁。

- 第二类：查询慢

  - 字段上没有索引

    - 这里为了把所有语句记录到 slow log 里，我在连接后先执行了 set long_query_time=0，将慢查询日志的时间阈值设置为 0。

  - > mysql> select * from t where id=1；
    >
    > 虽然扫描行数是 1，但执行时间却长达 800 毫秒。
    >
    > 
    >
    > select * from t where id=1 lock in share mode，执行时扫描行数也是 1 行，执行时间是 0.2 毫秒。
    > 第一个语句的查询结果里 c=1，带 lock in share mode 的语句返回的是 c=1000001

    - session A 先用 start transaction with consistent snapshot 命令启动了一个事务，之后 session B 才开始执行 update 语句。
      - session B 更新完 100 万次，生成了 100 万个回滚日志 (undo log)
      - 带 lock in share mode 的 SQL 语句，是当前读，因此会直接读到 1000001 这个结果，所以速度很快；
      - 而 select * from t where id=1 这个语句，是一致性读，因此需要从1000001 开始，依次执行 undo log，执行了 100 万次以后，才将 1 这个结果返回
      - undo log 里记录的其实是“把 2 改成 1”，“把 3 改成 2”这样的操作逻辑

- 这个语句序列是怎么加锁的呢？加的锁又是什么时候释放呢？

  - > 1 begin;
    > 2 select * from t where c=5 for update;
    > 3 commit;

  - RR隔离级别下

    - 为保证binlog记录顺序，非索引更新会锁住全表记录，且事务结束前不会对不符合条件记录有逐步释放的过程。

  - RC隔离级别下

    - 对非索引字段更新，有个锁全表记录的过程，不符合条件的会及时释放行锁，不必等事务结束时释放；
    - 而直接用索引列更新，只会锁索引查找值和行。c=5 这一行的行锁，还是会等到 commit 的时候才释放的。

- 一般说全表扫描默认是指“扫瞄主键索引”



## SQL语句执行得很慢的原因



### 分类讨论

一条 SQL 语句执行的很慢，那是每次执行都很慢呢？还是大多数情况下是正常的，偶尔出现很慢呢？所以我觉得，我们还得分以下两种情况来讨论。

1、大多数情况是正常的，只是偶尔会出现很慢的情况。

2、在数据量不变的情况下，这条SQL语句一直以来都执行的很慢。

针对这两种情况，我们来分析下可能是哪些原因导致的。



### 针对偶尔很慢的情况

一条 SQL 大多数情况正常，偶尔才能出现很慢的情况，针对这种情况，我觉得这条SQL语句的书写本身是没什么问题的，而是其他原因导致的，那会是什么原因呢？

#### 数据库在刷新脏页我也无奈啊

当我们要往数据库插入一条数据、或者要更新一条数据的时候，我们知道数据库会在**内存**中把对应字段的数据更新了，但是更新之后，这些更新的字段并不会马上同步持久化到**磁盘**中去，而是把这些更新的记录写入到 redo log 日记中去，等到空闲的时候，在通过 redo log 里的日记把最新的数据同步到**磁盘**中去。

不过，redo log 里的容量是有限的，如果数据库一直很忙，更新又很频繁，这个时候 redo log 很快就会被写满了，这个时候就没办法等到空闲的时候再把数据同步到磁盘的，只能暂停其他操作，全身心来把数据同步到磁盘中去的，而这个时候，**就会导致我们平时正常的SQL语句突然执行的很慢**，所以说，数据库在在同步数据到磁盘的时候，就有可能导致我们的SQL语句执行的很慢了。

#### 拿不到锁我能怎么办

这个就比较容易想到了，我们要执行的这条语句，刚好这条语句涉及到的**表**，别人在用，并且加锁了，我们拿不到锁，只能慢慢等待别人释放锁了。或者，表没有加锁，但要使用到的某个一行被加锁了，这个时候，我也没办法啊。

如果要判断是否真的在等待锁，我们可以用 **show processlist**这个命令来查看当前的状态哦，这里我要提醒一下，有些命令最好记录一下，反正，我被问了好几个命令，都不知道怎么写，呵呵。

下来我们来访分析下第二种情况，我觉得第二种情况的分析才是最重要的



### 针对一直都这么慢的情况

如果在数据量一样大的情况下，这条 SQL 语句每次都执行的这么慢，那就就要好好考虑下你的 SQL 书写了，下面我们来分析下哪些原因会导致我们的 SQL 语句执行的很不理想。

我们先来假设我们有一个表，表里有下面两个字段,分别是主键 id，和两个普通字段 c 和 d。

```text
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

#### 没用到索引

没有用上索引，我觉得这个原因是很多人都能想到的，例如你要查询这条语句

```text
select * from t where 100 <c and c < 100000;
```

**字段没有索引**

刚好你的 c 字段上没有索引，那么抱歉，只能走全表扫描了，你就体验不会索引带来的乐趣了，所以，这回导致这条查询语句很慢。

**字段有索引，但却没有用索引**

好吧，这个时候你给 c 这个字段加上了索引，然后又查询了一条语句

```text
select * from t where c - 1 = 1000;
```

我想问大家一个问题，这样子在查询的时候会用索引查询吗？

答是不会，如果我们在字段的左边做了运算，那么很抱歉，在查询的时候，就不会用上索引了，所以呢，大家要注意这种**字段上有索引，但由于自己的疏忽，导致系统没有使用索引**的情况了。

正确的查询应该如下

```text
select * from t where c = 1000 + 1;
```

有人可能会说，右边有运算就能用上索引？难道数据库就不会自动帮我们优化一下，自动把 c - 1=1000 自动转换为 c = 1000+1。

不好意思，确实不会帮你，所以，你要注意了。

**函数操作导致没有用上索引**

如果我们在查询的时候，对字段进行了函数操作，也是会导致没有用上索引的，例如

```text
select * from t where pow(c,2) = 1000;
```

这里我只是做一个例子，假设函数 pow 是求 c 的 n 次方，实际上可能并没有 pow(c,2)这个函数。其实这个和上面在左边做运算也是很类似的。

所以呢，一条语句执行都很慢的时候，可能是该语句没有用上索引了，不过具体是啥原因导致没有用上索引的呢，你就要会分析了，我上面列举的三个原因，应该是出现的比较多的吧。

#### 数据库自己选错索引了

我们在进行查询操作的时候，例如

```text
select * from t where 100 < c and c < 100000;

```

我们知道，主键索引和非主键索引是有区别的，主键索引存放的值是**整行字段的数据**，而非主键索引上存放的值不是整行字段的数据，而且存放**主键字段的值**。不大懂的可以看我这篇文章：[面试小知识：MySQL索引相关](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/RemJcqPIvLArmfWIhoaZ1g)    里面有说到主键索引和非主键索引的区别

也就是说，我们如果走 c 这个字段的索引的话，最后会查询到对应主键的值，然后，再根据主键的值走主键索引，查询到整行数据返回。

好吧扯了这么多，其实我就是想告诉你，就算你在 c 字段上有索引，系统也并不一定会走 c 这个字段上的索引，而是有可能会直接扫描扫描全表，找出所有符合 100 < c and c < 100000 的数据。

**为什么会这样呢？**

其实是这样的，系统在执行这条语句的时候，会进行预测：究竟是走 c 索引扫描的行数少，还是直接扫描全表扫描的行数少呢？显然，扫描行数越少当然越好了，因为扫描行数越少，意味着I/O操作的次数越少。

如果是扫描全表的话，那么扫描的次数就是这个表的总行数了，假设为  n；而如果走索引 c 的话，我们通过索引 c  找到主键之后，还得再通过主键索引来找我们整行的数据，也就是说，需要走两次索引。而且，我们也不知道符合 100 c < and c <  10000 这个条件的数据有多少行，万一这个表是全部数据都符合呢？这个时候意味着，走 c 索引不仅扫描的行数是  n，同时还得每行数据走两次索引。



**所以呢，系统是有可能走全表扫描而不走索引的。那系统是怎么判断呢？**

**判断来源于系统的预测，也就是说，如果要走 c 字段索引的话，系统会预测走 c 字段索引大概需要扫描多少行。如果预测到要扫描的行数很多，它可能就不走索引而直接扫描全表了。**

那么问题来了，**系统是怎么预测判断的呢？**

系统是通过**索引的区分度**来判断的，一个索引上不同的值越多，意味着出现相同数值的索引越少，意味着索引的区分度越高。我们也把区分度称之为**基数**，即区分度越高，基数越大。所以呢，基数越大，意味着符合 100 < c and c < 10000 这个条件的行数越少。

所以呢，一个索引的基数越大，意味着走索引查询越有优势。

**那么问题来了，怎么知道这个索引的基数呢？**

系统当然是不会遍历全部来获得一个索引的基数的，代价太大了，索引系统是通过遍历部分数据，也就是通过**采样**的方式，来预测索引的基数的。

**扯了这么多，重点的来了**，居然是采样，那就有可能出现**失误**的情况，也就是说，c 这个索引的基数实际上是很大的，但是采样的时候，却很不幸，把这个索引的基数预测成很小。例如你采样的那一部分数据刚好基数很小，然后就误以为索引的基数很小。**然后就呵呵，系统就不走 c 索引了，直接走全部扫描了**。

所以呢，说了这么多，得出结论：**由于统计的失误，导致系统没有走索引，而是走了全表扫描**，而这，也是导致我们 SQL 语句执行的很慢的原因。

> 这里我声明一下，系统判断是否走索引，扫描行数的预测其实只是原因之一，这条查询语句是否需要使用使用临时表、是否需要排序等也是会影响系统的选择的。

不过呢，我们有时候也可以通过强制走索引的方式来查询，例如

```text
select * from t force index(a) where c < 100 and c < 100000;
```

我们也可以通过

```text
show index from t;
```

来查询索引的基数和实际是否符合，如果和实际很不符合的话，我们可以重新来统计索引的基数，可以用这条命令

```text
analyze table t;
```

来重新统计分析。

**既然会预测错索引的基数，这也意味着，当我们的查询语句有多个索引的时候，系统有可能也会选错索引哦**，这也可能是 SQL 执行的很慢的一个原因。

好吧，就先扯这么多了，你到时候能扯出这么多，我觉得已经很棒了，下面做一个总结。

总结

以上是我的总结与理解，最后一个部分，我怕很多人不大懂**数据库居然会选错索引**，所以我详细解释了一下，下面我对以上做一个总结。

**一个 SQL 执行的很慢，我们要分两种情况讨论：**

1、大多数情况下很正常，偶尔很慢，则有如下原因

(1)、数据库在刷新脏页，例如 redo log 写满了需要同步到磁盘。

(2)、执行的时候，遇到锁，如表锁、行锁。

2、这条 SQL 语句一直执行的很慢，则有如下原因。

(1)、没有用上索引：例如该字段没有索引；由于对字段进行运算、函数操作导致无法用索引。

(2)、数据库选错了索引。

----------------------------------6月11号补充-------------------------------------

很多人说到了查看执行计划，是的，在我们SQL语句实际执行当然，如果要判断这条语句究竟是什么原因，那么我们就需要查看执行计划(explain)了，执行计划会给出系统究竟选择了什么样的一种方式来执行





## MySQL调优之SQL语句：如何写出高性能SQL语句（sql优化）

- **慢 SQL 语句的几种常见诱因**

  - 无索引、索引失效导致慢查询
  - 锁等待
  - 不恰当的 SQL 语句
    - 使用不恰当的 SQL 语句也是慢 SQL 最常见的诱因之一。例如，习惯使用 <SELECT *>，<SELECT COUNT(*)> SQL 语句，在大数据表中使用 <LIMIT M,N> 分页查询，以及对非索引字段进行排序等等。

- 优化 SQL 语句的步骤

  - **通过 EXPLAIN 分析 SQL 执行计划**
    - 上述通过 EXPLAIN 分析执行计划，仅仅是停留在分析 SQL 的外部的执行情况，如果我们想要深入到 MySQL 内核中，从执行线程的状态和时间来分析的话，这个时候我们就可以选择 Profile。
  - **通过 Show Profile 分析 SQL 执行性能**
    - Profile 除了可以分析执行线程的状态和时间，还支持进一步选择 ALL、CPU、MEMORY、BLOCK IO、CONTEXT SWITCHES 等类型来查询 SQL 语句在不同系统资源上所消耗的时间。
    - SELECT COUNT(*) FROM `order`; SQL 语句在 Sending data 状态所消耗的时间最长，这是因为在该状态下，MySQL 线程开始读取数据并返回到客户端，此时有大量磁盘 I/O 操作。

- 常用的 SQL 优化

  - **利用子查询优化分页查询**

    - 通常我们是使用 <LIMIT M,N> + 合适的 order by 来实现分页查询，这种实现方式在没有任何索引条件支持的情况下，需要做大量的文件排序操作（file sort），性能将会非常得糟糕。如果有对应的索引，通常刚开始的分页查询效率会比较理想，但越往后，分页查询的性能就越差。

    - 这是因为我们在使用 LIMIT 的时候，偏移量 M 在分页越靠后的时候，值就越大，数据库检索的数据也就越多。例如 LIMIT 10000,10 这样的查询，数据库需要查询 10010 条记录

    - ```sql
      select * from `demo`.`order` order by order_no limit 10000, 20;
      
      select * from `demo`.`order` where id> (select id from `demo`.`order` order by order_no limit 10000, 1)  limit 20;
      ```

      

  - **优化 SELECT COUNT(*)**

    - COUNT() 是一个聚合函数，主要用来统计行数，有时候也用来统计某一列的行数量（不统计 NULL 值的行）。我们平时最常用的就是 COUNT(*) 和 COUNT(1) 这两种方式了，其实两者没有明显的区别，在拥有主键的情况下，它们都是利用主键列实现了行数的统计。

      但 COUNT() 函数在 MyISAM 和 InnoDB 存储引擎所执行的原理是不一样的，通常在没有任何查询条件下的 COUNT(*)，MyISAM 的查询速度要明显快于 InnoDB。

      这是因为 MyISAM 存储引擎记录的是整个表的行数，在 COUNT(*) 查询操作时无需遍历表计算，直接获取该值即可。而在 InnoDB 存储引擎中就需要扫描表来统计具体的行数。而当带上 where 条件语句之后，MyISAM 跟 InnoDB 就没有区别了，它们都需要扫描表来进行行数的统计。

      如果对一张大表经常做 SELECT COUNT(*) 操作，这肯定是不明智的。那么我们该如何对大表的 COUNT() 进行优化呢？

      - 使用近似值

      有时候某些业务场景并不需要返回一个精确的 COUNT 值，此时我们可以使用近似值来代替。我们可以使用 EXPLAIN 对表进行估算，要知道，执行 EXPLAIN 并不会真正去执行查询，而是返回一个估算的近似值。

      - 增加汇总统计

      如果需要一个精确的 COUNT 值，我们可以额外新增一个汇总统计表或者缓存字段来统计需要的 COUNT 值，这种方式在新增和删除时有一定的成本，但却可以大大提升 COUNT() 的性能。

  - **优化 SELECT ***

    - 假设我们的订单表是基于 InnoDB 存储引擎创建的，且存在 order_no、status 两列组成的组合索引。此时，我们需要根据订单号查询一张订单表的 status，如果我们使用 select * from order where order_no='xxx’来查询，则先会查询组合索引，通过组合索引获取到主键 ID，再通过主键 ID 去主键索引中获取对应行所有列的值。
    - 如果我们使用 select order_no, status from order where order_no='xxx’来查询，则只会查询组合索引，通过组合索引获取到对应的 order_no 和 status 的值

- 我们可以通过以下命令行查询是否开启了记录慢 SQL 的功能，以及最大的执行时间是多少：

  - Show variables like 'slow_query%';

- Show variables like 'long_query_time';

  - 如果没有开启，我们可以通过以下设置来开启：

    ```sql
    set global slow_query_log='ON'; // 开启慢 SQL 日志
    set global slow_query_log_file='/var/lib/mysql/test-slow.log';// 记录日志地址
    set global long_query_time=1;// 最大执行时间
    ```

  

### **MySQL SQL优化**

　　1、建表

　　　　1）、主键使用合适的无符号unsigned整型自增（避免数据增加时索引页分裂）

　　　　2）、长度固定的字符串字段使用CHAR类型，Varchar会额外使用1或者2个字节（长度小于255使用1字节）存储字符串长度

　　　　3）、选择合适的字符串数据类型和日期类型（优先TIMESTAMP，占用4字节）

　　2、索引

　　　　1）、使用单独的列（不对索引进行表达式或者函数运算）

　　　　2）、对于BLOB、TEXT或者很长的VARCHAR类型的列，使用前缀索引

　　　　3）、多列索引选择合适的索引列顺序（索引列的选择度和最左原则）

　　　　4）、能使用覆盖索引的使用覆盖索引、使用索引排序

　　　　5）、索引列字段应不为NULL（会导致索引失效，若是唯一索引，多列可以有NULL值，使唯一索引失效）

　　　　6）、索引保持在5条左右，多的话影响插入和更新性能

​			  7）、更新十分频繁、数据区分度不高的字段上不宜建立索引

​					更新会变更B+树，更新频繁的字段建立索引会大大降低数据库性能。“性别”这种区分度不太大的属性，建立索引是没有什么意义的，不能有效过滤数据，性能与全表扫描类似。**一般区分度在80%以上就可以建立索引。区分度可以使用count（distinct（列名））/count（*）来计算。**

　　3、查询（避免索引失效）

　　　　1）、使用EXPLAIN查询SQL优化器执行计划，调整SQL查询

　　　　2）、使用not exists代替not in，not in不会使用索引，Union、in、or可以命中索引，建议使用in，！=、<>、not in、not exists、not like查询不能使用索引，可以优化为in查询

　　　　3）、查询条件避免使用前导模糊查询，如'%xxx'，因为无法使用索引

　　　　4）、查询条件使用or的话，要保证or两边的列都要有索引，否则索引失效

　　　　5）、字符串型字段为数字时，在where条件中要加单引号，否则索引失效（因为这样MySQL会讲表中字符串类型转换为数字之后再比较，导致索引失效）

　　　　6）、多列索引要满足最左前缀要求

　　　　7）、ISNULL判断不走索引，要慎用

　　　　8）、LIMIT分页的页码不能太大，会查询出所有的结果然后丢弃掉不需要的。使用主键做连接查询
			  9）、把计算放到业务层而不是数据库层。在字段上计算不能命中索引

​	 		10）、强制类型转换会全表扫描，如果phone字段是varcher类型，则下面的SQL不能命中索引。Select * fromuser where phone=13800001234

​			11）、**利用延迟关联或者子查询优化超多分页场景，**

　　MySQL并不是跳过offset行，而是取offset+N行，然后放弃前offset行，返回N行，那当offset特别大的时候，效率非常低下，要么控制返回的总数，要么对超过特定阈值的页进行SQL改写。

​			12）、**业务上唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。**

​			13）、**超过三个表最好不要用join，**

　　需要join的字段，数据类型必须一致，多表关联查询时，保证被关联的字段需要有索引。





MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过 `long_query_time` 值的SQL，则会被记录到慢查询日志中。

**当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。**

 

### 慢查询日志开启

- **查看是否开启：** show variables like '%slow_query_log%';
- **开启慢查询日志：**set global slow_query_log=1; (重启会失效)

 

开启了慢查询日志后，什么样的SQL才会记录到查询日志里面？

**这个是由参数 long_query_time 控制，默认情况下 long_query_time 的值为10秒**

**查看命令：** show variables like 'long_query_time%';

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221813588-807032440.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221813588-807032440.png)

**注： 永久设置慢查询日志开启，以及设置慢查询日志时间临界点（不建议）**

linux中，mysql配置文件一般默认在 /etc/my.cnf 更改对应参数即可

 

**1**|**2****2. 慢查询日志设置与查看**

**设置阀值命令：** set global long_query_time=3 （修改为阀值到3秒钟的就是慢sql）

 

**为什么设置后看不出变化：**

- 需要重新连接或新开一个会话才能看到修改值。 show variables like 'long_query_time%';
- 直接 show global variables like 'long_query_time';

 

**查看慢查询日志：**

**cat -n /data/mysql/mysql-slow.log**

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221835933-1197897695.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221835933-1197897695.png)

从慢查询日志中，我们可以看到每一条查询时间高于3s 的sql语句，并可以看到执行的时间是多少。

比如上面，就表示 sql语句  select * from comic where comic_id < 1952000;  执行时间为3.902864秒，超出了我们设置的慢查询时间临界点3s，所以被记录下来了

 

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221848830-1857798168.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190620221848830-1857798168.png)

**查看有多少条慢查询记录：** show global status like '%Slow_queries%';

 

### 日志分析工具mysqldumpslow

在生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活，MySQL提供了日志分析工具 mysqldumpslow

- s: 是表示按照何种方式排序
- c: 访问次数
- l: 锁定时间
- r: 返回记录
- t: 查询时间
- al:平均锁定时间
- ar:平均返回记录数
- at:平均查询时间
- t:即为返回前面多少条的数据
- g:后边搭配一个正则匹配模式，大小写不敏感的



| ere a = 3 and b like 'k%kk%' and c = 4 | Y,使用到a,b,c |
| -------------------------------------- | ------------- |
|                                        |               |

 

### **剖析报告:Show Profile**

**是什么：**是mysql提供可以用来分析当前会话中语句执行的资源消耗情况。可以用于SQL的调优的测量
**官网介绍：**[show profile](https://dev.mysql.com/doc/refman/5.5/en/show-profile.html)

默认情况下，参数处于关闭状态，开启后默认保存最近15次的运行结果

 

1.**是否支持，看看当前的mysql版本是否支持**

```
Show  variables like 'profiling';
```

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174115595-1462805032.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174115595-1462805032.png)

 

2.**开启功能，默认是关闭，使用前需要开启**

```
set profiling=on;
```

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174129855-1673018705.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174129855-1673018705.png)

3.**运行SQL**

```
select * from emp group by id%10 limit 150000;
select * from emp group by id%20  order by 5
```

 

4.**查看结果 show profile;**

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174146644-429834591.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174146644-429834591.png)

 

5.**诊断SQL，show profile cpu,block io for query 上一步前面的问题SQL数字号码;**

[![img](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174202931-952515884.png)](https://img2018.cnblogs.com/blog/1348730/201906/1348730-20190630174202931-952515884.png)

| **Status**                                                   | **建议**                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| System lock                                                  | 确认是由于哪个锁引起的，通常是因为MySQL或InnoDB内核级的锁引起的建议：如果耗时较大再关注即可，一般情况下都还好 |
| Sending data                                                 | 从server端发送数据到客户端，也有可能是接收存储引擎层返回的数据，再发送给客户端，数据量很大时尤其经常能看见备注：Sending Data不是网络发送，是从硬盘读取，发送到网络是Writing to net建议：通过索引或加上LIMIT，减少需要扫描并且发送给客户端的数据量 |
| Sorting result                                               | 正在对结果进行排序，类似Creating sort index，不过是正常表，而不是在内存表中进行排序建议：创建适当的索引 |
| Table lock                                                   | 表级锁，没什么好说的，要么是因为MyISAM引擎表级锁，要么是其他情况显式锁表 |
| create sort index                                            | 当前的SELECT中需要用到临时表在进行ORDER BY排序建议：创建适当的索引 |
| checking query cache for querychecking privileges on cachedsending cached result to clienstoring result in query cache | 和query cache相关的状态，已经多次强烈建议关闭                |

 











































