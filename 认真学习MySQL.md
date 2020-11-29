# 基础篇

## 基础架构

MySQL架构图

<img src="image/1.png" style="zoom: 33%;" />

连接数据库

```shell
mysql -h$ip -P$port -u$user -p
```

链接完成后，输入

```mysql
show processlist;
```

可以查看到已有的连接，第二行是当前的连接，第三行是另一个shell的链接

![](image/2.png)

客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时。

如果在连接被断开之后，客户端再次发送请求的话，就会收到一个错误提醒： Lost connection to MySQL server during query。这时候如果你要继续，就需要重连，然后再执行请求了。

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。建立连接的过程通常是比较复杂的，所以我建议你在使用中要尽量减少建立连接的动作，也就是尽量使用长连接。

# 日志篇

## 物理日志 redo log 和 WAL(Write-Ahead Logging)

WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。

InnoDB 引擎就会先把记录写到 redo log（粉板）里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做，这就像打烊以后掌柜做的事。

InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写。

<img src="image/3.png" style="zoom: 50%;" />

write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。(**绿色部分为空着的部分**)

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

## 逻辑日志 binlog

最开始 MySQL 里并没有 InnoDB 引擎。MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然**只依靠 binlog 是没有 crash-safe 能力的**，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。

## redo log 和 binlog 的 区别

- redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

例如：

```mysql
create table T(ID int primary key, c int);
update T set c=c+1 where ID=2;;
```

update语句的执行流程

<img src="image/4.png" style="zoom:50%;" />

## redo log 的两段提交

上面的图中，redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。

**如何让恢复到半个月内任意一秒的状态：**

- 首先，找到最近的一次全量备份；
- 然后，从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。

**所以为什么要先两段提交？**

比如某列 id 1 改为 2，假设

1. 先写 redo log 后写 binlog，在写完redo log后崩溃，数据库应该中id应该为2，而binlog中没有，恢复时丢了这个更新
2. 先写 binlog 后写 redo log，binlog写完后崩溃，而redo log没写，这个事务无效，而binlog中没有，恢复时多了这个更新

**事务 commit 和 提交事务处于commit状态 的联系？**

- 他说的“commit 语句”，是指 MySQL 语法中，用于提交一个事务的命令。一般跟 begin/start transaction 配对使用。
- 而我们图中用到的这个“commit 步骤”，指的是事务提交过程中的一个小步骤，也是最后一步。
- 当这个步骤执行完成后，这个事务就提交完成了。“commit 语句”执行的时候，会包含“commit 步骤”。

**不同阶段发生崩溃后**

- 也就是写入 redo log 处于 prepare 阶段之后、写 binlog 之前，发生了崩溃（crash），由于此时 binlog 还没写，redo log 也还没提交，所以崩溃恢复的时候，这个事务会回滚。这时候，binlog 还没写，所以也不会传到备库。
- 也就是 binlog 写完，redo log 还没 commit 前发生 crash
  - 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
  - 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
    - a. 如果是，则提交事务；
    - b. 否则，回滚事务。

**MySQL 怎么知道 binlog 是完整的?**

- statement 格式的 binlog，最后会有 COMMIT；
- row 格式的 binlog，最后会有一个 XID event。

**redo log 和 binlog 是怎么关联起来的?**

- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

**处于 prepare 阶段的 redo log 加上完整 binlog，重启就能恢复?**

binlog完整，备库就会执行，为了主备一致，所以这样设计

**为什么binlog不支持崩溃恢复？**

<img src="image/31.jpg" style="zoom:50%;" />

重启后，引擎内部事务 2 会回滚，然后应用 binlog2 可以补回来；但是对于事务 1 来说，系统已经认为提交完成了，不会再应用一次 binlog1。但是，**InnoDB 引擎使用的是 WAL 技术**，执行事务的时候，**写完内存和日志，事务就算完成了**。

如果之后崩溃，要依赖于日志来恢复数据页。也就是说在图中这个位置发生崩溃的话，事务 1 也是可能丢失了的，而且是数据页级的丢失。此时，binlog 里面并**没有记录数据页的更新细节**，是补不回来的。你如果要说，那我优化一下 binlog 的内容，让它来记录数据页的更改可以吗？但，这其实就是又做了一个 redo log 出来。

**那能不能反过来，只用 redo log，不要 binlog？**

