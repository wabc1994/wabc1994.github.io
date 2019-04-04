---
layout: post
title:  "Innodb存储引擎如何Update和Select"
date:   2019-03-25 22:14:54
categories: jekyll
tags: jekyll RubyGems
comments: true
markdown: kramdowm
kramdonw:
    input: GFM
tags:
    - MySQL
    - innodb
---

* content
{:toc}

# 存储引擎进行update

1. 重做日志缓冲区redo log buffer (在内存当中，一个事务当行没执行一个sql语句都写入一条日志) 是一个全局变量
2. 重做日志文件redo log file (一个事务完成后，将重做日志缓冲区落地到磁盘)， 每个事务提交时会将重做日志缓冲刷新到重做日志文件，具体重做日志缓冲区多久一次write 到 redo log file，这个频率是可以设置的，根据实际的生产需求
3. 缓冲池buffer poll，(放着数据页，在内存当中，当对数据库进行修改是，先修改缓冲池当中的，然后再修改)
2. 双缓冲区 Doublewrite buffer 
3. 磁盘当中的数据文件.idb，表空间
6. binlog  二进制文件，主要是存储在服务器层面，binlog 主要是作为redo log 实现数据库事务二阶段提交的协调者角色，处于redo log 第一阶段的prepare之后，就写入binlog,然后就是事务的commit 
7. double write



**关于redo log 和bin log日志什么频率刷新到磁盘**
>主要有两个参数来控制，也叫做双一原则

- innodb_flush_log_at_trx_commit =1 这两个参数控制MySQL
- sync_binlog = 1  表示事务写入binlog 并使用fsync() 同步到磁盘的

![](https://github.com/wabc1994/wabc1994.github.io/blob/master/img/myinnodb/Innodb.png)

>每个创建一个表，磁盘都要为改变表分配一定的存储空间，关于这个问题可以查找下MySQL存储引擎的表空间





比如insert一条新的数据，是先修改缓冲区的数据页，造成脏页，然后写入重做日志缓冲区， 然后将脏页采用double write 写入磁盘即可

对一个数据库当中的数据进行update 或者insert 后，是先到内存的，同时磁盘 


# 缓冲池，Doublewrite Buffer，double write 磁盘数据文件

下面先给出他们之间的关系图情况, 整个流程总体描述一个新修改update, insert 等操作导致的脏页过程是如何刷新到磁盘当中，完成一个事务的过程，


1. 在对缓冲池的脏页进行刷新时，并不直接写入磁盘，而是会通过memcpy函数将脏页先复制到内存中的doublewrite buffer，
2. 通过doublewrite buffer 再分两次，每次1MB顺序地写入共享表空间的物理磁盘double write上
3. 然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。
4. 在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的。


![流程图](https://github.com/wabc1994/InterviewRecord/blob/master/database/pic/%E5%AD%98%E5%82%A8%E8%BF%87%E7%A8%8B.jpeg)


>innodb存储存储引擎是通过数据页来存储的，一个数据页当中可以有多个记录
 


# 为何要采用double write


>两次写主要带来的是数据页的可靠性

1. 数据库发生了宕机时，某个页只写了一部分，称为部分写失效
2. 当写入失效时，先通过页的副本来还原，再进行重做，这就是double write 


# 案例
比如 update user set name="liuxiongcheng" where id=5; 这个一条SQL语句为一个事务案例

1. 事务开始
2. 针对id =5这条数据上排他锁(id是主键索引，保证走索引的行锁)，并且给5两边的间隙加上gap锁，行锁和间隙锁组成next-key log 防止幻读，也就是防止别的事务insert 一条新的数据
3. 对需要修改的数据页PIN 从磁盘复制到内存的缓冲区(innodb_buffer_cache)
4. 将id =5的数据unddo log，先写入撤销日志
5. 记录修改id=5的日志到redo log buffer
6. 修改id=5d的name="liuxiongcheng"，处于二阶段提交协议的第一阶段prepare
7. 然后写入bin log
8. commit， 将事务提交给执行器


从上面的案例可以引出一个问题为何要redo和bin两种日志

这样做主要是为了

# 总结

上面的这个存储过程也可以总结： 先写事务日志再写入磁盘，一个事务采用的是两阶段提交的东西

通过这种机制， Innodb 通过事务日志将随机I/O 变成顺序I/O， 这大大提高了Innodb 写入时的性能问题


# 意义

数据库的存储写入采用WAL技术，下面是维基百科对WAL技术的简单介绍
>In computer science, write-ahead logging (WAL) is a family of techniques for providing atomicity and durability (two of the ACID properties) in database systems. The changes are first recorded in the log, which must be written to stable storage, before the changes are written to the database.

1. 先持久化日志，再持久化数据，这样当系统崩溃时，虽然数据没有持久化，但是Redo Log已经持久化，我们可以通过redo log 重做日志实现。



# 参考链接

- [mysql存储引擎InnoDB插入数据的过程详解](https://blog.csdn.net/tangkund3218/article/details/47361705)

- [MySQL技术内幕InnoDB存储引擎学习笔记(第二章) - lanonola的专栏 - CSDN博客](https://blog.csdn.net/lanonola/article/details/51912534)
