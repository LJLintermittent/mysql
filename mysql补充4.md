### mysql的内部临时表

union的语义是取出这两个子查询的并集，并集的意思就是两个子查询的结果相加，重复的只添加一行

对子查询的结果做union的时候，使用explain分析后会看到Using temporary，表示使用了内部临时表，整个语句的执行流程是这样的：

(select 1000 as f) union (select id from t1 order by id desc limit 2);

1.创建一个临时表，这个临时表只有一个整型字段f，并且f是主键

2.执行第一个子查询，并将结果存入临时表

3.执行第二个子查询，试图将结果插入临时表，如果这个值已经在临时表中存在了，那么就违反了union的唯一性约束，所以插入失败，然后继续执行

4.从临时表中取出数据，返回结果，并删除临时表

如果把union改为了union all，没有了去重的概念后，这样在执行的时候直接执行两个子查询，然后将结果进行返回，这个过程不会使用到临时表，所以explain中没有Using temporary

group by也会出现Using temporary，不论是内存临时表还是磁盘临时表，group by逻辑都需要构造一个带唯一索引的表，执行代价都是比较高的。

group by的逻辑语义是：统计不同的值出现的个数。如果每一行的结果都是无序的，那就需要有一个临时表，来记录并统计结果，那么扫描过程如果可以保证出现的数据是有序的，就不需要临时表了

1.如果对group by语句的结果没有排序需求，要在语句后面加order by null

2.尽量让group by过程用上表的索引，确认方法是explain结果里没有Using temporary 和 Using filesort

3.如果group by需要统计的数据量不大，尽量只使用内存临时表；也可以通过适当调大tmp_table_size参数，来避免用到磁盘临时表；

4.如果数据量实在太大，使用SQL_BIG_RESULT这个提示，来告诉优化器直接使用排序算法得到group by的结果

InnoDB和Memory引擎的数据组织方式是不同的：

InnoDB引擎把数据放在主键索引上，其他索引上保存的是主键id。这种方式，我们称之为**索引组织表**（Index Organizied Table）。

而Memory引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为**堆组织表**（Heap Organizied Table）。

