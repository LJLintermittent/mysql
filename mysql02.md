# 《MySQL技术内幕：InnoDB存储引擎》

## Master Thread工作方式

InnoDB存储引擎的主要工作都是在一个单独的后台线程Master Thread中完成的。

## InnoDB 1.0.x版本之前的Master Thread

Master Thread具有最高的线程优先级别，其内部有多个循环（loop）组成：主循环，后台循环，刷新循环，暂停循环。Master Thread会根据数据库的运行状态在这些循环中进行切换。

Loop主循环，大多数操作是在这个循环中进行的，其中有两大部分的操作--每十秒的操作和每秒的操作

伪代码：

~~~c
void master_thread(){
    loop;
    for(int i = 0;i<10;i++){
        do thing per second;
        sleep 1 second if necessary;
    }
    do thing once per seconds;
    goto loop;
}
~~~

每一秒的操作包括：

* 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）
* 合并插入缓冲（可能）
* 至多刷新100个InnoDB的缓冲池中的脏页到磁盘（可能）
* 如果当前没有用户活动，则切换到backgroud loop (可能)

即使这个事务还没有提交，InnoDB存储引擎仍然会每秒将重做日志缓冲中的内容刷新到重做日志文件中，这也很好的解释了为什么再大的事务提交的时间也很短暂。

每十秒的操作包括：

* 刷新100个脏页到磁盘（可能的情况）
* 合并至多五个插入缓冲（总是）
* 将日志缓冲刷新到磁盘（总是）
* 删除无用的undo 页（总是）
* 刷新100个或者10个脏页到磁盘（总是）

## InnoDB 1.2.x版本之前的Master Thread

InnoDB存储引擎对于IO其实是有限制的，在缓冲池向磁盘刷新时都做了一定的硬编码，在固态硬盘出现的今天，这种规定很大程度上限制了InnoDB存储引擎对磁盘IO的性能，尤其是写入性能。

为此，谷歌工程师提出了这个问题后，进行了改进，InnoDB plugin提供了innodb_io_capacity参数，用来表示磁盘IO的吞吐量。

## InnoDB 1.2.x版本的Master Thread

在InnoDB 1.2.x版本再次对Master Thread进行了优化。由此可以看出Master Thread对于性能的重要性。

在这个版本中，Master Thread的伪代码如下：

~~~c
if InnoDB is idle
     srv_master_do_idle_tasks();
else 
     srv_master_do_active_tasks();
~~~

其中srv_master_do_idle_tasks()就是之前版本每10秒的操作，srv_master_do_active_tasks()就是每秒的操作。

同时对于刷新脏页的操作，从Master Thread线程分离到了一个独立的线程：Page Cleaner Thread，减轻了Master Thread的工作，进一步提高了系统的并发性。

## InnoDB关键特性

InnoDB存储引擎的关键特性包括：

* 插入缓冲
* 两次写
* 自适应哈希索引
* 异步IO
* 刷新邻接页

1.insert buffer

insert buffer可能是InnoDB存储引擎最令人激动的一个功能。

在InnoDB存储引擎中，主键是行的唯一标识符，通常来说应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的。因此，插入聚集索引一般是顺序的，不需要磁盘的随机读取，在这种情况下的插入操作，速度是非常快的。

但是如果主键是UUID这种类型的，那么它的插入和辅助索引一样，同样是随机的。即使主键是自增类型，但是插入的时候指定了值，而不是NULL值，那么同样可能导致插入并非连续的情况。

同样的，一张表中不可能只有一个聚集索引，可能还有多个非聚集的辅助索引。对于非聚集索引叶子节点的插入不再是顺序的了，这时候就需要离散的访问非聚集索引页，由于随机读取的存在而导致了插入操作性能下降。这是由于B+树的特性决定了非聚集索引插入的离散性。

InnoDB存储引擎设计了insert buffer ，对于非聚集索引的插入或者更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入，若不在，则先放入到一个insert buffer对象中，好似欺骗数据库这个非聚集的索引已经插入到了叶子节点，而实际并没有，只是存放在了另一个位置。然后再以一定的频率或情况进行insert buffer和辅助索引叶子节点的合并操作，这时通常能将多个插入合并到一个操作中，这样就大大提高了非聚集索引插入的性能。

insert buffer使用的要求有两个：

* 索引是辅助索引
* 索引不是唯一索引

辅助索引不能是唯一的，因为在插入缓冲时，数据库不会去查找索引页来判断插入的记录的唯一性，如果去查找，又会有离散读取的情况发生。从而导致insert buffer失去了意义。