一个是归档。redo log 是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log 也就起不到归档的作用。

下游系统就靠消费 MySQL 的 binlog 来更新自己的数据。关掉 binlog 的话，这些系统就没法输入了。

**redo log buffer 和 redo log 的关系？**

```mysql
begin;
insert into t1 ...
insert into t2 ...
commit;
```

这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。所以，**redo log buffer 就是一块内存，用来先存 redo 日志的**。

也就是说，在执行第一个 insert 的时候，数据的内存被修改了，**redo log buffer 也写入了日志**。但是，真正把日志**写到 redo log 文件**（文件名是 ib_logfile+ 数字），是**在执行 commit 语句的时候做的**。

## change buffer

当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页**不在内存中**的话，在不影响数据一致性的前提下，InnoDB 会将这些**更新操作缓存在 change buffer 中**，这样就**不需要从磁盘中读入这个数据页**了。

在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

change buffer，实际上它是可以持久化的数据。也就是说，change buffer 在内存中有拷贝，也会被写入到磁盘上。

将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge。除了**访问这个数据页会触发 merge** 外，系统有**后台线程会定期 merge**。在**数据库正常关闭（shutdown）的过程中，也会执行 merge 操作**。

**什么条件下可以使用 change buffer 呢？**

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入 (4,400) 这个记录，就要先判断现在表中是否已经存在 k=4 的记录，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。

因此，唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。

将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。change buffer 因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。

## 为什么MySQL会抖一下

一条 SQL 语句，正常执行的时候特别快，但是有时也不知道怎么回事，它就会变得特别慢，并且这样的场景很难复现，它不只随机，而且持续时间还很短。

InnoDB 在处理更新语句的时候，只做了写日志这一个磁盘操作。这个日志叫作 redo log（重做日志）

当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。

什么情况下会刷脏页？

1. InnoDB 的 redo log 写满了。这时候系统会停止所有更新操作，把 checkpoint 往前推进
2. 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
3. MySQL 认为系统“空闲”的时候
4. MySQL 正常关闭的情况

第二种情况，内存不够用是常态，**InnoDB 用缓冲池（buffer pool）管理内存**，缓冲池中的内存页有三种状态：

- 还没有使用的
- 使用了并且是干净页
- 使用了并且是脏页

InnoDB 的策略是尽量使用内存，因此对于一个长时间运行的库来说，**未被使用的页面很少**。而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把**最久不使用**的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就**直接释放**出来复用；但如果是脏页呢，**就必须将脏页先刷到磁盘，变成干净页后才能复用**。

刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

- 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
- 日志写满，更新全部堵住，要先向前推 checkpoint，写性能跌为 0，这种情况对敏感业务来说，是不能接受的。

所以，InnoDB 需要有**控制脏页比例**的机制，来尽量避免上面的这两种情况。

- 这就要用到 innodb_io_capacity 这个参数了，它会告诉 InnoDB 你的磁盘能力。这个值我建议你设置成磁盘的 IOPS

  ```
  fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
  ```

  正确地告诉 InnoDB 所在主机的 IO 能力，这样 InnoDB 才能知道需要全力刷脏页的时候，可以刷多快。

- 参数 innodb_max_dirty_pages_pct 是脏页比例上限，默认值是 75%

- MySQL 中的一个机制，可能让你的查询会更慢：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延，也就是对于每个邻居数据页，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷。
  InnoDB 中，innodb_flush_neighbors 参数就是用来控制这个行为的，值为 1 的时候会有上述的“连坐”机制，值为 0 时表示不找邻居，自己刷自己的。
  在 MySQL 8.0 中，innodb_flush_neighbors 参数的默认值已经是 0 了。





# 事务篇

## 事务隔离级别

- 读未提交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交是指，一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

查询隔离级别:

```mysql
show variables like 'transaction_isolation'
```

![](image/5.png)

在 information_schema 库的 innodb_trx 这个表中查询长事务，比如下面这个语句，用于查找持续时间超过 60s 的事务

