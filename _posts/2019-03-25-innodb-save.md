---
layout: post
title:  "Innodb存储引擎如何Update和Select"
date:   2019-03-25 22:14:54
categories: jekyll
comments: true
tags:
    - MySQL
    - innodb
---

* content
{:toc}

# 数据记录是如何存储在innodb引擎

- 首先，在innobd当中，数据库的物理存储是会分成多个页，以页为单位，多个记录存储在一个页当中，数据每一次I/O, 读取的单位是一个页。
- 类似于磁盘的预读多读机制，innodb一次读取数据也是对读取周边部分的数据的，局部性原理，起到缓存机制作用
- 数据库运行时，会在内存中为何一公分buffer pool， 缓存部分页。在每次对数据进行select或者update，如果数据在内存当中，比如 select 就可以查询缓存，直接返回， 不用进行磁盘I/O, 否则才进行低效率的I/O； 如果是update操作的话，先将数据页加载内存，然后对内存里面的数据进行修改，当然在对内存进行修改的过程伴随着一系列的redo log和bin log 操作， 这个是我们下面介绍的重点，理解了这个过程，我们也就理解了innodb到底是如何工作的


**修改流程， 先判断数据是否在buffer pool当中，下面的图标没有画出redo log和bin log的修改操作，实质上采用的WAL技术，先走日志文件，再buffer pool 修改数据行记录，最后在必要的时候再讲dirty page flush到磁盘的情况， 完成整个持久化的机制**


