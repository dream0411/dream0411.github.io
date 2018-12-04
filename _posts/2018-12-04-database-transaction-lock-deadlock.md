---
layout: default
title: mysql 事务、锁、索引、死锁
---

{{ page.title }}
===

# 一、事务

## (一) 事务四要素：ACID

* 原子性（Atomicity）：要么全部完成，要么全部不完成；
* 一致性（Consistency）：一个事务单元需要提交之后才会被其他事务可见；
* 隔离性（Isolation）：并发事务之间不会互相影响，设立了不同程度的隔离级别，通过适度的破坏一致性，得以提高性能；
* 持久性（Durability）：事务提交后即持久化到磁盘不会丢失。

## (二)事务并发存在的问题

1. 脏读dirty rea）: 没有提交的事务被其他事务读取到了，这叫做脏读

2. 不可重复读unrepeatable read: 不可重复读是读取了另一个事务提交之后的修改

3. 幻读phantom read: 同样的条件，第一次和第二次读出来的记录数不一样

幻读和不可重复读的区别：
* 幻读和不可重复读的区别在于，后者是两次读取同一条记录，得到不一样的结果；
* 而前者是两次读取同一个范围内的记录，得到不一样的记录数（这种说法其实只是便于理解，但并不准确，因为可能存在另一个事务先插入一条记录然后再删除一条记录的情况，这个时候两次查询得到的记录数也是一样的，但这也是幻读，所以严格点的说法应该是：两次读取得到的结果集不一样）。
* 很显然，不可重复读是因为其他事务进行了 UPDATE 操作，幻读是因为其他事务进行了 INSERT 或者 DELETE 操作

4. 丢失更新lost update: 提交覆盖和回滚覆盖；两段事务并发运行，先读后写的过程交叉，一个打断了另一个的读写结果，比如修改库存，各自减一，但存在可能导致一共只减掉1个

## (三)隔离级别

1. 对操作的数据进行加锁
2. 数据库的一致性得以保障，但会降低事务的并发能力
3. 四个隔离级别：
* 读未提交（Read Uncommitted）：可以读取未提交的记录，会出现脏读，幻读，不可重复读，所有并发问题都可能遇到；
* 读已提交（Read Committed）：事务中只能看到已提交的修改，不会出现脏读现象，但是会出现幻读，不可重复读；（大多数数据库的默认隔离级别都是 RC，但是 MySQL InnoDb 默认是 RR）
* 可重复读（Repeatable Read）：MySQL InnoDb 默认的隔离级别，解决了不可重复读问题，但是任然存在幻读问题；（MySQL 的实现有差异，后面介绍）
* 序列化（Serializable）：最高隔离级别，啥并发问题都没有。

4. 针对这四种隔离级别，应该根据具体的业务来取舍，如果某个系统的业务里根本就不会出现重复读的场景，完全可以将数据库的隔离级别设置为 RC，这样可以最大程度的提高数据库的并发性

5. 在标准的传统实现中，RR 隔离级别是使用持续的 X 锁和持续的 S 锁来实现的（参看下面的 “隔离级别的实现” 一节），由于是持续的 S 锁，所以避免了其他事务有写操作，也就不存在提交覆盖问题。但是 MySQL 在 RR 隔离级别下，普通的 SELECT 语句只是快照读，没有任何的加锁，和标准的 RR 是不一样的。如果要让 MySQL 在 RR 隔离级别下不发生提交覆盖，可以使用`SELECT ... LOCK IN SHARE MODE`或者`SELECT ... FOR UPDATE`

## (四)隔离级别的实现

### 传统的隔离级别