```mysql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

## MVCC

举例：

建表：

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `k` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

插入数据：

```mysql
insert into t(id, k) values(1,1),(2,2);
```

执行：

![](image/14.png)

三个事务的执行流程：

<img src="image/15.png" style="zoom: 67%;" />

begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用 start transaction with consistent snapshot 这个命令。

- 第一种启动方式，一致性视图是在执行第一个快照读语句时创建的；
- 第二种启动方式，一致性视图是在执行 start transaction with consistent snapshot 时创建的。

数据表中的一行记录，其实可能有多个版本 (row)，每个版本有自己的 row trx_id。

<img src="image/17.png" style="zoom:50%;" />

图中的三个虚线箭头，就是 **undo log**；而 V1、V2、V3 **并不是物理上真实存在的**，而是每次需要的时候根据当前版本和 undo log 计算出来的。比如，需要 V2 的时候，就是通过 V4 依次执行 U3、U2 算出来。

InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务 ID。“活跃”指的就是，启动了但还没提交。

对于当前事务的启动瞬间来说，一个数据版本的 row trx_id，有以下几种可能：

- 如果落在绿色部分，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的；
- 如果落在红色部分，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
- 如果落在黄色部分，那就包括两种情况
  - 若 row trx_id 在数组中(这个事务比当前事务开始的早)，表示这个版本是由还没提交的事务生成的，不可见；
  - 若 row trx_id 不在数组中(这个事务比当前事务开始的晚)，表示这个版本是已经提交了的事务生成的，可见。

## 当前读

**实测在上面实例中语句7之前查询`select * from test1 where id = 1`会得到 1，而执行了语句7之后的查询得到 3，为什么？**

更新数据的时候，就不能再在历史版本上更新了，否则事务 C 的更新就丢失了。因此，事务 B 此时的 set k=k+1 是在（1,2）的基础上进行的操作。

所以，这里就用到了这样一条规则：**更新数据都是先读后写的，而这个读，==只能读当前的值==，称为“当前读”（current read）**。自己的版本号和最新的版本号一致，这个更新才是"自己的"，才能更新。

update触发了当前读，获取到最新版本的(1, 2)，再update得到(1, 3)，而事务A没有触发当前读，所以还是(1, 1)。

**select语句触发当前读**

如果把事务 A 的查询语句 select * from t where id=1 修改一下，加上 lock in share mode 或 for update，也都可以读到最新版本的数据，返回的 k 的值是 3。下面这两个 select 语句，就是分别加了读锁（S 锁，共享锁）和写锁（X 锁，排他锁）。

```mysql
select k from t where id=1 lock in share mode;
select k from t where id=1 for update;
```

实测结果：

![](image/18.png)

执行第9句时卡住了，因为事务B没有提交，事务B提交（第10句）后，事务A的第9句出现了结果。因为事务A的当前读需要id = 1的行锁，而事务B没提交，没有释放这个行锁，事务A就需要等待。

**可重复读的核心就是一致性读（consistent read）**；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

**可重复读和读提交最主要的区别是：**

- 在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；
  - 个人理解：事务B证明了当前读会更新这个一致性视图
- 在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图。



# 索引篇

回表和最左前缀不表

## 索引下推

假设有表

```mysql
CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card` (`id_card`),
  KEY `name_age` (`name`,`age`)
) ENGINE=InnoDB
```

查询

```mysql
mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
```

这个查询可以用到索引 `name_age`，在MySQL5.6之前，只会从第一个命中索引的开始一个个回表，一个个比对 age 和 ismale 字段

<img src="image/6.jpg" style="zoom:50%;" />

而在MySQL5.6之后，会对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

<img src="image/7.jpg" style="zoom:50%;" />

## 索引的查询过程

先是通过 B+ 树**从树根开始，按层搜索到叶子节点**，然后可以认为**数据页内部通过二分法来定位记录**。

- 对于普通索引来说，查找到满足条件的第一个记录 、后，需要查找下一个记录，直到碰到第一个不满足条件的记录。
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。

因为引擎是按页读写的，所以说，当找到 k=5 的记录的时候，它所在的数据页就都在内存里了。那么，对于普通索引来说，要多做的那一次“查找和判断下一条记录”的操作，就只需要一次指针寻找和一次计算。

如果这个记录刚好是这个数据页的最后一个记录，那么要取下一个记录，必须读取下一个数据页，这个操作会稍微复杂一些。

但是，对于整型字段，一个数据页可以放近千个 key，因此出现这种情况的概率会很低。

所以普通索引和唯一索引**在查询性能上的差距不大**。

更新时，唯一索引因为要判断是否违反唯一性约束，索引不能使用change buffer，而普通索引可以。

## 选索引

建表