[![V1qsdP.md.png](https://s2.ax1x.com/2019/06/01/V1qsdP.md.png)](https://imgchr.com/i/V1qsdP)


先从整体上来讲，innodb的一条事务日志经历四个阶段：
1. 创建阶段：事务创建一条日志
2. 日志落盘：将日志写入磁盘的日志文件，当然在这里面可以根据应用的一致性和磁盘I/0次数效率考虑，要求控制日志落盘的时间和间隙，
3. 数据落潘，事务日志对应修改的dirty page 数据写入磁盘的数据文件，当然同日志文件一样，buffer pool当中的dirty page也不是每执行一个事务就fsync到磁盘，这样磁盘I/0的次数就会非常高，为了效率都是多个事务或者dirty page数量达到一定的程度再几种fsync到磁盘当中去，
4. 写入CKP: 日志被当做checkpoint 写入日志文件

上面四个步骤创建的相关日志文件可以使用下面的图标标记出来

 [![V3AE6I.md.png](https://s2.ax1x.com/2019/06/01/V3AE6I.md.png)](https://imgchr.com/i/V3AE6I)



# 存储引擎进行update

1. 重做日志缓冲区redo log buffer (在内存当中，一个事务当行没执行一个sql语句都写入一条日志) 是一个全局变量
2. 重做日志文件redo log file (一个事务完成后，将重做日志缓冲区落地到磁盘)， 每个事务提交时会将重做日志缓冲刷新到重做日志文件，具体重做日志缓冲区多久一次write 到 redo log file，这个频率是可以设置的，根据实际的生产需求
3. 缓冲池buffer poll，(放着数据页，在内存当中，当对数据库进行修改是，先修改缓冲池当中的，然后再修改)
2. 双缓冲区 Doublewrite buffer 
3. 磁盘当中的数据文件.idb，表空间
6. binlog 二进制文件，主要是存储在服务器层面，binlog 主要是作为redo log 实现数据库事务二阶段提交的协调者角色，处于redo log 第一阶段的prepare之后，就写入binlog,然后就是事务的commit 
7. double write

**思考**
从上面的过程来看，有一个问题就是为何数据的修改现在内存的buffer pool？如果此时宕机了呢，重启，那么内存当中的数据不久不见了吗，事务的修改也就不见了，
>例如如果一个用户完成一个操作（数据库完成了一个事务，page已经在buffer pool中修改，但dirty page尚未flush 落盘持久到磁盘），这时系统断电，buffer pool数据全部消失。那么，这个用户完成的操作（导致的数据库修改）是否会丢失呢？答案是不会(innodb_flush_log_at_trx_commit=1)。这就是redo log要做的事情，在disk上记录更新

**关于redo log 和bin log日志什么频率刷新到磁盘**
>主要有两个参数来控制，也叫做双一原则，这种设置情况下，单机在任何情况下不会丢失事务更新。

- innodb_flush_log_at_trx_commit =1 每次事务的redo log都直接持久化到磁盘，可以保证MySQL异常重启之后数据不会丢失
- sync_binlog = 1  表示每次事务的binlog 都持久化到磁盘，这个MySQL异常重启之后binlog 不丢失

**sync_binlog =  N：**

N>0 — 每向二进制日志文件写入N条SQL或N个事务后，则把二进制日志文件的数据刷新到磁盘上；

N=0 — 不主动刷新二进制日志文件的数据到磁盘上，而是由操作系统决定；


[![A2l1c6.md.png](https://s2.ax1x.com/2019/04/04/A2l1c6.md.png)](https://imgchr.com/i/A2l1c6)

>每个创建一个表，磁盘都要为改变表分配一定的存储空间，关于这个问题可以查找下MySQL存储引擎的表空间







对一个数据库当中的数据进行update 或者insert 后，是先到内存的，同时磁盘 


# 缓冲池buffer pool ，Doublewrite Buffer，double write,  磁盘数据文件

下面先给出他们之间的关系图情况, 整个流程总体描述一个新修改update, insert 等操作导致的脏页过程是如何刷新到磁盘当中，完成一个事务的过程，


1. 在对缓冲池的脏页进行刷新时，并不直接写入磁盘，而是会通过memcpy函数将脏页先复制到内存中的doublewrite buffer，
2. 通过doublewrite buffer 再分两次，每次1MB顺序地写入共享表空间的物理磁盘double write上
3. 然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。
4. 在这个过程中，因为double write页是连续的，因此这个过程是顺序写的。 记住double write是在磁盘当中的了，只不过是在共享表空间，而不是数据文件
5. 所有在将double write buffer里面数据分两次copy到 double buffer共享表空间，最后到在进行一次fsync到磁盘数据文件




>innodb存储存储引擎是通过数据页来存储的，一个数据页当中可以有多个记录
 


# 为何要采用double write

>两次写主要带来的是数据页的可靠性,解决部分写失效的问题

1. 数据库发生了宕机时，某个页只写了一部分，称为部分写失效
2. 当写入失效时，先通过页的副本来还原，再进行重做，这就是double write 


下面通过图片的形式查看具体写入的过程
![A2lJBD.png](https://s2.ax1x.com/2019/04/04/A2lJBD.png)


>dirty page代表被update过后的脏页，修改过的数据,也就是存在buffer pool缓存当中数据

1. copy过程，操作系统crash, 重启之后，脏页未刷到磁盘，但更早的数据并没有发生损坏，重新写入即可
2. write到共享表空间过程中，操作系统crash，重启之后，脏页未刷到磁盘，但更早的数据并没有发生损坏，重新写入即可
3. write到独立表空间过程中，操作系统crash，重启之后，发现：（1）数据文件内的页损坏：头尾checksum值不匹配（即出现了partial page write的问题）。从共享表空间中d的doublewrite segment内恢复该页的一个副本到数据文件，再应用redo log；（2）若页自身的checksum匹配，但与doublewrite segment中对应页的checksum不匹配，则统一可以通过apply redo log来恢复。
4. recover 过程中，操作系统crash, 重启后，更新并没有成功刷新到磁盘当中，这个时候我们可以直接利用redo 文件




# 案例
比如 update user set name="liuxiongcheng" where id=5; 这个一条SQL语句为一个事务案例

1. 事务开始
2. 针对id =5这条数据上排他锁(id是主键索引，保证走索引的行锁)，并且给5两边的间隙加上gap锁，行锁和间隙锁组成next-key log 防止幻读，也就是防止别的事务insert 一条新的数据
3. 对需要修改的数据页PIN 从磁盘复制到内存的缓冲区(innodb_buffer_cache)
4. 将id =5的数据unddo log，先写入撤销日志
5. 记录修改id=5的日志到redo log buffer
6. 修改id=5d的name="liuxiongcheng"，处于二阶段提交协议的第一阶段prepare
7. 然后写入bin log
8. commit， 将事务提交给执行器，


[![V1jCF0.md.png](https://s2.ax1x.com/2019/06/01/V1jCF0.md.png)](https://imgchr.com/i/V1jCF0)


# 有了redo 为何还需binlog 

从上面的案例可以引出一个问题为何要redo和bin两种日志？

每一个事务redo log对应一个binlog


这样做主要是为了crash recovery 和主从备份的安全性，缺一不可。 下面分几种情况说明下

1. prepare阶段，redo log落盘前，mysqld crash， **也就是redo log 和binlog都没有成功写完**
2. prepare阶段，redo log落盘后，binlog落盘前，mysqld crash，**也就是说redolog成功写完， binlog没有成功写完**
3. commit阶段，binlog落盘后，mysqld crash， **也就是redo和binLog 两个文件都写完了的情况**

针对第一种情况，主库修改还没有正在刷新到磁盘上，我们可以直接利用redo重做即可，不会造成数据不一致性

针对第二种情况，当没有成功写入binLog, 

# 两阶段提交源码

XA 两阶段提交协议，Binlog作为事务协调者，

**关系**

1. redo, prepare
2. commit ,包括两个步骤，实质上BinLog文件的修改
    - write/sync Binlog；
    - InnoDB commit 
    




1. prepare 阶段，写入redo log， lsn 代表redo log文件的写入位置序号，InnoDB 事务进入 prepare 状态, 

```java

innobase_xa_prepare  // InnoDB Prepare 
    trx_prepare_for_mysql   //事务层Prepare
        {
        trx->op_info = "preparing"; //设置事物的操作状态为preparing
        trx_prepare(trx);  ////事物Prepare
            trx_undo_set_state_at_prepare()  //设置undo状态为Prepare
                undo->state = TRX_UNDO_PREPARED；
                undo->xid   = trx->xid;
            trx->state = TRX_STATE_PREPARED  //设置事物状态为prepare阶段
            trx_flush_log_if_needed(lsn, trx);  //刷新redo日志
                {
                trx->op_info = "flushing log";
                trx_flush_log_if_needed_low(lsn); //刷新到指定lsn的redo
                    log_write_up_to()    //实际刷新redo的函数
                trx->op_info = "";
                }
        trx->op_info = "";  //设置事物的操作状态为""
        }
```
**控制redo log buffer 写入磁盘的频率， 一般我们是设置为1**
innodb_flush_log_at_trx_commit参数指定了InnoDB redo刷新的策略，在trx_flush_log_if_needed_low函数中体现，说白了就是在写入效率和数据安全性方面的一个权衡
0：表示不刷新redo，啥也不干，写入效率比较高，但是数据安全性比较低，
1：刷新redo，同时刷新日志到磁盘，然后将buffer pool当中的dirty page 落盘，写入效率比较低，当时数据安全性比较高
2：刷新redo但落盘由操作系统控制


2. trx_flush_log_if_needed_low(lsn); //刷新到指定lsn的redo,lsn代表redo log的写入位置问题

```c
trx_flush_log_if_needed_low(
/*========================*/
    lsn_t    lsn)    /*!< in: lsn up to which logs are to be
            flushed. */
{
    switch (srv_flush_log_at_trx_commit) {
    case 0:      //参数设置为0时不做任何操作
        /* Do nothing */
        break;
    case 1:      //参数为1时，flush redo且写到磁盘
        /* Write the log and optionally flush it to disk */
        log_write_up_to(lsn, LOG_WAIT_ONE_GROUP,
                srv_unix_file_flush_method != SRV_UNIX_NOSYNC);
        break;
    case 2:      //参数为2时，flush redo，但不写到磁盘
        /* Write the log but do not flush it to disk */
        log_write_up_to(lsn, LOG_WAIT_ONE_GROUP, FALSE);
        break;
    default:
        ut_error;
    }
} 
```

3. commit阶段，binlog 写盘，InooDB 事务进入 commit 状态


每个事务binlog的末尾，会记录一个 XID event，标志着事务是否提交成功，该标记用于recovery 过程中，binlog 最后一个 XID event 之后的内容都应该被 purge。




# checkpoint机制
思考之前的数据安全性问题，当出现宕机时，我们需要依靠重做日志进行recovery, 那么这个时候就会带来一个问题， redo log 是一个顺序写入的文件，那不可能将redo log里面记录的所有操作都重做一遍，这种效率是非常低效的情况，那么肯定要思考对这种情况进行一个优化，让recovery 工作有规律可循，知道从哪个位置处的redo log记录的事务是已经flush到磁盘的了,  哪一部分是由于故障丢失的，宕机导致的没有及时flush到磁盘， 这就引入了checkpoint 机制,，当然mysql checkPoint 机制还不止上述一个功能，innobd存储引擎还可以通过checkpoint来控制buffer pool 里面dirty page flush到磁盘操作。

checkppoint机制只要有下面三种作用
1. 缩短数据库recovery 的时间，避免重做所有的redo log事务
2. 缓冲池buffer pool 不够用时， 将dirty page刷新到磁盘，
3. 重做日志不可用时，刷新脏页


工作原理
1. 当数据库发生宕机时，数据库不需要重做所有的日志， 因为Checkpoint之前的页都已经刷新回磁盘。数据库只需对Checkpoint后的重做日志进行恢复，这样就大大缩短了恢复的时间。
2. 当缓冲池不够用时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘
3. 当重做日志出现不可用时，因为当前事务数据库系统对重做日志的设计都是循环使用的，并不是让其无限增大的，重做日志可以被重用的部分是指这些重做日志已经不再需要，当数据库发生宕机时，数据库恢复操作不需要这部分的重做日志，因此这部分就可以被覆盖重用。如果重做日志还需要使用，那么必须强制Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置。


同redo log一样，checkpoint是通过LSN序号来标记的，

**两种checkpoint **

在innodb里面有两个中 checkpoint机制，分别是Sharp CheckPoint, Fuzzy Checkpoint机制，。但是若数据库在运行时也使用Sharp Checkpoint，那么数据库的可用性就会受到很大的影响。故在InnoDB存储引擎内部使用Fuzzy Checkpoint进行页的刷新，即只刷新一部分脏页，而不是刷新所有的脏页回磁盘。

**Sharp CheckPoint**
Sharp Checkpoint 发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，即参数innodb_fast_shutdown=1。将所有的数据flush到磁盘，效率比较低效，故在InnoDB存储引擎内部使用Fuzzy Checkpoint进行页的刷新，即只刷新一部分脏页，而不是刷新所有的脏页回磁盘。

**Fuzzy Checkpoint**
懒惰方式checkpoint也分一下几种方式
1. Master Thread Checkpoint, 以每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘，这个过程是异步的，此时InnoDB存储引擎可以进行其他的操作，用户查询线程不会阻塞。

2. FLUSH_LRU_LIST checkpoint, 采用LRU 算法保证Buffer pool里面至少有

3. Async/Sync Flush checkpoint,用在重做日志不可用的时候，情况


4. Drity Page too much,
即脏页的数量太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总的来说还是为了保证缓冲池中有足够可用的页。其可由参数innodb_max_dirty_pages_pct控制：

innodb_max_dirty_pages_pct值为75表示，当缓冲池中脏页的数量占据75%时，强制进行Checkpoint，刷新一部分的脏页到磁盘。在InnoDB 1.0.x版本之前，该参数默认值为90，之后的版本都为75。

# 总结

上面的这个存储过程也可以总结：先写入日志再写入到磁盘，一个事务采用的是两阶段提交的东西

通过这种机制， Innodb 通过事务日志将随机I/O 变成顺序I/O， 这大大提高了Innodb 写入时的性能问题


# 意义

数据库的存储写入采用WAL技术，下面是维基百科对WAL技术的简单介绍
>In computer science, write-ahead logging (WAL) is a family of techniques for providing atomicity and durability (two of the ACID properties) in database systems. The changes are first recorded in the log, which must be written to stable storage, before the changes are written to the database.

针对mysql innodb关系型支持事务型的数据库来说

1. 先持久化日志，再持久化数据，这样当系统崩溃时，虽然数据没有持久化，但是Redo Log已经持久化，我们可以通过redo log 重做日志实现。


更多

[MySQL · 引擎特性 · WAL那些事儿](http://mysql.taobao.org/monthly/2018/07/01/)


# 参考链接

- [mysql存储引擎InnoDB插入数据的过程详解](https://blog.csdn.net/tangkund3218/article/details/47361705)

- [MySQL技术内幕InnoDB存储引擎学习笔记(第二章)](https://blog.csdn.net/lanonola/article/details/51912534)

- [MySQL · 源码分析 · binlog crash recovery](http://mysql.taobao.org/monthly/2018/07/05/)
