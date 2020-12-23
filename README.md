# MySQL
#MySQL的引擎都有哪些？

MySQL内部可以分为服务层和存储引擎层两部分：服务层包括连接器、查询缓存、分析器、优化器、执行器等；存储引擎层负责数据的存储和提取。我就说一下自己了解的InnoDB和MyISAM引擎

InnoDB：

是 MySQL 默认的事务型存储引擎，只有在需要它不支持的特性时，才考虑使用其它存储引擎。
实现了四个标准的隔离级别，默认级别是可重复读(REPEATABLE READ)。在可重复读隔离级别下，通过多版本并发控制(MVCC)+ (Next-Key Locking)防止幻影读。
主索引是聚簇索引，在索引中保存了数据，从而避免直接读取磁盘，因此对查询性能有很大的提升。
内部做了很多优化，包括从磁盘读取数据时采用的可预测性读、能够加快读操作并且自动创建的自适应哈希索引、能够加速插入操作的插入缓冲区等。
支持真正的在线热备份。其它存储引擎不支持在线热备份，要获取一致性视图需要停止对所有表的写入，而在读写混合场景中，停止写入可能也意味着停止读取。
MyISAM：

设计简单，数据以紧密格式存储。对于只读数据，或者表比较小、可以容忍修复操作，则依然可以使用它。
提供了大量的特性，包括压缩表、空间数据索引等。
不支持事务。
不支持行级锁，只能对整张表加锁，读取时会对需要读到的所有表加共享锁，写入时则对表加排它锁。但在表有读取操作的同时，也可以往表中插入新的记录，这被称为并发插入(CONCURRENT INSERT)。
我一般还会回答一个索引文件上的区别

MyISAM：

MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址，同样使用B+Tree作为索引结构，叶节点的data域存放的是数据记录的地址
在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复
MyISAM中索引检索的算法为首先按照B+Tree搜索算法搜索索引，如果指定的Key存在，则取出其data域的值，然后以data域的值为地址，读取相应数据记录
InnoDB：

InnoDB的数据文件本身就是索引文件，这棵树的叶节点data域保存了完整的数据记录（聚集索引）
InnoDB的辅助索引data域存储相应记录主键的值而不是地址
聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。
其实个人还知道一点，分页查询的时候还有一点区别，这点区别也是根据索引文件的区别来的。

咱们知道，使用limit分页查询，offset越大，性能越差，比如：

-- 以真实的生产环境的6万条数据的一张表为例，比较一下优化前后的查询耗时：
-- 传统limit，文件扫描
select * from table order by id limit 50000,2;
受影响的行: 0
时间:  0.171s

-- 子查询方式，索引扫描
select * from table
where id >= (select id from table order by id limit 50000 , 1)
limit 2;
受影响的行: 0
时间: 0.035s

-- JOIN分页方式
select * from table as t1
join (select id from table order by id limit 50000, 1) as t2
where t1.id <= t2.id order by t1.id limit 2;
受影响的行: 0
时间: 0.036s
原因：因为 MySQL 并非是跳过偏移量直接去取后面的数据，而是先把偏移量+要取的条数，然后再把前面偏移量这一段的数据抛弃掉再返回的。比如上面的(50000，2)，每次取2条，还要经过回表，发现不是想要的，舍弃。那肯定非常耗时间，而通过子查询通过id索引，只查询id，使用到了innodb的索引覆盖, 在内存缓冲区中进行检索,没有回表查询. 然后再用id >= 条件,进一步的缩小查询范围.这样就大大提高了效率。

而MyISAM，是直接索引是分离的，通过索引文件查到的数据记录地址，不需要回表，直接对应数据记录，效率也很高。

面试官：分别讲一下MySQL的几大文件，你懂的

我：我不懂，ok，好的。