```mysql
CREATE TABLE `test2` (
  `id` int NOT NULL,
  `a` int DEFAULT NULL,
  `b` int DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

使用存储过程生成数据：

```mysql
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into test2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

**执行**

```mysql
explain select * from test2 where a between 10000 and 20000;
```

得到

![](image/19.png)

key 字段为 a，表示优化器选择了索引 a。

**再做如下操作**

<img src="image/22.png" style="zoom:67%;" />

课程中的到的结果(5.6)是：

![](image/21.png)

实测结果（8.0）是：

![](image/20.png)

因为教程中的索引统计不准确

![](image/23.png)

而实测的索引统计准确

![](image/24.png)

教程中的索引在重新统计后，计算正确

![](image/25.png)

**另一个选错索引的例子**

![](image/26.png)

不管事务A提交没提交都是如此。提交了事务B再运行也是如此

![](image/27.png)

事实的运行速度也是使用force index会更快。

第一种方法就是 force index，很多程序员不喜欢使用 force index，一来这么写不优美，二来如果索引改了名字，这个语句也得改，显得很麻烦。而且如果以后迁移到别的数据库的话，这个语法还可能会不兼容。

第二种方法就是，我们可以考虑修改语句，**引导 MySQL 使用我们期望的索引**。比如，在这个例子里，显然把“order by b limit 1” 改成 “order by b,a limit 1”

![](image/28.png)

第三种方法是，在有些场景下，我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。比如**直接把索引b删掉**。

## 如何对字符串建索引

也就是说使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

首先，你可以使用下面这个语句，算出这个列上有多少个不同的值：

```mysql
select count(distinct email) as L from SUser;
```

然后，依次选取不同长度的前缀来看这个值，比如我们要看一下 4~7 个字节的前缀索引，可以用这个语句：

```mysql
select 
	count(distinct left(email,4)）as L4, 
    count(distinct left(email,5)）as L5, 
    count(distinct left(email,6)）as L6, 
    count(distinct left(email,7)）as L7,
from SUser;
```

建立前缀索引

```mysql
alter table SUser add index index2(email(6));
```

遇到前缀相同的字符串时，前缀索引可能不适用，此时的解决方法：

- 倒着存字符串，再使用前缀索引

- 再加一个Hash字段，再对这个hash字段建索引

  ```mysql
  alter table t add id_card_crc int unsigned, add index(id_card_crc);
  select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string‘
  ```

## 什么情况下会用不到索引

1. 对字段做了函数计算，就用不上索引了
2. 隐式类型转换
   where varchar = int，到底是左值转换成int，还是右值转换成varchar？
   ![](image/34.png)
   答案是**会将左值转换成int**。
   ![](image/35.png)
   看起来是**把varchar转成int,无论左右值**
   ![](image/36.png)
3. 隐式字符集编码转换，比如表A是utf8，表B是utf8mb4，这样在where A.name=B.userName时会对A.Name作CONVERT(A.Name USING utf8mb4)，这是一个函数操作，按照函数操作不走索引的规则，这个也用户到索引。

# 锁

## 全局锁

全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。

当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

**为什么不用 mysqldump？**

当 mysqldump 使用参数**–single-transaction** 的时候，导数据之前就会启动一个**事务**，来确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

有了这个功能，为什么还需要 FTWRL 呢？**一致性读是好，但前提是引擎要支持这个隔离级别**。比如，对于 MyISAM 这种不支持事务的引擎，如果备份过程中有更新，总是只能取到最新的数据，那么就破坏了备份的一致性。这时，我们就需要使用 FTWRL 命令了。

**为什么不使用set global readonly=true 的方式？**

确实 readonly 方式也可以让全库进入只读状态，但我还是会建议你用 FTWRL 方式，主要有两个原因：

- 在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改 global 变量的方式影响面更大。
- 在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高。

## 表级锁

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)

**表锁的语法是 lock tables … read/write**。与 FTWRL 类似，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。

需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。举个例子, 如果在某个线程 A 中执行 lock tables t1 read, t2 write; 这个语句，则其他线程写 t1、读写 t2 的语句都会被阻塞。同时，线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。连写 t1 都不允许，自然也不能访问其他表。

**另一类表级的锁是 MDL（metadata lock)**。MDL 不需要显式使用，在访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

**MDL写锁**：更改表结构时，加MDL写锁，例如：

