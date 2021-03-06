### mysql高可用架构

#### mysql的双1模式

~~~wiki
innodb_flush_log_at_trx_commit和sync_binlog 两个参数是控制MySQL 磁盘写入策略以及数据安全性的关键参数。

innodb_flush_log_at_trx_commit如果设置为0，表示log buffer将每秒一次写入到磁盘中，并且将log file page cache进行fsync，这个操作会同时进行。在该模式下，事务的提交不会主动触发写入磁盘的操作，如果设置为1，表示每次事务提交的时候都要进行log buffer往磁盘写入的操作，并且会fsync，如果设置为2，表示每次事务提交都会将buffer写入file，但是不flush，在这个模式下会每秒执行一次fsync

如果设置为0，在由于每秒一次的logbuffer写入，说明在这段时间数据都是在内存中的，所以这种模式既没办法满足mysql实例进程的crash-safe，也无法满足os的crash-safe，在最极端情况下可能会造成损失上一秒的所有数据

如果设置为1，每次提交都会提交到file cache，并且还会调用os的filecaceh flush操作，所以可以保证mysql的crash和os的crash

如果设置为2，每次提交都会写cache，这个写cache再次强调一遍，是把buffer中的内容写到操作系统的文件系统中的文件缓存中。每次提交都会写cache但是不会立即调用文件系统的flush操作，所以可以保证mysql的crash-safe，但是不能保证os的crash-safe

如果对系统的磁盘io能力很自信，同时需要满足数据的强一致性需求，不能容忍任何mysql crash和os crash，那么就是用1模式

sync_binlog 的默认值是0，像操作系统刷其他文件的机制一样，MySQL不会同步到磁盘中去而是依赖操作系统来刷新binary log。 当sync_binlog =N (N>0) MySQL 在每写 N次 二进制日志binary log时，会使用fdatasync()函数将它的写二进制日志binary log同步到磁盘中去

sync_binlog 为0时：当事务提交以后，os中的文件系统不会进行fsync调用，而让文件系统自行决定什么时候让文件缓存真正写到文件中，比如说cache满了的时候，所以如果os crash，会发生数据丢失

当为1时，事务只要提交，那么binlog会先从binlogbuffer写到os的binlog file的cache，然后会立即flush，可以保证数据的完全准备性，当然不可避免的是磁盘io的损失

所谓的双1模式就是每次事务提交不仅会做到redolog的最终落盘，还要保证binlog的最终落盘
~~~

当innodb_flush_log_at_trx_commit和sync_binlog 都为 1 时是最安全的，在mysql服务崩溃crash的时候，binlog只有可能丢失最多一个语句或者一个事务，但是双1模式会导致频繁的io操作，因此这个模式也是最慢的一种

正常情况下，只要主库执行更新生成所有的binlog，都可以传到备库并被正确的执行，备库就能达到跟主库一致的状态，这就是最终一致性，但是mysql要提供高可用的能力，只有最终一致性的是不够的。

双M结构的主备切换流程：

主备切换可能是一个运维动作，比如软件升级，主库所在机器定时下线等，也可能是被动操作，比如主库所在机器掉电

主备延迟：

1.主库A执行完一个事务，写入binlog，把这个时间作为T1

2.从库读取binlog，把这个binlog接收完毕这是时刻称为T2

3.从库B执行完这个事务，这个时刻称为T3

所谓的主备的延迟就是这个T3 - T1，备库执行show slave status可以通过参数列表查看当前系统的差值，至于两个库的系统时间不一致这个问题，其实不用担心，备库在连接入主库时，会从主库获取它的系统时间，需要注意的是，在网络正常的情况下，主库将日志打入从库这个时间延迟是很低的，主要时间会花费在从库接收和执行binlog这个时间，所以主备延迟最直接的表现是，备库消费中转日志relay log的速度，比主库生产binlog的速度要慢。

主备延迟的来源：

1.备库机器的性能低于主库，其实更新请求对于IOPS的压力，在主库和备库上是没有差别的，所以在做部署的时候，一般都会将备库设置为非双1模式

2.备库压力过大，主库主要提供写能力，从库提供查询能力，这种可以使用一主多从的架构，除了主备以外，再来几个从库，分担备库的读压力