undoLog 也就是我们常说的回滚日志文件 主要用于事务中执行失败，进行回滚，以及MVCC中对于数据历史版本的查看。由引擎层的InnoDB引擎实现,是逻辑日志,记录数据修改被修改前的值,比如"把id='B' 修改为id = 'B2' ，那么undo日志就会用来存放id ='B'的记录”。当一条数据需要更新前,会先把修改前的记录存储在undolog中,如果这个修改出现异常,则会使用undo日志来实现回滚操作,保证事务的一致性。当事务提交之后，undo log并不能立马被删除,而是会被放到待清理链表中,待判断没有事务用到该版本的信息时才可以清理相应undolog。它保存了事务发生之前的数据的一个版本，用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读。
redoLog 是重做日志文件是记录数据修改之后的值，用于持久化到磁盘中。redo log包括两部分：一是内存中的日志缓冲(redo log buffer)，该部分日志是易失性的；二是磁盘上的重做日志文件(redo log file)，该部分日志是持久的。由引擎层的InnoDB引擎实现,是物理日志,记录的是物理数据页修改的信息,比如“某个数据页上内容发生了哪些改动”。当一条数据需要更新时,InnoDB会先将数据更新，然后记录redoLog 在内存中，然后找个时间将redoLog的操作执行到磁盘上的文件上。不管是否提交成功我都记录，你要是回滚了，那我连回滚的修改也记录。它确保了事务的持久性。
binlog由Mysql的Server层实现,是逻辑日志,记录的是sql语句的原始逻辑，比如"把id='B' 修改为id = ‘B2’。binlog会写入指定大小的物理文件中,是追加写入的,当前文件写满则会创建新的文件写入。 产生:事务提交的时候,一次性将事务中的sql语句,按照一定的格式记录到binlog中。用于复制和恢复在主从复制中，从库利用主库上的binlog进行重播(执行日志中记录的修改逻辑),实现主从同步。业务数据不一致或者错了，用binlog恢复。
MVCC多版本并发控制是MySQL中基于乐观锁理论实现隔离级别的方式，用于读已提交和可重复读取隔离级别的实现。在MySQL中，会在表中每一条数据后面添加两个字段：最近修改该行数据的事务ID，指向该行（undolog表中）回滚段的指针。Read View判断行的可见性，创建一个新事务时，copy一份当前系统中的活跃事务列表。意思是，当前不应该被本事务看到的其他事务id列表。
binlog和redolog的区别：

redolog是在InnoDB存储引擎层产生，而binlog是MySQL数据库的上层服务层产生的。
两种日志记录的内容形式不同。MySQL的binlog是逻辑日志，其记录是对应的SQL语句。而innodb存储引擎层面的重做日志是物理日志。
两种日志与记录写入磁盘的时间点不同，binlog日志只在事务提交完成后进行一次写入。而innodb存储引擎的重做日志在事务进行中不断地被写入，并日志不是随事务提交的顺序进行写入的。
binlog不是循环使用，在写满或者重启之后，会生成新的binlog文件，redolog是循环使用。
binlog可以作为恢复数据使用，主从复制搭建，redolog作为异常宕机或者介质故障后的数据恢复使用。
MVCC的缺点：

MVCC在大多数情况下代替了行锁，实现了对读的非阻塞，读不加锁，读写不冲突。缺点是每行记录都需要额外的存储空间，需要做更多的行维护和检查工作。 要知道的，MVCC机制下，会在更新前建立undo log，根据各种策略读取时非阻塞就是MVCC，undo log中的行就是MVCC中的多版本。 而undo log这个关键的东西，记载的内容是串行化的结果，记录了多个事务的过程，不属于多版本共存。 这么一看，似乎mysql的mvcc也并没有所谓的多版本共存

读写分离原理：

主库（master）将变更写binlog日志，然后从库（slave）连接到主库之后，从库有一个IO线程，将主库的binlog日志拷贝到自己本地，写入一个中继日志中。接着从库中有一个SQL线程会从中继日志读取binlog，然后执行binlog日志中的内容，也就是在自己本地再次执行一遍SQL，这样就可以保证自己跟主库的数据是一样的。

这里有一个非常重要的一点，就是从库同步主库数据的过程是串行化的，也就是说主库上并行的操作，在从库上会串行执行。所以这就是一个非常重要的点了，由于从库从主库拷贝日志以及串行执行SQL的特点，在高并发场景下，从库的数据一定会比主库慢一些，是有延时的。所以经常出现，刚写入主库的数据可能是读不到的，要过几十毫秒，甚至几百毫秒才能读取到。

而且这里还有另外一个问题，就是如果主库突然宕机，然后恰好数据还没同步到从库，那么有些数据可能在从库上是没有的，有些数据可能就丢失了。

所以mysql实际上在这一块有两个机制，一个是半同步复制，用来解决主库数据丢失问题；一个是并行复制，用来解决主从同步延时问题。

所谓并行复制，指的是从库开启多个线程，并行读取relay log中不同库的日志，然后并行重放不同库的日志，这是库级别的并行。