![](image/37.png)
可以查到阻塞select语句的pid
![](image/38.png)
kill 掉连接![](image/39.png)
发现查询返回了结果，原来加MDL写锁的connection需要重连
![](image/40.png)

**MDL读锁**：增删改查都加MDL读锁

![](image/8.png)

执行顺序是左上，右上，↙，↘。即：

<img src="image/9.jpg" style="zoom:50%;" />

**session C 被 blocked 的原因：**session A 的 MDL 读锁还没有释放，而 session C 需要 MDL 写锁，因此只能被阻塞

**session D 被 blocked 的原因：**新申请 MDL 读锁的请求也会被 session C 阻塞

此时再在session A 和 session B 中输入

```mysql
select * from t limit 1;
```

![image-20201107221358171](C:\Users\ADMIN\AppData\Roaming\Typora\typora-user-images\image-20201107221358171.png)

session B 也会被阻塞，而 session A 不会，也就是除了没有结束事务的 session A，其他 session 全都被阻塞！

事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。

如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新 session 再请求的话，这个库的线程很快就会爆满。

session A 提交后，其他 session 取消阻塞。

![](image/11.png)

**如何安全地给小表加字段？**

首先我们要解决长事务，事务不提交，就会一直占着 MDL 锁。在 MySQL 的 information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。如果你要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务。

但考虑一下这个场景。如果你要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁，而你不得不加个字段，你该怎么做呢？

这时候 kill 可能未必管用，因为新的请求马上就来了。比较理想的机制是，在 alter table 语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者 DBA 再通过重试命令重复这个过程。

MariaDB 已经合并了 AliSQL 的这个功能，所以这两个开源分支目前都支持 DDL NOWAIT/WAIT n 这个语法。

```mysql
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 
```

MySQL8.0.21亲测无效

**为什么只查一行的语句执行的很慢？**

1. 等MDL锁

2. 等flush，flush在正常情况下执行的很快，但可能出现其他语句堵住flush的情况

3. 查询慢
   ![](image/41.png)

   `select * from t where id = 1`要比`select * from t where id = 1 lock in share mode`慢很多，因为带 lock in share mode 的 SQL 语句，是当前读，因此会直接读到 1000001 这个结果，所以速度很快；而 select * from t where id=1 这个语句，是一致性读，因此需要从 1000001 开始，依次执行 undo log，执行了 100 万次以后，才将 1 这个结果返回。
   <img src="image/42.png" style="zoom: 50%;" />

## 行锁

在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是**两阶段锁协议**。

![](image/12.png)

行锁死锁

![](image/13.png)

事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。

当出现死锁以后，有两种策略：

- 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。
- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

上图中实测采用了第二种策略，而且 innodb_deadlock_detect 的默认值本身就是 on。

你可以考虑通过将一行改成逻辑上的多行来减少锁冲突。还是以影院账户为例，可以考虑放在多条记录上，比如 10 个记录，影院的账户总额等于这 10 个记录的值的总和。这样每次要给影院账户加金额的时候，随机选其中一条记录来加。这样每次冲突概率变成原来的 1/10，可以减少锁等待个数，也就减少了死锁检测的 CPU 消耗。

## select lock in share mode 和 select for update

### lock in share mode加读锁

在能用上索引的情况下加的是行锁：

![](image/45.png)

用不到索引的情况下加的是表锁：

![](image/46.png)

### for update加写锁

能用上索引就是行锁

![](image/47.png)

用不上索引就是表锁

![](image/48.png)

结论：**有索引加行锁，没索引加表锁**。推论：**锁是加在索引上的**。

### LOCK IN SHARE MODE 有什么用？

**两张表的数据一致**

一个表是child表，一个是parent表，假设child表的某一列child_id映射到parent表的c_child_id列，那么从业务角度讲，此时我直接insert一条child_id=100记录到child表是存在风险的，因为刚insert的时候可能在parent表里删除了这条c_child_id=100的记录，那么业务数据就存在不一致的风险。正确的方法是再插入时执行select * from parent where c_child_id=100 lock in share mode,锁定了parent表的这条记录，然后执行insert into child(child_id) values (100)就ok了。

### FOR UPDATE 有什么用？

**一张表的数据一致**

电商系统中计算一种商品的剩余数量，在产生订单之前需要确认商品数量>=1,产生订单之后应该将商品数量减1。

如果：

