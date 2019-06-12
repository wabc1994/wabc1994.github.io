---
layout: post
title:  "MySQL锁机制总结以及MVCC"
date:   2018-8-15 8:50:9
categories: MySQL
comments: true
tags:
    - MySQL
    - 数据库
---

* content
{:toc}

# 前言 

多版本控制是一个乐观锁，在不同的存储引擎当中具有不同的实现方式， 面试过程当中我们主要是关注innodb当中的实现？
 

**表锁在服务器层实现, 行锁在存储引擎层实现**

**MySQL 会在两个层面做并发控制: 服务器层和存储引擎层**


MySQL锁机制总体概览图

[![AWTo9K.md.png](https://s2.ax1x.com/2019/04/06/AWTo9K.md.png)](https://imgchr.com/i/AWTo9K)


## 乐观锁



**有了数据库锁了我们为何还需要乐观锁**
> 在数据库锁中，在表锁中我们读写是阻塞的，基于提升并发性能的考虑，MVCC一般读写是不阻塞的(所以说MVCC很多情况下避免了加锁的操作)

乐观锁：利用程序处理并发。方式大概有下面几种
1. 对记录加版本号,一个数据库可以有多个版本号
2. 对记录加时间戳

## 悲观锁 
悲观锁一般就是我们通常说的数据库锁机制，下面讨论的都是基于悲观锁，比如表锁，行锁，页锁


在MyISAM中只用到表锁，不会有死锁的问题(MyIsam不支持事务。)，锁的开销也很小，但是相应的并发能力很差(高版本的MySQL基本放弃MyISAM)。innodb实现了行级锁和表锁，锁的粒度变小了，并发能力变强，但是相应的锁的开销变大，很有可能出现死锁。同时inodb需要协调这两种锁，算法也变得复杂。InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。

表锁和行锁都分为共享锁和排他锁（独占锁），而更新锁是为了解决行锁升级（共享锁升级为独占锁）的死锁问题。

innodb中表锁和行锁一起用，所以为了提高效率才会有意向锁（意向共享锁和意向排他锁）


## InnoDB行锁的细分
多个事务操作同一行数据时，后来的事务处于阻塞等待状态。这样可以避免了脏读等数据一致性的问题。后来的事务可以操作其他行数据，解决了表锁高并发性能低的问题

innodb 的行锁是在有索引的情况下，没有索引的表是锁定全表的

1. 共享锁
2. 排他锁
3. 更新锁


**InnoDB什么情况下使用表锁什么情况使用行锁**

InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。
## 意向锁
为了解决表锁和行锁之间存在的问题而映射出来的意向锁,意向锁的本质是为行锁和表锁的共存问题

要申请行锁的事务，数据库自动为我们申请一个意向锁，有了意向锁，就代表了该表有了行锁的存在或者即将偶行级锁的存在，

[意向锁存在的意义](https://www.zhihu.com/question/51513268)

![](https://github.com/wabc1994/InterviewRecord/blob/master/database/pic/%E6%84%8F%E5%90%91%E9%94%81.jpeg)

将锁冲突检测的时间复杂度由O(N)降低为O(1), 有了意向锁，就跟宣誓了自己的领地一样，这个表已经有一个行锁了， 就可以直接阻塞其他尝试获得排他锁的表锁

>Intention locks reduce the number of locks that must be examined when a new lock is allocated. Intention locks allow transactions to quickly determine if rows (or pages) in a given table have been locked by other transactions.

**我自己对意向锁的理解？**

为了减少数据库监测冲突的次数，减少时间复杂度，减少行遍历的次数

# MVCC

  MySQL的InnoDB存储引擎默认事务隔离级别是RR(可重复读), 是通过 "行排他锁+MVCC" 一起实现的, 不仅可以保证可重复读, 还可以防止幻读,行锁加间隙锁组合成为next-key-lock防止幻读。

**这个问题也可以这么问？**

> 既然有了行锁和表锁了，为何要需要更进一步的实现多版本控制

- 大多数的MYSQL事务型存储引擎，比如InnoDB,都不使用一种简单的行锁机制.事实上,他们都和MVCC–多版本并发控制来一起使用，mvcc是一种读不加锁的实现机制。

- 大家都应该知道,锁机制可以控制并发操作,但是其系统开销较大,而MVCC可以在大多数情况下代替行级锁,使用MVCC,能降低其系统开销.
 

innodb 当中的并发控制机制，行锁在一定程度开销比较大，大多数的存储引擎都不适用一种简单的行锁机制。

>innodb 有两种锁机制，一种是行锁，另一种是表锁，这两种锁可以认为是悲观锁来的， 而innodb当中的mvcc 其实是另一种锁思想，乐观锁的实现，乐观锁的实现一般由版本号和时间戳，在更新的同时更新版本号和时间戳。


从用户的角度来看，好像是数据库可以提供同一数据的多个版本。


**什么时候使用mvcc, 什么情况下使用？**

>能不用行锁，能使用mvcc 版本控制可以减少锁的使用，降低系统的开销


## 隔离级别与MVCC
MVCC 只有在RR和CR两个级别下才会工作，并且在这两个级别下导致事务生成Read View不一样，也就是导致了是否能够解决是否可读的问题。

```c
set global transaction isolation level READ COMMITED;

set session transaction isolation level READ COMMITED;
```
## 添加的隐藏列
每行数据存储都会添加三个隐藏列

1. 六个字节的DB\_TRX\_ID:表示最近修改该行数据的事务ID

2. 7个字节的DB\_ROLL\_PTR: 指向rollback segement当中的undo log,找到之前版本的数据就是通过这个列, 数据库的事务系统通过DB\_ROLL\_PTR解析找到undo log， 然后通过undo log找到旧版本事务， 形成一条链

3. 6个字节的DB_ROW_ID


三个隐藏列当中，我们有时候经常说道是两个，主要是因为我们在MVCC当中只需要用到DB\_TRX\_ID 和DB\_ROLL\PTR 

## 事务链表

事务从开始到提交之前，一直保存在一个链表当中，当提交之后才从该链表当中删除

[![AWOZ1P.png](https://s2.ax1x.com/2019/04/06/AWOZ1P.png)](https://imgchr.com/i/AWOZ1P)

>事务编号的大小关系： trx1 >trx2>trx3>trx4 越小的事务是越早开始的情况

## read view 

InnoDB MVCC使用的内部快照的意思。在不同的隔离级别下，事务启动时（有些情况下，可能是SQL语句开始时）看到的数据快照版本可能也不同，

我们知道读分为两种，分别是当前读和一致性读，一致性读也就是利用read view来实现的，说白了就是mvcc

在普通select操作当中都是通过mvcc 不加锁实现高效读数据操作的，除了 select····lock in share mode和select···· for update 这两个，其他的

- update 
- delete
- insert 
- select ··· for update
- select ··· lock in share mode 


这些都是当前读， 当前读都是读取到最新版本的数据，二快照读是通过mvcc快照机制读取到旧版本的数据的
 
1. 在可重读情况下，一个事务当中的所有select语句共用一个read view,也就是说只有在事务的一个select请求才创建read view
2. 在RC隔离级别下，每个SELECT都会获取最新的read view, 也就是每个select 语句不是共用一个read view一致性读视图




通过对别read view 视图当中的版本号

**Read View 是跟SQL语句绑定在一起的，而不是事务**

**read view 数据结构**

**read view {low\_trx\_id,up\_trx\_id,trx\_ids}**,在并发情况下，一个事务启动时。


**第一种表示方法**

![Afxgpj.png](https://s2.ax1x.com/2019/04/07/Afxgpj.png)

比如上图， 对于当前事务启动的瞬间来说， 一个数据版本的row trx\_id, 有以下几种可能

1. 如果落在绿色部分，表示这个版本是已经提交的事务或者当前事务自己生成的，这个数据是可见的

2. 如果落在红色部门，表示这个版本是由将来启动的事务生成的，是肯定不可见的


3. 如果事务是落在未提交事务集合当中，那么包括两种情况
	- 若row trx\_id 在未提交事务集合当中，表示这个版本是由还没提交的事务生成的，不可见
	- 若row trx\_id 不在未提交事务集合当中，表示这个版本是已经提交的事务生成的，可见的


**另一种表示方法**

- low\_trx\_id 表示该事务启动时，当前事务链表中最大的事务id编号，也就是最近创建的除自身以外最大事务编号，比如trx1

- up\_trx\_id 表示事务启动时，当前事务链最小的事务id编号，也就是最近创建的当前系统中还未提交的事务trx4

trx_ids表示所有事务链表中事务的id集合。代表未提交的事务

## 比较版本号算法


我们是通过比较当前操作事务的版本号和事务当中的


在代码体当中创建一个read\_view 的函数主要是

```c
row_search_for_mysql(){
}
```

- 设该行的当前事务id为row trx\_id< 视图中up\_trx
\_id的记录对于当前read view 是一定可见的


## 总体概览图

[![Ah9PCF.md.png](https://s2.ax1x.com/2019/04/07/Ah9PCF.md.png)](https://imgchr.com/i/Ah9PCF)

## 一致性读读请求基于某个时间点得到一份那时的数据快照，而不管同时其他事务对数据的修改。

[![AhSiIU.md.png](https://s2.ax1x.com/2019/04/07/AhSiIU.md.png)](https://imgchr.com/i/AhSiIU)

## 为何要比较版本号
因为并发的实质是多个事务并发执行，操作同一个数据库表，同一个表当中不同行通过不同的创建时间和删除时间，就可以跟是哪个事务操作他的隔离开来， 先操作的事务，或后面再操作的事务情况

## read view创建的时机


1. 对于可重复读，查询只承认在事务启动前就已经提交完成的数据

>if the transction isolation level is REPEATABLE READ ，all consisten reads（一致性读） within the same transaction read the snapshot established by the first such read in that transaction.

2. 对于读提交， 查询只承认在语句启动前就已经提交完成的数据
>With READ COMMITTED isolation level, each consistent read within a transaction sets and reads its own fresh snapshot.
## SELECT比较版本号

select根据下面两个条件来检索每行记录：

- 当前事务号
- 行的创建时间和删除时间
1. innoDB 只会选择那些行的创建时间早于或者等于当前事务号的数据行。这样可以保证确保读取的行，要么是在事务开始之前就已经存在，要么是事务本次插入或者修改过的
2. 行的删除版本要么是未定义的， 要么大于当前事务版本号，这可以确保事务读取到的行，在事务开始之前未被删除(未被定义就是未被删除)**(是后面的事务导致的删除)**

只有满足1，2同时满足的记录，才能返回作为查询的结果


## DELETE

InnoDB会为删除的每一行保存当前系统的版本号(事务的ID)作为删除标识. 


## UPDATE
是直接在表当中生成生成新的一行，

## INSERT

直接在表当中插入新的一行，将当前事务的ID作为该行数据的创建版本号，删除版本号未定，



# 如何理解MVCC
其实mvcc 我们也可以使用 CAS机制来理解的？
也是一种不加锁的， 比如比较当前事务的版本号和创建时间版本号和删除版本号这两个东西，判断我们是否要进行update、select、insert、delete等的行为情况
特别是在查询的时候select操作， select操作的情况


## 当前读和快照读

 1. 快照度（snapshot read）,也就是一致性读 consistent read
 >简单的select操作，不包括(select ····lock in share mode, select····for update)
 
>在RR级别下，快照读是通过MVVC(多版本控制)和undo log（回滚日志)来实现的，
 
 2. 当前读(current read)， 当前读主要有下面几种方式
 	- select ··· lock in share mode
 	- select ··· for update
 	- insert   
 	- update
 	- deletce 
 >加record lock(记录锁)和gap lock (gap lock)来实现的 



**两者的区别**
>innodb在快照读的情况下并没有真正的避免不可重复读(出现幻读，也导致了不可重复读), 但是在当前读的情况下避免了不可重复读和幻读!!! 

# 参考链接

[数据库的锁机制及原理 ](https://blog.csdn.net/C_J33/article/details/79487941)

[MYSQL MVCC 实现机制 ](https://blog.csdn.net/whoamiyang/article/details/51901888)

[MySQL-InnoDB-MVCC多版本并发控制 - 后端 - 掘金](https://juejin.im/entry/5a4b52eef265da431120954b)

[丁奇MySQL45讲]()

[MySQL :: MySQL 8.0 Reference Manual :: 15.7.2.3 Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)

[innodb当前读&快照读](https://blog.csdn.net/shiqining888/article/details/53607916)