3.大事务，主库上必须等待事务执行完才会写入binlog，再传给备库，典型的就是一次delete过多的数据，另一种典型的大事务场景是大表的DDL，这种如果真的有业务需求需要对线上的业务表进行在线DDL，可以使用gh-ost方案，也是一种online DDL

### mysql误删数据的处理

如果使用delete语句误删除了数据行，可以用flushback工具通过闪回将数据进行恢复，恢复的原理是：修改binlog日志内容，拿回原库进行重放，但是这种方式的前提是binlog的格式必须是row格式，并且binlog_row_image = full，binlog日志的前镜像只记录唯一识别列(唯一索引列、主键列)，后镜像只记录修改列，而binlog_row_image = full表示前镜像和后镜像都开启

具体恢复的过程是：

1.对于insert语句，对应的binlog是write_rows_event，把他改成delete_rows_event即可

2.对于delete语句，改成write即可

3.如果是update，binlog里面记录了数据行修改前和修改后的值，对调这两行位置即可

如果误删涉及多个事务，还需要将事务的顺序调过来执行，但是在恢复的过程中一般不会在主库上进行，会找一个库作为临时库，在这个临时库执行这些操作，然后通过确认后再恢复回主库

这是对于误删数据的事后处理方式，更重要的是做好事前预防。

1.可以将sql_safe_updates设置为on，这样如果忘记在delete后面跟where语句，或者where条件里面没有包含索引字段的话，这条语句执行会直接报错，但是如果业务上真的有删除一个小表中所有数据的需求时，可以使用delete from t where id >=0的方式来删除所有数据

2.代码上线前，要做好sql审计

使用delete删除的语句，可以使用flushback来借助binlog进行恢复，但是truncate和drop删除后无法通过flushback来进行恢复，注意这俩还是DDL，也就是执行会隐式提交事务，对于truncate和drop，即使binlog是row格式，执行这俩命令binlog还是statement格式，statement格式格式的binlog只记录了这俩的sql语句，对于恢复没有任何帮助

这种情况下，就需要使用全量备份和增量日志来恢复数据，这个方案要求线上有定期的全量备份，并且实时备份增量日志binlog

在这两个条件都具备的情况下，假如有人中午12点误删了一个库，恢复数据的流程如下：

1. 取最近一次全量备份，假设这个库是一天一备，上次备份是当天0点；
2. 用备份恢复出一个临时库；
3. 从日志备份里面，取出凌晨0点之后的日志；
4. 把这些日志，除了误删除数据的语句外，全部应用到临时库。

关于全量备份和增量日志的恢复方案，需要注意以下两点：

1.为了加速数据恢复，如果这个临时库上有多个数据库，可以使用mysqlbinlog命令时，加上一个–database参数，用来指定误删表所在的库。这样，就避免了在恢复数据时还要应用其他库日志的情况，因为增量日志会记录所有库的日志信息，而我们只需要误删表所在的那个库的增量日志

2.注意不要带上误删的语句

预防误删库方案：

只给业务开发同学DML权限，而不给DDL权限，即使是DBA成员，日常也只能使用只读权限的账号，必要时才使用有更新权限的账号

指定删库的流程规范：

如果确实有业务需要删除整张表的所有数据，在删除表之前，可以先把表名改了，然后让业务跑一段时间，看有没有影响，相当于逻辑上这个表没了，看整个业务系统有没有影响，确定了以后再回来真正删除

至于rm删除mysql实例，其实对于高可用的mysql集群并且带有HA系统，最不怕的就是rm，只要不是一个一个把整个集群删了，只是删了一个节点的话，HA系统会开始工作，找出一个新的主库

永远记住，删除的操作预防的意义远大于处理

### 大数据量查询会不会把内存打满

Innodb内存的一个作用：是保存更新的结果，再配合redo log，就避免了随机写盘，内存的数据页是在buffer pool中管理的，在wal里buffer pool起到了加速更新的作用，而实际上buffer pool还有一个作用就是加速查询。

由于有WAL机制，当事务提交的时候，磁盘上的数据页是旧数据，buffer pool中的是新数据，如果这时候有一个查询来读这个数据页，可以直接从内存中拿到结果。buffer pool对查询的加速效果，依赖于一个重要的指标，即内存命中率，可以用过show engine innodb status查看到一个系统的内存命中率， 一个稳定的线上系统，它的内存命中率应该是99％以上