```mysql
select amount from product where product_name='XX';
update product set amount=amount-1 where product_name='XX';
```

可能在事务A在第一条语句执行完时，事务B执行了第二条语句，那么事务A再执行第二条语句就会导致数据不一致。

那么，**LOCK IN SHARE MODE 可行吗？**

事务A在第一条语句执行完时，事务B的第二条语句也可以执行下去，那么事务A再执行第二条语句就会阻塞（因为事务B获取到了读锁）

![](image/50.png)

事务B执行第二条语句会死锁。

![](image/52.png)

**使用for update**，事务B在select时就被堵住，事务A提交后，B就可以继续执行

![](image/53.png)

## S锁、X锁、IS锁、IX锁

IS锁和IX锁是意向锁，表达一个事务想要获取什么，在加行锁或表锁之前要都加意向锁，**意向锁是表级锁**。

不会和行级的S锁，X锁冲突，只会和表级的S锁，X锁冲突。

![](image/49.png)

意向锁的作用：协调行锁和表锁的关系，提升加表锁时的性能。

假设有一张1w条记录的表，有一行上了**行写锁**，**在上行X锁之前就上了IX锁**，在请求**表S锁**时，就会直接判断为阻塞。如果没有IX锁，加表S锁时则需要逐行判断S锁。

IS锁同理。

- 意向共享锁和意向排他锁都是系统自动添加和自动释放的，整个过程无需人工干预
- 由于InnoDB存储引擎支持的是行级别的锁，因此意向锁不会阻塞除全表扫描以外的任何请求。

## 幻读和间隙锁

建表

