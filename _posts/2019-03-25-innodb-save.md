---
layout: post
title:  "数据库存储引擎如何Update和Select"
date:   2019-03-25 22:14:54
categories: MySQL
comments: true
tags:
    - MySQL
---

* content
{:toc}


# 前言
数据库存储引擎是如何工作的，本文主要从关系型数据MySQL Innodb和非关系数据leveldb key value 键值数据库来讲解。无论是关系型还是非关系型数据库，一个优秀的数据存储组织者- 存储引擎都需要考虑下面个点

- 索引组织
- 缓存管理， 冷热数据分开存储
- 事务管理， 高可用等方案，崩溃恢复， 数据一致性ACID 等
- 查询优化，无论是结构化的SQL语句还是key-value的put(), delete等简单查询


# 数据记录是如何存储在innodb引擎

- 首先，在innobd当中，数据库的物理存储是会分成多个页，以页为单位，多个记录存储在一个页当中，数据每一次I/O, 读取的单位是一个页。
- 类似于磁盘的预读多读机制，innodb一次读取数据也是对读取周边部分的数据的，局部性原理，起到缓存机制作用
- 数据库运行时，会在内存中为何一公分buffer pool， 缓存部分页。在每次对数据进行select或者update，如果数据在内存当中，比如 select 就可以查询缓存，直接返回， 不用进行磁盘I/O, 否则才进行低效率的I/O； 如果是update操作的话，先将数据页加载内存，然后对内存里面的数据进行修改，当然在对内存进行修改的过程伴随着一系列的redo log和bin log 操作， 这个是我们下面介绍的重点，理解了这个过程，我们也就理解了innodb到底是如何工作的


**innodb存储内存**
InnoDB存储引擎内存由以下几个部分组成：
- 缓冲池（buffer pool）
- 重做日志缓冲池（redo log buffer）
- 内存池（additional memory pool）

别由配置文件中的参数innodb_buffer_pool_size、innodb_log_buffer_size、innodb_additional_mem_pool_size的大小决定

其中最重要的是要理解Buffer Pool 缓存冲池中缓存的数不仅据页类型有：索引页、数据页、undo页、插入缓冲（insert buffer），自适应哈希索引（adaptive hash index）、InnoDB存储的锁信息（lock info）、数据字典信息（data dictionary）等，缓冲池有数据页和索引页，只是他们占缓冲池的很大部分，InnoDB存储引擎中内存的结构如下图：

