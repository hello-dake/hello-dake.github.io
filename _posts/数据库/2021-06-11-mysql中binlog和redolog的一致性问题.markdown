---
layout: post
title: "mysql中binlog和redolog的一致性问题"
categories: 数据库
date: 2021-06-11
---

#### mysql一共有三种log，binlog，redolog，undolog，每种log分别是用来干什么的？

binlog：一种二进制文件，记录对mysql的insert update delete操作，是mysql server的，不依赖于底层存储引擎，用来做数据恢复和主从复制；

redolog：是InnoDB引擎的，为了实现事务的持久性，对于mysql的每一个操作，都被持久化到磁盘，不可以再回退；

undolog：是mvcc中对于每一行数据都有一个DB_PTR的指针指向该数据行上一个版本，形成一个类似版本链的概念，可以用来做数据回滚。

#### redolog和binlog的对比

| |binlog|redolog |
| :---: | :----:| ---: |
|产生方式 | mysql server产生| 存储引擎产生|
|记录形式| 二进制，逻辑日志，通过sql记录修改| 物理日志，记录的是对磁盘中每一个数据页的修改|
|记录的时间点不同| 事务提交后一次性写入 | 针对一个事务会有多个不连续的记录|

#### mysql是先写binlog还是先写redolog呢？怎么维护保证binlog和redolog的一致性？

先写redolog，再写binlog，再在redolog中写入事务commit标签

mysql内部使用了XA（2PC）来保证binlog和redolog的一致性，具体做法如下：

1.prepare阶段。此时sql已经成功执行，生成了事务ID信息以及redolog和undolog的内存日志，此阶段会写入redolog，但是redolog中只是记录了事务的修改操作，
但是没有记录commit操作，此时事务是prepare状态，所以不会对binlog进行操作；

2.commit阶段。这个阶段又分为两个步骤，第一步写入mysql binlog，先调用write()将内存binlog日志写入文件系统缓存，再调用fsync()将缓存写入磁盘；第二步commit事务，
所做的操作就是在redolog中标记事务为commit状态。

#### 故障恢复是怎么做的？

1.如果写入redolog还没来得及写入binlog crash，数据库会发现redolog中并没有commit标签，binlog中也没有记录，数据库认为该事务没有成功，会回滚操作；

2.如果写入redolog，也写入了binlog，但是还没来得及给redolog中标记commit crash，虽然redolog中该事务没有commit标签，但是数据库仍然认为该事务提交了，
因为在上面的两阶段中，binlog写入就认为提交了，数据库扫描binlog文件，取出事务id和redolog中没有commit的事务id（即redolog中prepare状态的事务id）对比，如果binlog中
存在，那么提交该事务，并在redolog中写入commit记录。