1. 传统的隔离级别是基于锁实现的，这种方式叫做 基于锁的并发控制（Lock-Based Concurrent Control，简写 LBCC）。通过对读写操作加不同的锁，以及对释放锁的时机进行不同的控制，就可以实现四种隔离级别。传统的锁有两种：读操作通常加共享锁（Share locks，S锁，又叫读锁），写操作加排它锁（Exclusive locks，X锁，又叫写锁）；加了共享锁的记录，其他事务也可以读，但不能写；加了排它锁的记录，其他事务既不能读，也不能写。另外，对于锁的粒度，又分为行锁和表锁，行锁只锁某行记录，对其他行的操作不受影响，表锁会锁住整张表，所有对这个表的操作都受影响。

2. 四种隔离级别的加锁策略如下：

* 读未提交（Read Uncommitted）：事务读不阻塞其他事务读和写，事务写阻塞其他事务写但不阻塞读；通过对写操作加 “持续X锁”，对读操作不加锁 实现；
* 读已提交（Read Committed）：事务读不会阻塞其他事务读和写，事务写会阻塞其他事务读和写；通过对写操作加 “持续X锁”，对读操作加 “临时S锁” 实现；不会出现脏读；
* 可重复读（Repeatable Read）：事务读会阻塞其他事务事务写但不阻塞读，事务写会阻塞其他事务读和写；通过对写操作加 “持续X锁”，对读操作加 “持续S锁” 实现；
* 序列化（Serializable）：为了解决幻读问题，行级锁做不到，需使用表级锁。

3. 加锁策略实际上又称为 封锁协议（Locking Protocol），所谓协议，就是说不论加锁还是释放锁都得按照特定的规则来。读未提交 的加锁策略又称为 一级封锁协议，后面的分别是二级，三级，序列化 的加锁策略又称为 四级封锁协议。

4. 二段锁协议和一次封锁

* 三级封锁协议在事务的过程中为写操作加持续 X 锁，为读操作加持续 S 锁，并且在事务结束时才对锁进行释放，像这种加锁和解锁明确的分成两个阶段我们把它称作 两段锁协议（2-phase locking，简称 2PL）。在两段锁协议中规定，加锁阶段只允许加锁，不允许解锁；而解锁阶段只允许解锁，不允许加锁。这种方式虽然无法避免死锁，但是两段锁协议可以保证事务的并发调度是串行化的（关于串行化是一个非常重要的概念，尤其是在数据恢复和备份的时候）。在两段锁协议中，还有一种特殊的形式，叫 一次封锁，意思是指在事务开始的时候，将事务可能遇到的数据全部一次锁住，再在事务结束时全部一次释放，这种方式可以有效的避免死锁发生。但是这在数据库系统中并不适用，因为事务开始时并不知道这个事务要用到哪些数据，一般在应用程序中使用的比较多
* 简单说，两段锁，是事务过程中，遇到写则加写锁，遇到读则加读锁，但只能事务结束时解锁；一次封锁，则是事务开始的时候，一次性加所有的锁，结束时解锁；后者问题在于开始时数据库系统并不知道需要哪些数据。

### MySQL的隔离级别

1. 虽然数据库的四种隔离级别通过 LBCC 技术都可以实现，但是它最大的问题是它只实现了并发的读读，对于并发的读写还是冲突的，写时不能读，读时不能写，当读写操作都很频繁时，数据库的并发性将大大降低，针对这种场景，MVCC 技术应运而生。

2. MVCC 的全称叫做 Multi-Version Concurrent Control（多版本并发控制）

* InnoDb 会为每一行记录增加几个隐含的“辅助字段”，（实际上是 3 个字段：一个隐式的 ID 字段，一个事务 ID，还有一个回滚指针）

* 事务在写一条记录时会将其拷贝一份生成这条记录的一个原始拷贝，写操作同样还是会对原记录加锁，但是读操作会读取未加锁的新记录，这就保证了读写并行。

* 要注意的是，生成的新版本其实就是 undo log，它也是实现事务回滚的关键技术。