[![ZS64E9.md.png](https://s2.ax1x.com/2019/06/21/ZS64E9.md.png)](https://imgchr.com/i/ZS64E9)




**修改流程， 先判断数据是否在buffer pool当中，下面的图标没有画出redo log和bin log的修改操作，实质上采用的WAL技术，先走日志文件，再buffer pool 修改数据行记录，最后在必要的时候再讲dirty page flush到磁盘的情况， 完成整个持久化的机制**


[![V1qsdP.md.png](https://s2.ax1x.com/2019/06/01/V1qsdP.md.png)](https://imgchr.com/i/V1qsdP)


先从整体上来讲，innodb的一条事务日志经历四个阶段：
1. 创建阶段：事务创建一条日志
2. 日志落盘：将日志写入磁盘的日志文件，当然在这里面可以根据应用的一致性和磁盘I/0次数效率考虑，要求控制日志落盘的时间和间隙，
3. 数据落潘，事务日志对应修改的dirty page 数据写入磁盘的数据文件，当然同日志文件一样，buffer pool当中的dirty page也不是每执行一个事务就fsync到磁盘，这样磁盘I/0的次数就会非常高，为了效率都是多个事务或者dirty page数量达到一定的程度再几种fsync到磁盘当中去，
4. 写入CKP: 日志被当做checkpoint 写入日志文件

上面四个步骤创建的相关日志文件可以使用下面的图标标记出来

 [![V3AE6I.md.png](https://s2.ax1x.com/2019/06/01/V3AE6I.md.png)](https://imgchr.com/i/V3AE6I)


# 如何管理Buffer pool 

Innodb存储引擎管理Buffer Pool 实质上管理方式也是跟操作系统管理内存一个道理，不可能将所有的数据都放在内存当中，必然就会涉及到内存块的替换规则， 但是由于磁盘的局部性原理，特别是数据库当中数据是16k为页的大小进行读取的，因此一般读取的时候回读取目标页加周边页的数据一起到内存当中，这个特性也决定了我们不能粗放直接得使用操作管理内存的方式，比如基于双向链表的Least recent used(LRU)算法, 热点的数据页放在链表头，冷的数据放在链表后面，更低级别的数据直接放在磁盘当中，对内存的当中数据使用一定的淘汰策略。

传统的操作内存淘汰策略LRU算法在MySQL 内存管理当中是失效的，主要有下面个知识点
1. 预读失效
2. 缓冲池污染

**什么是预读机制？**
由于是预读机制(Read-ahead)，提前把页放到缓冲池，但最终MySQL 并没有从页当中读取数据，称为预读失效。

**如何对预读失效进行优化**
要优化预读失效，思路是：
1. 让预读失败的页，停留在缓冲池LRU里的时间尽可能短
2. 让真正被读取的页，才挪到缓冲池LRU的头部

具体方法是：
1. 将LRU分为两部分：
- 新生代(new subList) 方法
- 老年代(old sublist) 方法

![ZShsZ8.png](https://s2.ax1x.com/2019/06/21/ZShsZ8.png)

2. 新老生代收尾相连，即：新生代的尾(tail)连接着老生代的头(head)；

3. 新页（例如被预读的页）加入缓冲池时，只加入到老生代头部
- 如果数据真正被读取(预读成功)，才会被加入到新生代的头部
- 如果数据没有被读取，则会被新生代里的“热数据页”更早被淘汰出缓冲池


**预读失效**
innodb为了防止缓存污染：

>也就是说，在我们扫描数据量比较大的一个表时，有可能将整个表的数据都缓存到LRU链表里，淘汰掉其他有用的数据，为了防止这种情况，innodb引入了一个机制：先将数据移动到旧LRU链表中，然后如果超过了1S，那么就会移动到热数据LRU里。


**什么叫MySQL缓冲污染池**
当一个SQL语句，要批量扫面大量数据时，可能会把缓冲池的所有页都替换出去，导致大量热数据被换出，MYSQL性能下降得很快，这种情况叫做缓冲池污染池。


**MySQL如何解决缓冲池失效的问题**
在移动数据时，会把数据移动到旧LRU链表的头部，当出现全表扫描时，InnoDB先将数据块载入到旧LRU链表上，程序读取数据块，因为这时，数据块在旧LRU链表中的停留时间还不到innodb_old_blocks_time（1S），所以不会被转移到热LRU链表中这就避免了Buffer Pool被污染的情况。

**分为多种循环链表管理不同的buffer pool**
LRU链表包含所有读入内存的数据页：
- Flush_list包含被修改过的脏页；
- unzip_LRU包含所有解压页；
- Free list上存放当前空闲的block,当前空闲的内存块，

# 存储引擎进行update

1. 重做日志缓冲区redo log buffer (在内存当中，一个事务当行没执行一个sql语句都写入一条日志) 是一个全局变量
2. 重做日志文件redo log file (一个事务完成后，将重做日志缓冲区落地到磁盘)， 每个事务提交时会将重做日志缓冲刷新到重做日志文件，具体重做日志缓冲区多久一次write 到 redo log file，这个频率是可以设置的，根据实际的生产需求
3. 缓冲池buffer poll，(放着数据页，在内存当中，当对数据库进行修改是，先修改缓冲池当中的，然后再修改)
2. 双缓冲区 Doublewrite buffer 
3. 磁盘当中的数据文件.idb，表空间
6. binlog 二进制文件，主要是存储在服务器层面，binlog 主要是作为redo log 实现数据库事务二阶段提交的协调者角色，处于redo log 第一阶段的prepare之后，就写入binlog,然后就是事务的commit 
7. Double write

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


数据库存储引擎是如何工作的，本文主要从关系型数据MySQL Innodb和非关系数据leveldb key value 键值数据库来讲解。


# leveldb put和delete操作
与关系型数据innodb采用B+树不同，键值型数据库level db主要采用
log structure merger tree 来作为存储结构， 充分将内存和磁盘的不同特点。
## LSMT特点 
 “LSMT”的架构提供了一些有趣的行为： 写入操作总是很快速，并且不用考虑数据集的大小（追加）并是写入到内存当中的memtable文件； 随机读取或者通过读取内存或者通过一次快速的磁盘寻道来完成 。然而，该如何处理更新和删除操作呢？
 
 
但是内存当中memtable 最后是要持久化到磁盘sstable, 一旦SSTable被写入磁盘，它就不再可变，因此，更新和删除操作不能通过操作SSTable来实现；即使sstable可变，我们也要经历一个漫长磁盘的搜索过程，查找某个特定的key, 这个速度是很慢的，并且要经过磁盘寻道和旋转等耗时操作.

**结论**
因此leveldb 优化update 和delete 开销的主要思路是如何避免磁盘查找，尽量跟写入操作一样，写入到内存操作。
 
## 解决方案
leveldb当中解决这个问题很巧妙的做法就是将update和delete操作都变成Put操作,我们知道put操作是写入到内存当中的memtable,写入速度很快，

比如删除一个sstable里<key,value>，我们调用update入口后，更新操作会通过write操作往memtable 追加一条激励<key, 新的value> ， '墓碑'标记就代表这条记录是作废了，如果是delete 操作，我们就往memtable里面写入记录
<key,’墓碑‘>操作，在后期的读操作会获取到更新的或有墓碑标记的记录，而不会获取到旧值；


puth和delete的实现都是通过封装Write来实现的，函数调用关系如下：

- leveldb:DBImpl:put => leveldb:DB:Put=>leveldb::DBImpl::write
- leveldb:DBImpl:Delete => leveldb:DB:Delete=>leveldb::DBImpl::write

**疑问**
这也会到带一个问题，如果频繁更新某个key, 那么岂不是memetable对同一个key会存在很多个<key,'墓碑'>,或者多个无效的旧数据版本,堆积起来也是占据一定的系统存储空间，因此有必要定期或者定量对sstable里面过期的数据进行清理，这就是compression 压缩要做的工作了, compaction process 压缩工作进程会进行合并sstable和去除对应旧数据版本，因此利用LSMT 树作为存储结构的数据一般都具有 在磁盘当中的Compaction 工作步骤，作用可以总结为下面两个点

- 为了丢弃不再被使用的旧版本数据，清理他们所占据的空间
- 为了控制LSMT树的层次结构， 一般LSM树的形状都是层次越低(从高到底), 数据量，少量热数据存储在高层次的树，大量的冷数据存储在低层次的树当中， 这么做主要是为了优化读性能

[![ZVQpUP.md.png](https://s2.ax1x.com/2019/06/25/ZVQpUP.md.png)](https://imgchr.com/i/ZVQpUP)

在leveldb当中Compaction会从大的类别分为两种， compactions 分别是
1. MinorCompaction, 指的是immutable memtable 持久化到为sst 文件
2. Major Compaction, 指的是sst文件之间的merger和compaction

其中删除旧版本无效数据就放在major compaction 过程当中，

```java
void WriteBatch::Delete(const Slice& key) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);//操作总数加1
  rep_.push_back(static_cast<char>(kTypeDeletion));//写入操作类型
  PutLengthPrefixedSlice(&rep_, key);//写入待删除数据的key值
}
```

## 为何需要Compaction

compaction是leveldb中最核心的东西了。

1. 前面我们说过，当用户调用delete删除一个键值时，leveldb并没有真正把它删除掉，而只是简单将这个键值对的type标记为kTypeDeletion，然后和正常的键值对一样写盘。这个特点使得leveldb的写操作很快，但是问题也是很显然的，就是会造成大量无效数据，占用磁盘空间。
2. 除此之外，leveldb添加数据时并不会将过期的key-value覆盖，而是通过序列号将其当成完全不同的key写入进去，因此会使得系统中存在很多过期数据。，毫无疑问这也是很占用空间的。

而compaction操作可以解决这些问题。通过compaction，leveldb可以将过期的key丢弃，而且在一定条件下丢弃标记为kTypeDeletion的数据。同时通过compaction控制了每层的文件数目。可以说compaction是leveldb的写操作得以高效的主要原因。


**compaction如何操作**
对于磁盘当中的多个层次间的LSMT树我们该如何进行压缩，才能有利于我们进行高效地读取数据，让冷热数据从上层开始到下层，这样读取出来也是比较好的处理方式

关于这部分内容可以参考下阿里提出的数据库存储引擎X-Engine，是如何对这部门内容进行一个优化的，

# leveldb源码

数据库定义文件
```c
class DB {
 public:
  // Open the database with the specified "name".
  // Stores a pointer to a heap-allocated database in *dbptr and returns
  // OK on success.
  // Stores NULL in *dbptr and returns a non-OK status on error.
  // Caller should delete *dbptr when it is no longer needed.
  static Status Open(const Options& options,
                     const std::string& name,
                     DB** dbptr);
// 构造函数
  DB() { }
  // 析构函数
  virtual ~DB();
  // Set the database entry for "key" to "value".  Returns OK on success,
  // and a non-OK status on error.
  // Note: consider setting options.sync = true.
  virtual Status Put(const WriteOptions& options,
                     const Slice& key,
                     const Slice& value) = 0;
  // Remove the database entry (if any) for "key".  Returns OK on
  // success, and a non-OK status on error.  It is not an error if "key"
  // did not exist in the database.
  // Note: consider setting options.sync = true.
  virtual Status Delete(const WriteOptions& options, const Slice& key) = 0;

  // Apply the specified updates to the database.
  // Returns OK on success, non-OK on failure.
  // Note: consider setting options.sync = true.
  virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;

  // If the database contains an entry for "key" store the
  // corresponding value in *value and return OK.
  //
  // If there is no entry for "key" leave *value unchanged and return
  // a status for which Status::IsNotFound() returns true.
  //

//先利用布隆过滤器查找是sstable否存在某个值key，
  virtual Status Get(const ReadOptions& options,
                     const Slice& key, std::string* value) = 0;

  // Return a heap-allocated iterator over the contents of the database.
  // The result of NewIterator() is initially invalid (caller must
  // call one of the Seek methods on the iterator before using it).
  //
  // Caller should delete the iterator when it is no longer needed.
  // The returned iterator should be deleted before this db is deleted.
  virtual Iterator* NewIterator(const ReadOptions& options) = 0;

  // Return a handle to the current DB state.  Iterators created with
  // this handle will all observe a stable snapshot of the current DB
  // state.  The caller must call ReleaseSnapshot(result) when the
  // snapshot is no longer needed.
  virtual const Snapshot* GetSnapshot() = 0;

  // Release a previously acquired snapshot.  The caller must not
  // use "snapshot" after this call.
  virtual void ReleaseSnapshot(const Snapshot* snapshot) = 0;

  // DB implementations can export properties about their state
  // via this method.  If "property" is a valid property understood by this
  // DB implementation, fills "*value" with its current value and returns
  // true.  Otherwise returns false.
 
 
   // Valid property names include:
  //
  //  "leveldb.num-files-at-level<N>" - return the number of files at level <N>,
  //     where <N> is an ASCII representation of a level number (e.g. "0").
  //  "leveldb.stats" - returns a multi-line string that describes statistics
  //     about the internal operation of the DB.
  //  "leveldb.sstables" - returns a multi-line string that describes all
  //     of the sstables that make up the db contents.
  //  "leveldb.approximate-memory-usage" - returns the approximate number of
  //     bytes of memory in use by the DB.
  virtual bool GetProperty(const Slice& property, std::string* value) = 0;

  // For each i in [0,n-1], store in "sizes[i]", the approximate
  // file system space used by keys in "[range[i].start .. range[i].limit)".
  //
  // Note that the returned sizes measure file system space usage, so
  // if the user data compresses by a factor of ten, the returned
  // sizes will be one-tenth the size of the corresponding user data size.
  //
  // The results may not include the sizes of recently written data.
  virtual void GetApproximateSizes(const Range* range, int n,
                                   uint64_t* sizes) = 0;

  // Compact the underlying storage for the key range [*begin,*end].
  // In particular, deleted and overwritten versions are discarded,
  // and the data is rearranged to reduce the cost of operations
  // needed to access the data.  This operation should typically only
  // be invoked by users who understand the underlying implementation.
  //
  // begin==NULL is treated as a key before all keys in the database.
  // end==NULL is treated as a key after all keys in the database.
  // Therefore the following call will compact the entire database:
  //    db->CompactRange(NULL, NULL);
  virtual void CompactRange(const Slice* begin, const Slice* end) = 0;

 private:
  // No copying allowed
  // 外接无法访问其拷贝构造函数
  DB(const DB&);
  void operator=(const DB&);
};

``` 

# 总结
- 上面的这个存储过程也可以总结：先写入日志再写入到磁盘，一个事务采用的是两阶段提交的东西
通过这种机制， Innodb 通过事务日志将随机I/O 变成顺序I/O， 这大大提高了Innodb 写入时的性能问题。


# 意义

数据库的存储写入采用WAL技术，下面是维基百科对WAL技术的简单介绍
>In computer science, write-ahead logging (WAL) is a family of techniques for providing atomicity and durability (two of the ACID properties) in database systems. The changes are first recorded in the log, which must be written to stable storage, before the changes are written to the database.

针对mysql innodb关系型支持事务型的数据库来说

1. 先持久化日志，再持久化数据，这样当系统崩溃时，虽然数据没有持久化，但是Redo Log已经持久化，我们可以通过redo log 重做日志实现。

[MySQL · 引擎特性 · WAL那些事儿](http://mysql.taobao.org/monthly/2018/07/01/)

# 参考链接

- [mysql存储引擎InnoDB插入数据的过程详解](https://blog.csdn.net/tangkund3218/article/details/47361705)

- [MySQL技术内幕InnoDB存储引擎学习笔记(第二章)](https://blog.csdn.net/lanonola/article/details/51912534)

- [MySQL · 源码分析 · binlog crash recovery](http://mysql.taobao.org/monthly/2018/07/05/)

- [阿里技术-X-Engine](https://mp.weixin.qq.com/s/Sif4d8zqKPuhVq25m0SXGA)