```mysql
 CREATE TABLE `test3` (
  `id` int NOT NULL,
  `c` int NOT NULL,
  `d` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

insert into test3 values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。**间隙锁之间都不存在冲突关系**。

![](image/55.PNG)

两个session都加了(5, 10)的间隙锁，它们有共同的目标，即：保护这个间隙，不允许插入值。但，它们之间是不冲突的。

**间隙锁和行锁合称 next-key lock**，每个 next-key lock 是**前开后闭区间**。也就是说，我们的表 t 初始化以后，如果用 select * from t for update 要把整个表所有记录锁起来，就形成了 7 个 next-key lock，分别是 (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]。(InnoDB 给每个索引加了一个不存在的最大值 supremum)

**间隙锁引发的死锁**

![](image/57.png)

sessionB被阻塞，然后在sessionA中执行`INSERT INTO test3 VALUES(9,9,9)`。

![](image/58.png)

死锁，原因是sessionB在等待sessionA释放间隙锁，sessionA在等待sessionB释放间隙锁。

**间隙锁是在可重复读隔离级别下才会生效的。所以，你如果把隔离级别设置为读提交的话，就没有间隙锁了。但同时，你要<u>解决可能出现的数据和日志不一致问题</u>，需要把 binlog 格式设置为 row。**

## 加锁原则

两个“原则”、两个“优化”和一个“bug”：

- 原则 1：加锁的基本单位是 next-key lock，next-key lock 是前开后闭区间。
- 原则 2：查找过程中访问到的对象才会加锁。
- 优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
- 优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
- 一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。（已修复）

**案例1：**

![](image/59.png)

用到原则1，锁 (5, 10] ,再用优化2，锁(5, 10)，所以9被阻塞10不被阻塞。

**案例2：**

![](image/61.png)

因为原则2，访问到的对象才加锁，所以锁只加在 索引c 上的，没有加在主键索引上，所以sessionB可以通过主键索引访问。

因为原则1 + 优化2，索引c上锁的范围是(5, 10)，所以sessionC的insert语句会被阻塞。

如果把sessionA的select id变成select *：

![](image/60.png)

所以sessionB可以通过主键索引的访问被阻塞，因为这个select *会回表，回表访问了主键索引，所以主键索引上也是有锁的。

**案例3：**

mysql45讲的(5.x 系列 <=5.7.24，8.0 系列 <=8.0.13)：

<img src="image/63.png" style="zoom: 50%;" />

mysql8.0.19实测：

![](image/62.png)

也就是sessionC表明 id = 15 的行没有被锁。

根据原则1应该加next_key_lock(5, 10]，(10, 15]，根据优化1应该将(5, 10]退化为id = 10的行锁。会优化为[10, 15]，可能8.0.19做了其他优化，变成了[10, 15)。**其实根据优化2，15不符合条件，所以退化成间隙锁，所以只是解决了bug而已**

而`select * from test3 where id =  10`只有一个行锁。

**案例4：**

<img src="image/65.png" style="zoom:50%;" />

8.0.19实测：

![](image/73.png)

因为c不是唯一索引，所以不使用优化1，加锁范围(5, 10]，(10, 15)。

**案例5：**

> 接下来再看一个关于加锁规则中 bug 的案例。
>
> <img src="image/67.png" style="zoom:50%;" />
>
> session A 是一个范围查询，按照原则 1 的话，应该是索引 id 上只加 (10,15]这个 next-key lock，并且因为 id 是唯一键，所以循环判断到 id=15 这一行就应该停止了。
>
> 但是实现上，InnoDB 会往前扫描到第一个不满足条件的行为止，也就是 id=20。而且由于这是个范围扫描，因此索引 id 上的 (15,20]这个 next-key lock 也会被锁上。
>
> 所以你看到了，session B 要更新 id=20 这一行，是会被锁住的。同样地，session C 要插入 id=16 的一行，也会被锁住。
>
> 照理说，这里锁住 id=20 这一行的行为，其实是没有必要的。因为扫描到 id=15，就可以确定不用往后再找了。但实现上还是这么做了，因此我认为这是个 bug。

8.0.19实测效果：

![](image/66.png)

**案例6**

<img src="image/69.png" style="zoom:50%;" />

![](image/74.png)

delete 语句加锁的逻辑，其实跟 select ... for update 是类似的，根据原则1和优化2，加(5, 10] 和(10, 15)。

<img src="image/70.png" style="zoom:50%;" />

**案例7**

<img src="image/71.png" style="zoom: 67%;" />

![](image/75.png)

delete 语句明确加了 limit 1 的限制，因此在遍历到 (c=10, id=10) 这一行之后，满足条件的语句已经有两条，循环就结束了。

在删除数据的时候尽量加 limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。

<img src="image/72.png" style="zoom:50%;" />

**案例8**

![](image/77.png)

session B 的“加 next-key lock(5,10] ”操作，实际上分成了两步，先是加 (5,10) 的间隙锁，加锁成功；然后加 c=10 的行锁，需要等待sessionA释放。

而sessionA的insert操作需要sessionB释放（5，10）的间隙锁，形成了死锁。

死锁四要素：

1. 互斥：一次只允许一个进程访问
2. 占有且等待：占有A资源，等待B资源
3. 不可抢占
4. 循环等待

# 实现原理

## count(*)

不同的 MySQL 引擎中，count(*) 有不同的实现方式。

- MsyISAM 引擎把一个表的总行数存在了磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高；
- 而 InnoDB 引擎就麻烦了，它执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。

如果加了 where 条件的话，MyISAM 表也是不能返回得这么快的。

**为什么 InnoDB 不跟 MyISAM 一样，也把数字存起来呢？**

因为MVCC

<img src="image/29.png" style="zoom:67%;" />

show table status 命令的输出结果里面也有一个 TABLE_ROWS 用于显示这个表当前有多少行

TABLE_ROWS 就是从这个采样估算得来的，因此它也很不准。有多不准呢，官方文档说误差可能达到 40% 到 50%。所以，show table status 命令显示的行数也不能直接使用。

**如何快速计算总数？**

1. 使用Reids存一个整数，缺点是会丢失更新，Redis可能会异常重启，即使设置了持久化，也可能丢失重启时的更新
   即使重启时不丢失更新，也可能数据不一致，比如取最近的100条数据。

   <img src="image/30.png" style="zoom:67%;" />

   此时 redis 的值是被会话A更新过的，而 mysql 中读出的数据是会话B的一致性视图，造成了数据不一致。

2. 用数据库保存总数。首先InnoDB支持崩溃恢复，然后事务可以避免数据不一致。

**count是怎么数的？**

- 对于 count(主键 id) 来说，InnoDB 引擎会遍历整张表，把每一行的 id 值都取出来，返回给 server 层。server 层拿到 id 后，判断是不可能为空的，就按行累加。
- 对于 count(1) 来说，InnoDB 引擎遍历整张表，但不取值。server 层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。
- 对于 count(字段) 来说：
  - 如果这个“字段”是定义为 not null 的话，一行行地从记录里面读出这个字段，判断不能为 null，按行累加；
  - 如果这个“字段”定义允许为 null，那么执行的时候，判断到有可能是 null，还要把值取出来再判断一下，不是 null 才累加。
- 但是 count(\*) 是例外，并不会把全部字段取出来，而是专门做了优化，不取值。count(\*) 肯定不是 null，按行累加。

效率：count(字段) > count(主键 id) > count(1) ≈ count(*)

**为什么count(1) 执行得要比 count(主键 id) 快**？因为从引擎返回 id 会涉及到解析数据行，以及拷贝字段值的操作。

## Order By

举例：

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```