* InnoDb 通过 MVCC 实现了读写并行，但是在不同的隔离级别下，读的方式也是有所区别的。首先要特别指出的是，在 read uncommit 隔离级别下，每次都是读取最新版本的数据行，所以不能用 MVCC 的多版本，而 serializable 隔离级别每次读取操作都会为记录加上读锁，也和 MVCC 不兼容，所以只有 RC 和 RR 这两个隔离级别才有 MVCC

* RC总是读取记录的最新版本，如果该记录被锁住，则读取该记录最新的一次快照，而RR是读取该记录事务开始时的那个版本。

* 虽然这两种读取方式不一样，但是它们读取的都是快照数据，并不会被写操作阻塞，所以这种读操作称为 快照读（Snapshot Read），有时候也叫做 非阻塞读（Nonlocking Read），RR 隔离级别下的叫做 一致性非阻塞读Consistent Nonlocking Read: https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html

* 当前读（Current Read），有时候又叫做 加锁读（Locking Read） 或者 阻塞读（Blocking Read），这种读操作读的不再是数据的快照版本，而是数据的最新版本，并会对数据加锁，根据加锁的不同，又分成两类：

```
SELECT ... LOCK IN SHARE MODE：加 S 锁
SELECT ... FOR UPDATE：加 X 锁
INSERT / UPDATE / DELETE：加 X 锁
```

* 当前读在 RR 和 RC 两种隔离级别下的实现也是不一样的：RC 只加记录锁，RR 除了加记录锁，还会加间隙锁，用于解决幻读问题

4. 查看和设置MySQL的隔离级别:

* 读取: `select @@session.tx_isolation, @@global.tx_isolation;`
* 写入，`SET TRANSACTION`命令只对当前会话的下一个事务有效，当下个事务结束之后，下下个事务又会恢复到当前会话的隔离级别:
```
-- 下个事务
set transaction isolation level read uncommitted;
set transaction isolation level read committed;
set transaction isolation level repeatable read;
set transaction isolation level serializable;

-- 当前会话
session transaction isolation level read committed;

-- 全局变量，但当前会话无效
set global transaction isolation level read committed;
```

# 二、锁

* 锁的类型: 表锁和行锁，行锁还可细分

## (一)表锁和行锁

* 锁是加在索引上的

### 表锁

1. 表锁是直接对表加读锁或写锁
2. 表锁使用`一次封锁`技术，即在事务开始时一次性加所有的锁，最后通过unlock tables一次性释放所有的锁
3. MySQL 表锁的加锁规则如下：

* 对于读锁
持有读锁的会话可以读表，但不能写表；
允许多个会话同时持有读锁；
其他会话就算没有给表加读锁，也是可以读表的，但是不能写表；
其他会话申请该表写锁时会阻塞，直到锁释放。

* 对于写锁
持有写锁的会话既可以读表，也可以写表；
只有持有写锁的会话才可以访问该表，其他会话访问该表会被阻塞，直到锁释放；
其他会话无论申请该表的读锁或写锁，都会阻塞，直到锁释放。
锁的释放规则如下：

4. 使用 UNLOCK TABLES 语句可以显示释放表锁；

* 如果会话在持有表锁的情况下执行 LOCK TABLES 语句，将会释放该会话之前持有的锁；
* 如果会话在持有表锁的情况下执行 START TRANSACTION 或 BEGIN 开启一个事务，将会释放该会话之前持有的锁；
* 如果会话连接断开，将会释放该会话所有的锁。

### 行锁的类型

* `LOCK_ORDINARY`: next-key lock，锁一条记录及其之前的间隙，RR隔离级别用的最多
* `LOCK_GAP`: 间隙锁，两头开区间
* `LOCK_REC_NOT_GAP`: 只锁记录
* `LOCK_INSERT_INTENSION`: 插入意向GAP锁，插入记录时使用，是LOCK_GAP的特例

## (二)读锁和写锁

### 锁模式

* `LOCK_IS`：读意向锁；
* `LOCK_IX`：写意向锁；
* `LOCK_S`：读锁；
* `LOCK_X`：写锁；
* `LOCK_AUTO_INC`：自增锁；