buffer pool的大小是可以设置的，一般建议设置为物理内存的0.6到0.8，如果buffer pool满了，而又要从磁盘中读入一个新的数据页，那肯定要淘汰一个旧的数据页，innodb对于内存管理使用LRUlist来管理，但是如果直接使用LRU对于buffer pool来做管理，思考这样一种场景，如果有时候需要做一个全表扫描，而这个表不是业务常用表，那么会污染LRUlist，将真正有用的数据淘汰了出去，那么会出现buffer pool 的内存命中率急剧下降，磁盘io陡增，sql响应变慢，所以在mysql中将LRUlist划分为了oldlist和younglist，靠近尾部八分之三处是oldlist，也就是加入了一个midpoint机制，如果新加入一个数据页不在LRUlist的，那么会直接加入到midpoint处，当然淘汰的还是old区的最后一个数据页，当然改进后的LRUlist还有一个逻辑：如果这个数据页在old区待得时间超过了一秒，就会被移动到链表头部，如果短于一秒，那么处于原位置不变。

另外再说一个问题，大数据量查询会不会把内存打满，其实是不会的，mysql采用边算边发的机制，因此对于数据量很大的查询结果集来说，不会在server端保存完整的结果集，所以如果客户端接收的能力较弱，会堵住mysql的查询过程，但是不会把内存打爆

### 到底可不可以用join查询

select * from t1 straight_join t2 on (t1.a=t2.a);

如果直接使用join语句，mysql优化器可能会选择t1或者t2作为驱动表，这样我们分析sql语句时比较困难，所以为了便于分析，改用straight_join。STRAIGHT_JOIN只适用于inner join，并不使用与left join，right join，因为left join，right join已经指定了表的执行顺序

稍微备注一下mysql驱动表的概念：mysql中指定了连接条件时，满足查询条件的的记录行数少的表的会作为驱动表，如果未指定查询条件，则扫描行数少的表为驱动表，mysql优化器就是粗暴的使用小表驱动大表的方式来决定执行顺序的

STRAIGHT_JOIN会强行按照我们的顺序，即左表驱动右表

select * from t1 straight_join t2 on (t1.a=t2.a);这条语句的执行流程:

1.从表1中读入一条数据行R

2.从数据行R中取出连接条件字段的值去表2查找

3.取出表2中满足条件的行，跟表1中取出的行组装，作为结果集

4.重复执行上面的三步，直到驱动表末尾

### index nested loop join

这个过程是先遍历表1，然后根据表1中取出来的每行数据的连接字段，去表2中查询满足条件的记录，在形式上有点类似于业务代码中嵌套查询，从过程中可以看出，对于表2也就是大表的查询，如果加上了索引，那么速度是非常快的，这种算法称为index nested loop join，简称NLJ，其实从名字也能看出来，就是嵌套循环查询

结论：

使用join语句，性能比强行拆分成多个单表执行sql语句的性能要好

如果使用join语句，需要让小表驱动大表

这个结论的前提是可以使用被驱动表的索引

如果被驱动表用不上索引，假设小表驱动表有100行数据，大表被驱动表有一千行数据，那么如果不加索引需要搜索100 * 1000为10万，这种时间复杂度为N*M，N,M分别为小表和大表的行数

如果被驱动表加了索引，那么它的时间复杂度是首先小表必须全表扫描，而被驱动表是索引树搜索，时间复杂度为logM，而由于这个键肯定是普通索引，那么需要回主键索引继续查找，再来一个logM，所以一次连接是2logM，那么这个过程需要重复小表的行数那么多次，所以时间复杂度为N+N×2×logM，从这个时间复杂度可以看出来，N对结果的影响更大，所以确实应该是小表驱动大表。

当然如果真的在join查询的时候没有在被驱动表的连接字段加索引，那么也不可能直接N*M这种时间复杂度去查，这种做法叫simple nested loop join，实际上会采用block nested loop join算法

### block nested loop join

当被驱动表上没有索引上时，执行流程如下：

1.把表1的数据读入join buffer。

2.扫描表2，把表2的每一行取出来，跟join buffer中的数据做对比，满足join条件的，作为结果集的一部分返回