explain这个语句:

```mysql
select city,name,age from t where city='杭州' order by name limit 1000;
```

![](image/31.png)

Extra 这个字段中的“Using filesort”表示的就是需要排序，MySQL 会给每个线程分配一块内存用于排序，称为 sort_buffer。

执行流程：

<img src="image/32.png" style="zoom: 50%;" />

1. 初始化 sort_buffer，确定放入 name、city、age 这三个字段；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id，也就是图中的 ID_X；
3. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到 city 的值不满足查询条件为止，对应的主键 id 也就是图中的 ID_Y；
6. 对 sort_buffer 中的数据按照字段 name 做快速排序；
7. 按照排序结果取前 1000 行返回给客户端。

我们暂且把这个排序过程，称为**全字段排序**。

**如果 MySQL 认为排序的单行长度太大会怎么做呢？**

Mysql中有一个专门控制排序的**行数据的长度**的参数，单位是字节

```mysql
SET max_length_for_sort_data = 16;
```

name + city + age = 16 + 16 + 4 = 36 Bytes > 16 Bytes，此时放入 sort_buffer 的字段，只有要排序的列（即 name 字段）和主键 id。

此时的执行流程：

1. 初始化 sort_buffer，确定放入两个字段，即 name 和 id；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id，也就是图中的 ID_X；
3. 到主键 id 索引取出整行，取 name、id 这两个字段，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到不满足 city='杭州’条件为止，也就是图中的 ID_Y；
6. 对 sort_buffer 中的数据按照字段 name 进行排序；
7. 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

内存放不下时，就需要使用外部排序，**外部排序**一般使用**归并排序**算法。可以这么简单理解，MySQL 将需要排序的数据分成 n 份，每一份单独排序后存在这些临时文件中。然后把这 n 个有序文件再合并成一个有序的大文件。

**如果索引是city、name**，此时的执行流程：

1. 从索引 (city,name) 找到第一个满足 city='杭州’条件的主键 id；
2. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，作为结果集的一部分直接返回；
3. 从索引 (city,name) 取下一个记录主键 id；
4. 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

**如果索引是city、name、age**，此时的执行流程：

1. 从索引 (city,name,age) 找到第一个满足 city='杭州’条件的记录，取出其中的 city、name 和 age 这三个字段的值，作为结果集的一部分直接返回；
2. 从索引 (city,name,age) 取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回；
3. 重复执行步骤 2，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

## 如何正确显示随机消息-单词软件随机三个单词

建表 + 存储过程

```mysql

mysql> CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

**方法一：order by rand() 使用内存临时表**

```mysql
select word from words order by rand() limit 3;
```

explain一下

![](/image/33.png)

Extra 字段显示 Using temporary，表示的是需要使用临时表；Using filesort，表示的是需要执行排序操作。

即 **需要在临时表上排序！**

- 对于InnoDB表来说，执行全字段排序会减少回表造成的磁盘访问，因此会被优先选择。
- 而我们使用了内存临时表，回表就是访问内存而非磁盘，所以不会全字段排序，而是排序的行越小越好。

这个语句的执行流程：

1. 创建一个内存临时表，memory引擎，假设是double类型的R + varchar(64)类型的word
2. 从words表中按主键顺序取出所有的word值，rand()得到的值存到R，word存到W
3. 对内存临时表按照R排序
4. 初始化sort_buffer，sort_buffer中有整型的row_id和浮点型的R
5. 从内存临时表中取出row_id和R，存到sort_buffer
6. 在sort_buffer中根据R排序
7. 排序完成后，取出排序后的前三的row_id，然后从内存临时表中根据row_id找到对应的行

总结：**order by rand() 使用了内存临时表，内存临时表排序的时候使用了 rowid 排序方法。**

