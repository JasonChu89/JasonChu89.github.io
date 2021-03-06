---
layout: post
title: "MySQL binlog"
description: "MySQL binlog"
categories: [MySQL]
tags: [MySQL, binlog]
redirect_from:
  - /2018/11/15/
---

* Kramdown table of contents
{:toc .toc}

MySQL binlog
============

本文转载自公众号   打杂的ZRJ

# 一个定义
binlog是记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的二进制日志。  
binlog不会记录SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改，但你可以通过查询通用日志来查看MySQL执行过的所有语句。  
如果update操作没有造成数据变化，也是会记入binlog。  

# 两个误解
## 误解一:binlog只是一类记录操作内容的日志文件    
因为binlog称之为二进制日志，很多研发会把这个二进制日志和我们平时在代码里写的代码日志联系在一起。因为我们的代码日志，只有一类记录操作容的文件，并不包含索引文件。然而，这个二进制日志包括两类文件：  
索引文件（文件名后缀为.index）用于记录哪些日志文件正在被使用  
日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML(除了数据查询语句)语句事件。  
这么说可能还有一点抽象，假设文件my.cnf中有这么三条配置  
```
log_bin：on 打开binlog日志
log_bin_basename：bin文件路径及名前缀（/var/log/mysql/mysql-bin）
log_bin_index：bin文件index（/var/log/mysql/mysql-bin.index）
```
那么你会在文件目录/var/log/mysql/下面发现两个文件mysql-bin.000001和mysql-bin.index。  
mysql-bin.index就是我们所说的索引文件，打开瞅瞅，内容是下面这样,记录哪些文件是日志文件。  
```
./mysql-bin.000001
```
那么说到日志文件。在innodb里其实又可以分为两部分，一部分在缓存中，一部分在磁盘上。这里业内有一个词叫做刷盘，就是指将缓存中的日志刷到磁盘上。跟刷盘有关的参数有两个:sync_binlog和binlog_cache_size。这两个参数作用如下
```
binlog_cache_size: 二进制日志缓存部分的大小，默认值32k
sync_binlog=[N]: 表示写缓冲多少次，刷一次盘,默认值为0
```
注意两点:
1. binlog_cache_size设过大，会造成内存浪费。binlog_cache_size设置过小，会频繁将缓冲日志写入临时文件。具体怎么设，有兴趣自行查询，我觉得研发大大根本没机会去设这个值的，了解即可。
2. sync_binlog=0:表示刷新binlog时间点由操作系统自身来决定，操作系统自身会每隔一段时间就会刷新缓存数据到磁盘，这个性能最好。sync_binlog=1，代表每次事务提交时就会刷新binlog到磁盘。sync_binlog=N,代表每N个事务提交会进行一次binlog刷新。

另外，这里存在一个一致性问题，sync_binlog=N，数据库在操作系统宕机的时候，可能数据并没有同步到磁盘，于是再次重启数据库，会带来数据丢失问题。   
当sync_binlog=1，事务在Commit的时候，数据写入binlog，但是还没写入事务日志(redo log和undo log)。此时宕机，重启数据库，数据被回滚。但是binlog里已经记录，这里存在不一致问题。这个事务日志和binlog一致性的问题，大家可以查询mysql的内部XA协议，该协议就是解决这个一致性问题的。

## 误解二:binlog是InnoDb独有的
binlog是以事件形式记录的，这句话通俗点说，就是binlog的内容都是一个个的事件。这块具体的我会在下一篇讲，这篇记住binlog的内容就是一个个事件就行。  
注意了，这里的用词，是一个个事件，而不是事务。大家应该知道InnoDB和mysiam最显著的区别就是一个支持事务，一个不支持事务。  
因此你可以说，binlog是基于事务来记录二进制日志，比如sync_binlog=1,每提交一次事务，就写入binlog。你却不能说binlog是事务日志，binlog不仅记录innodb日志，在myisam中，也一样存在binlog。  

# 三个用途
这三个用途，出自《MySQL技术内幕 InnoDB存储引擎》一书，分别为恢复、复制、审计。这三个用途，研发大大们了解一下即可，比如数据恢复，你碰到同事删库的机会实在太少。假如真的有同事舍己为人，冒着离职的风险给你提供做数据恢复的机会，大把运维工程师待命在那，轮不到你的。所以，这三个功能了解即可。

恢复：这里网上有大把的文章指导你，如何利用binlog日志恢复数据库数据。如果你真的觉得自己很有时间，就自己去创建个库，然后删了，再去恢复一下数据，练练手吧。

复制: 如图所示（图片不是自己画的，偷懒了）  

![](/upload_imgs/2018111509155918422.png)

主库有一个log dump线程，将binlog传给从库
从库有两个线程，一个I/O线程，一个SQL线程，I/O线程读取主库传过来的binlog内容并写入到relay log,SQL线程从relay log里面读取内容，写入从库的数据库。  

审计：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入攻击。

# 四个常识
常识一:binlog常见格式
这块知识我用一个表格来表示，没必要啰嗦一大堆。

业内目前推荐使用的是row模式，准确性高，虽然说文件大，但是现在有SSD和万兆光纤网络，这些磁盘IO和网络IO都是可以接受的。  
那么，大家一定想问，为什么不推荐使用mixed模式，理由如下
假设master有两条记录，而slave只有一条记录。  
![](/upload_imgs/2018111509311882088.png)
当在master上更新一条从库不存在的记录时，也就是id=2的记录，你会发现master是可以执行成功的。而slave拿到这个SQL后，也会照常执行，不报任何异常，只是更新操作不影响行数而已。并且你执行命令show slave status，查看输出，你会发现没有异常。但是，如果你是row模式，由于这行根本不存在，是会报1062错误的。

常识二:怎查看binlog
binlog本身是一类二进制文件。二进制文件更省空间，写入速度更快，是无法直接打开来查看的。

因此mysql提供了命令mysqlbinlog进行查看。
一般的statement格式的二进制文件，用下面命令就可以。
```
mysqlbinlog mysql-bin.000001 
```
如果是row格式，加上-v或者-vv参数就行，如
```
mysqlbinlog -vv mysql-bin.000001 
```

常识三:怎么删binlog  
删binlog的方法很多，有三种是常见的  
(1) 使用reset master,该命令将会删除所有日志，并让日志文件重新从000001开始。  
(2) 使用命令  
```
PURGE { BINARY | MASTER } LOGS { TO 'log_name' | BEFORE datetime_expr }
```
例如
```
purge master logs to "binlog_name.00000X"
``` 
将会清空00000X之前的所有日志文件.
(3) 使用expire_logs_days=N选项指定过了多少天日志自动过期清空。

常识四:binlog常见参数  
常见参数，列举如下，有个印象就好。
![](/upload_imgs/2018111509335894419.png)