对于扫描表的时间复杂度是M+N，但是join buffer在内存中是以无序数组组织（mysql不支持hash join的诟病来源就是在这，补充：在mysql8.0.18以后支持了hash join）的，因此对于t2中的每一行（共M行），总共都要进行N此判断，因此从时间复杂复杂度上来说，都是一样，只不过对于block算法，由于是在内存中做判断，速度上会快很多，但是有个问题，join buffer是在线程内存中的，如果join buffer满了怎么办，一次放不下那么多的数据，其实这种情况也很常见，对于大表，join buffer是通过分块读取的，每次都读满，然后让表2的每一行都来判断，然后加入结果集，然后清空join buffer，重复这个步骤，这就是名字block的由来

回到开头的问题，该不该用join？

1.如果使用的是index nested loop join，join语句其实是可以使用的

2.不能使用被驱动表的索引，只能使用block nested loop join 这种语句尽量不要使用

3.在使用join的时候，一定记住一个原则，小表驱动大表，这个是通过时间复杂度推理得出来的结论

关于join buffer 分块后的时间复杂度计算：

我们再来看下，在这种情况下驱动表的选择问题。

假设，驱动表的数据行数是N，需要分K段才能完成算法流程，被驱动表的数据行数是M。

注意，这里的K不是常数，N越大K就会越大，因此把K表示为λ*N，显然λ的取值范围是(0,1)。

所以，在这个算法的执行过程中：

1. 扫描行数是 N+λ*N*M；
2. 内存判断 N*M次

显然内存判断不受驱动表的选择而影响的，考虑到扫描行数的影响，N越小越好，所以是驱动表是小表

对于大表来说，使用join，如果无法用到索引，即使有NLJ算法，在内存中N*M的比较次数（N，M：两个表参与join的行数的乘积）是一个大问题，非常消耗CPU资源

在总结join的优化方案之前，先搞清一个概念，MRR，mutil range read，这个优化的目的是尽量使用顺序读盘

因为大多数数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能

这就是MRR的优化思路：

思考这样一种场景，我们普通索引是对索引列进行排序，然后在索引树中根据排序好的索引列进行树中伪二分查找，最终找到所对应的主键id，然后拿着主键id去主键索引树查找到叶子节点，也就是完整的行记录，可以看出索引列的顺序拿出来的主键id不一定就是主键id自增的顺序，那么回表的过程是一行行搜索主键的过程，所以索引列的顺序与主键的顺序一点关系都没有，那么就是磁盘的随机读，而MRR的思路的是：

既然你拿出来的主键id是无序的，那我把拿出来的主键id先放在一个缓存中，进行主键id排序，这个缓存叫做read_rnd_buffer，排序好的主键id拿去主键索引树回表，这是一个顺序查询。

如果使用MRR，extra列中会出现Using MRR，表示的是用上了MRR优化

MRR能够提升性能的核心是在于，这条查询语句在索引a上做了一个范围查询，也就是说这是一个多值查询，可以通过足够多的主键id，并对主键id进行排序，然后再去主键索引查数据，体现出了顺序性的优势

BKA算法对NLJ算法的优化 batched key access

那么再来看index nlj的思路：从驱动表1取出一个一个数据，再到被驱动表2做join，也就是说对于2来说，每次都是匹配一个值，即使加了索引，看起来好像也很繁琐，而通常join操作都需要做很多次这样重复的事情，但是这种一个一个的传值，也用不上MRR，

解决思路：

一次从t1中多拿些值出来，放到内存中，这个临时内存就是上面用来优化没有索引情况下使用的block nested loop join算法中的 join buffer

在BNL算法中joinbuffer做为缓存驱动表的数据，但是在NLJ算法中没有使用，所以可以复用这个算法到bak算法。bak算法的启动的需要依赖MRR

关于BNL算法对于LRUlist的影响：

BNL算法是把驱动表加入join buffer，然后不停的扫描未加索引的被驱动表，然后在内存中进行比较，时间复杂度N ×M，思考这样一个问题：由于未加索引的大表的全表扫描，而且还是一个循环扫描，会将巨量的数据页加入到lrulist，最终这个大表的数据会到lru头，这会导致正常业务的数据页无法进入young区，进入young区的规则的是，需要这个数据页在一秒后再次被访问到，但是由于我们的join操作在不停的往old区加数据，导致业务数据在一秒内被淘汰出lrulist，造成内存命中率的急速下跌，对buffer pool的影响是持续性的，需要依靠后续的查询语句慢慢恢复命中率

一般情况下，我们在被驱动表上建索引，这时就可以直接从BNL转BKA算法了(不是转NLJ)