### 锁之间的兼容矩阵

![兼容矩阵图](http://www.aneasystone.com/usr/uploads/2017/10/1431433403.png)

* 意向锁之间互不冲突；
* S 锁只和 S/IS 锁兼容，和其他锁都冲突；
* X 锁和其他所有锁都冲突；
* AI 锁只和意向锁兼容；

### AUTO_INC锁

当插入的表中有自增列（AUTO_INCREMENT）的时候可能会遇到

特点：
* AUTO_INC锁互不兼容，也就是说同一张表同时只允许有一个自增锁；
* 自增锁不遵循二段锁协议，它并不是事务结束时释放，而是在 INSERT 语句执行结束时释放，这样可以提高并发插入的性能。
* 自增值一旦分配了就会 +1，如果事务回滚，自增值也不会减回去，所以自增值可能会出现中断的情况。


#### Mysql从5.1.22开始引入了轻量级锁来代替AUTO_INC锁

通过参数`innodb_autoinc_lock_mode` 控制分配自增值时的并发策略。参数`innodb_autoinc_lock_mode`可以取下列值：

1. innodb_autoinc_lock_mode = 0 （traditional lock mode）
使用传统的 AUTO_INC 表锁，并发性比较差。

2. innodb_autoinc_lock_mode = 1 （consecutive lock mode）
* MySQL 默认采用这种方式，是一种比较折中的方法。
* MySQL 将插入语句分成三类：`Simple inserts`、`Bulk inserts`、`Mixed-mode inserts`。通过分析 INSERT 语句可以明确知道插入数量的叫做 Simple inserts，譬如最经常使用的 INSERT INTO table VALUE(1,2) 或 INSERT INTO table VALUES(1,2), (3,4)；通过分析 INSERT 语句无法确定插入数量的叫做 Bulk inserts，譬如 INSERT INTO table SELECT 或 LOAD DATA 等；还有一种是不确定是否需要分配自增值的，譬如 INSERT INTO table VALUES(1,'a'), (NULL,'b'), (5, 'C'), (NULL, 'd') 或 INSERT ... ON DUPLICATE KEY UPDATE，这种叫做 Mixed-mode inserts。
* Bulk inserts 不能确定插入数使用表锁；Simple inserts 和 Mixed-mode inserts 使用轻量级锁 mutex，只锁住预分配自增值的过程，不锁整张表。Mixed-mode inserts 会直接分析语句，获得最坏情况下需要插入的数量，一次性分配足够的自增值，缺点是会分配过多，导致浪费和空洞。
* 这种模式的好处是既平衡了并发性，又能保证同一条 INSERT 语句分配的自增值是连续的。

3. innodb_autoinc_lock_mode = 2 （interleaved lock mode）
* 全部都用轻量级锁 mutex，并发性能最高，按顺序依次分配自增值，不会预分配。
* 缺点是不能保证同一条 INSERT 语句内的自增值是连续的，这样在复制（replication）时，如果 binlog_format 为 statement-based（基于语句的复制）就会存在问题，因为是来一个分配一个，同一条 INSERT 语句内获得的自增值可能不连续，主从数据集会出现数据不一致。所以在做数据库同步时要特别注意这个配置。

4. 参考：[innodb-auto-increment-handling](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)

## (三)细说Mysql锁类型

1. 记录锁 Record Locks

2. 间隙锁 Gap Locks

* 又称为范围锁（Range Locks）
* 间隙锁可以防止其他事务在这个范围内插入或修改记录，保证两次读取这个范围内的记录不会变，从而不会出现幻读现象
* 间隙锁和间隙锁之间是互不冲突的，间隙锁唯一的作用就是为了防止其他事务的插入

3. Next-Key Locks

* 是记录锁和间隙锁的组合，它指的是加在某条记录以及这条记录前面间隙上的锁
* 假设一个索引包含
10、11、13 和 20 这几个值，可能的 Next-key 锁如下：
```
(-∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +∞)
```
* 前面四个都是 Next-key 锁，最后一个为间隙锁。和间隙锁一样，在 RC 隔离级别下没有 Next-key 锁，只有 RR 隔离级别才有

4. 插入意向锁（Insert Intention Locks）

* 插入意向锁，是一种特殊的间隙锁（所以有的地方把它简写成 II GAP），这个锁表示插入的意向，只有在 INSERT 的时候才会有这个锁
* 和上面介绍的表级意向锁是两个完全不同的概念
* 插入意向锁和插入意向锁之间互不冲突，所以可以在同一个间隙中有多个事务同时插入不同索引的记录
* 插入意向锁只会和间隙锁或 Next-key 锁冲突

5. 行锁的兼容矩阵

* 矩阵图，行是要加的锁，列是已有的锁： http://www.aneasystone.com/usr/uploads/2017/11/3404508090.png

* 插入意向锁不影响其他事务加其他任何锁。也就是说，一个事务已经获取了插入意向锁，对其他事务是没有任何影响的

* 插入意向锁与间隙锁和 Next-key 锁冲突。也就是说，一个事务想要获取插入意向锁，如果有其他事务已经加了间隙锁或 Next-key 锁，则会阻塞

* 间隙锁不和其他锁（不包括插入意向锁）冲突

* 记录锁和记录锁冲突，Next-key 锁和 Next-key 锁冲突，记录锁和 Next-key 锁冲突

6. 在MySQL中观察行锁

* 打印出 InnoDb 的所有锁信息，包括锁 ID、事务 ID、以及每个锁的类型和模式等其他信息
`select * from information_schema.INNODB_LOCKS;`

* 输出当前 InnoDb 引擎的状态信息，包括：BACKGROUND THREAD、SEMAPHORES、TRANSACTIONS、FILE I/O、INSERT BUFFER AND ADAPTIVE HASH INDEX、LOG、BUFFER POOL AND MEMORY、ROW OPERATIONS 等
`show engine innodb status\G`

* 哪个事务被阻塞，可以通过`information_schema.innodb_lock_waits`表来查看

* `information_schema.INNODB_LOCKS`: 只有在两个事务出现锁竞争时才能在这个表中看到锁信息，譬如你执行一条 UPDATE 语句，它会对某条记录加 X 锁，这个时候表里是没有任何记录的。

* `information_schema.INNODB_LOCKS`: 只能得到当前持有锁的事

## (四)乐观锁 vs 悲观锁

1. 悲观锁Pessimistic Lock，顾名思义就是很悲观，每次拿数据时都假设有别人会来修改，所以每次在拿数据的时候都会给数据加上锁，用这种方式来避免跟别人冲突，虽然很有效，但是可能会出现大量的锁冲突，导致性能低下。

2. 乐观锁Optimistic Lock，则是完全相反，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有改过这个数据，可以使用版本号等机制来判断。

3. 悲观锁需要使用数据库的锁机制来实现，而乐观锁是通过程序的手段来实现，这两种锁各有优缺点

4. 乐观锁适用于读多写少的情况下，即冲突真的很少发生，这样可以省去锁的开销，加大系统的吞吐量。但如果经常产生冲突，上层应用不断的进行重试，这样反倒是降低了性能，所以这种情况下用悲观锁更合适。虽然使用带版本检查的乐观锁能够同时保持高并发和高可伸缩性，但它也不是万能的，譬如它不能解决脏读问题，所以在实际应用中还是会和数据库的隔离级别一起使用

5. 对读的响应速度和并发性要求比较高的场景适合MVCC；而retry代价越大的场景越适合悲观锁机制。

6. 总结：一种基于多版本并发控制（MVCC）思想的Conditional Update解决分布式系统并发控制问题的方法。和基于悲观锁的方法相比，该方法避免了大粒度和长时间的锁定，能更好地适应对读的响应速度和并发性要求高的场景。

# 三、常见 SQL 语句的加锁分析

# 四、分析和解决死锁

## (一)开启锁监控

1. 通过`show engine innodb status`命令来获取死锁信息，但是它有个限制，只能拿到最近一次的死锁日志，默认情况下监控是关闭的；

2. InnoDb 的监控主要分为四种：标准监控（Standard InnoDB Monitor）、锁监控（InnoDB Lock Monitor）、表空间监控（InnoDB Tablespace Monitor）和表监控（InnoDB Table Monitor）。后两种监控已经基本上废弃；官方监控文档：https://dev.mysql.com/doc/refman/5.6/en/innodb-enabling-monitors.html

3. 通过基于系统表开启监控(表的结构和表里的内容无所谓)

```
-- 开启标准监控
CREATE TABLE innodb_monitor (a INT) ENGINE=INNODB;

-- 关闭标准监控
DROP TABLE innodb_monitor;

-- 开启锁监控
CREATE TABLE innodb_lock_monitor (a INT) ENGINE=INNODB;

-- 关闭锁监控
DROP TABLE innodb_lock_monitor;
```

4. 通过参数开启锁监控（Mysql 5.6.16后）

```
-- 开启标准监控
set GLOBAL innodb_status_output=ON;

-- 关闭标准监控
set GLOBAL innodb_status_output=OFF;

-- 开启锁监控
set GLOBAL innodb_status_output_locks=ON;

-- 关闭锁监控
set GLOBAL innodb_status_output_locks=OFF;
```

5. MySQL提供了一个系统参数 `innodb_print_all_deadlocks` 专门用于记录死锁日志，当发生死锁时，死锁日志会记录到 MySQL 的错误日志文件中
`set GLOBAL innodb_print_all_deadlocks=ON;`

6. 一些有趣的监控工具也很有用，比如 Innotop 和 Percona Toolkit 里的小工具 pt-deadlock-logger

## (二)读懂死锁日志

## (三)常见死锁分析

1. 对索引加锁顺序的不一致很可能会导致死锁，所以如果可以，尽量以相同的顺序来访问索引记录和表。在程序以批量方式处理数据的时候，如果事先对数据排序，保证每个线程按固定的顺序来处理记录，也可以大大降低出现死锁的可能；
2. Gap 锁往往是程序中导致死锁的真凶，由于默认情况下 MySQL 的隔离级别是 RR，所以如果能确定幻读和不可重复读对应用的影响不大，可以考虑将隔离级别改成 RC，可以避免 Gap 锁导致的死锁；
3. 为表添加合理的索引，如果不走索引将会为表的每一行记录加锁，死锁的概率就会大大增大；
4. 我们知道 MyISAM 只支持表锁，它采用一次封锁技术来保证事务之间不会发生死锁，所以，我们也可以使用同样的思想，在事务中一次锁定所需要的所有资源，减少死锁概率；
5. 避免大事务，尽量将大事务拆成多个小事务来处理；因为大事务占用资源多，耗时长，与其他事务冲突的概率也会变高；
6. 避免在同一时间点运行多个对同一表进行读写的脚本，特别注意加锁且操作数据量比较大的语句；我们经常会有一些定时脚本，避免它们在同一时间点运行；
7. 设置锁等待超时参数：`innodb_lock_wait_timeout`，这个参数并不是只用来解决死锁问题，在并发访问比较高的情况下，如果大量事务因无法立即获得所需的锁而挂起，会占用大量计算机资源，造成严重性能问题，甚至拖跨数据库。我们通过设置合适的锁等待超时阈值，可以避免这种情况发生。

# 四、总结

[参考原文](http://www.aneasystone.com/archives/2017/10/solving-dead-locks-one.html)
[作者: aneasystone](http://www.aneasystone.com/)
