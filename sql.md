### 建表

~~~sql
学生表
CREATE TABLE `Student`(
    `s_id` VARCHAR(20),
    `s_name` VARCHAR(20) NOT NULL DEFAULT '',
    `s_birth` VARCHAR(20) NOT NULL DEFAULT '',
    `s_sex` VARCHAR(10) NOT NULL DEFAULT '',
    PRIMARY KEY(`s_id`)
);

课程表
CREATE TABLE `Course`(
    `c_id`  VARCHAR(20),
    `c_name` VARCHAR(20) NOT NULL DEFAULT '',
    `t_id` VARCHAR(20) NOT NULL,
    PRIMARY KEY(`c_id`)
);

教师表
CREATE TABLE `Teacher`(
    `t_id` VARCHAR(20),
    `t_name` VARCHAR(20) NOT NULL DEFAULT '',
    PRIMARY KEY(`t_id`)
);

成绩表
CREATE TABLE `Score`(
    `s_id` VARCHAR(20),
    `c_id`  VARCHAR(20),
    `s_score` INT(3),
    PRIMARY KEY(`s_id`,`c_id`)
);
~~~

### 插入测试数据

~~~sql
学生表：
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-05-20' , '男');
insert into Student values('04' , '李云' , '1990-08-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-03-01' , '女');
insert into Student values('07' , '郑竹' , '1989-07-01' , '女');
insert into Student values('08' , '王菊' , '1990-01-20' , '女');

课程表
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03');

教师表
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五');

成绩表
insert into Score values('01' , '01' , 80);
insert into Score values('01' , '02' , 90);
insert into Score values('01' , '03' , 99);
insert into Score values('02' , '01' , 70);
insert into Score values('02' , '02' , 60);
insert into Score values('02' , '03' , 80);
insert into Score values('03' , '01' , 80);
insert into Score values('03' , '02' , 80);
insert into Score values('03' , '03' , 80);
insert into Score values('04' , '01' , 50);
insert into Score values('04' , '02' , 30);
insert into Score values('04' , '03' , 20);
insert into Score values('05' , '01' , 76);
insert into Score values('05' , '02' , 87);
insert into Score values('06' , '01' , 31);
insert into Score values('06' , '03' , 34);
insert into Score values('07' , '02' , 89);
insert into Score values('07' , '03' , 98);

~~~

### SQL练习

1.

~~~sql
查询"01"课程比"02"课程成绩高的学生的信息及课程分数

select stu.* ,s1.s_score as 01_score,s2.s_score as 02_score 
from student stu 
join score s1 on stu.s_id=s1.s_id and s1.c_id='01'
left join score s2 on stu.s_id=s2.s_id and s2.c_id='02' 
or s2.c_id = NULL 
where s1.s_score>s2.s_score
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210723231409673.png)

~~~wiki
id:选择标识符
select_type:表示查询的类型。
table:输出结果集的表
partitions:匹配的分区
type:表示表的连接类型
possible_keys:表示查询时，可能使用的索引
key:表示实际使用的索引
key_len:索引字段的长度
ref:列与索引的比较
rows:扫描出的行数(估算的行数)
filtered:按表条件过滤的行百分比
Extra:执行情况的描述和说明
~~~

~~~wiki
重点解释：
*****type字段：

ALL、index、range、 ref、eq_ref、const、system、NULL（从左到右，性能从差到好）

对表访问方式，表示MySQL在表中找到所需行的方式，又称“访问类型”。

ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行

index: Full Index Scan，index与ALL区别为index类型只遍历索引树

range:只检索给定范围的行，使用一个索引来选择行

ref: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值，引用到的上一个表的列(上一个select结果)

eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件

const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量，system是const类型的特例，当查询的表只有一行的情况下，使用system

NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

*****key_len字段：

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）

不损失精确性的情况下，长度越短越好 


*****ref字段：
列与索引的比较，表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值


*****Extra字段：

Using where:不用读取表中所有信息，仅通过索引就可以获取所需数据，这发生在对表的全部的请求列都是同一个索引的部分的时候，表示mysql服务器将在存储引擎检索行后再进行过滤

Using temporary：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询，常见 group by ; order by

Using filesort：当Query中包含 order by 操作，而且无法利用索引完成的排序操作称为“文件排序”

Using join buffer：该值强调了在获取连接条件时没有使用索引，并且需要连接缓冲区来存储中间结果。如果出现了这个值，那应该注意，根据查询的具体情况可能需要添加索引来改进能。

Impossible where：这个值强调了where语句会导致没有符合条件的行（通过收集统计信息不可能存在结果）。

Select tables optimized away：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行

No tables used：Query语句中使用from dual 或不含任何from子句


*****注意:
EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划。
~~~

2.

~~~sql
查询"01"课程比"02"课程成绩低的学生的信息及课程分数

SELECT s.*,sc1.s_score as 01_score,sc2.s_score as 02_score from 
student s left join score sc1 on s.s_id = sc1.s_id and sc1.c_id = '01' or sc1.c_id = null
join score sc2 on s.s_id = sc2.s_id and sc2.c_id = '02' WHERE sc1.s_score < sc2.s_scores
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210803012505169.png)

~~~wiki
对表score的s_score字段加了索引
create index idx_s_score on score (s_score);
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210803013027295.png)

3.

~~~sql
查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩

SELECT stu.s_id,stu.s_name,ROUND(AVG(sc.s_score),2) as avg_score from 
student stu 
join score sc on stu.s_id = sc.s_id
GROUP BY stu.s_id,stu.s_name HAVING ROUND(AVG(sc.s_score),2) >= 60 ORDER BY avg_score DESC;
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210803195137121.png)

~~~wiki
ROUND(x) 求x的四舍五入的结果
ROUND(x,y) 求x的四舍五入结果，结果精度保留y位小数
本题的HAVING不能用WHERE来代替，因为：
1、where 后不能跟聚合函数，因为where执行顺序大于聚合函数。
2、where 子句的作用是在对查询结果进行分组前，将不符合where条件的行去掉，即在分组之前过滤数据，条件中不能包含聚组函数，使用where条件显示特定的行。
3、having 子句的作用是筛选满足条件的组，即在分组之后过滤数据，条件中经常包含聚组函数，使用having 条件显示特定的组，也可以使用多个分组标准进行分组。
~~~

4.

~~~sql
查询平均成绩小于60分的同学的学生编号和学生姓名和平均成绩

SELECT stu.s_id,stu.s_name,ROUND(AVG(sc.s_score),2) as avg_score from student stu
left JOIN score sc on stu.s_id = sc.s_id GROUP BY stu.s_id,stu.s_name HAVING ROUND(AVG(sc.s_score),2) < 60
UNION 
SELECT stu.s_id,stu.s_name,0 as avg_score from student stu 
WHERE stu.s_id not in (SELECT DISTINCT s_id as 学生表里有的学生编号 FROM score)
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210804194005329.png)

~~~wiki
SELECT DISTINCT ：查找不重复的值
~~~

5.

~~~sql
查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩

SELECT stu.s_id,stu.s_name,count(sc.c_id) as sum_course,sum(sc.s_score) as all_score_sum from student stu
left JOIN score sc on stu.s_id = sc.s_id GROUP BY stu.s_id,stu.s_name;

给student表name字段建立索引
create index idx_s_name on student(s_name);
删除索引
drop index idx_s_name on student;
~~~

1.无索引测试

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210805170428039.png)

2.有索引测试

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210805170540348.png)

~~~wiki
 show profile for query 2;  
 获取指定sql语句的执行开销
~~~

6.

~~~sql
查询"李"姓老师的数量

explain select COUNT(t_id) as 姓李的老师的数量 from teacher WHERE t_name LIKE '李%';
~~~

~~~wiki
当SQL语句改为如下写法时：

SELECT COUNT(t_id) as 姓李的老师的数量 from teacher where t_name  like '%李%';
或者
SELECT COUNT(t_id) as 姓李的老师的数量 from teacher where t_name  like '%李';

如果t_name字段加了索引，explain后发现依然使用了索引

“用索引” 和 “用索引快速定位记录”是有区别的。“用索引”有一种用法是 “顺序扫描索引”。
Like ‘y’ 或 ‘y%’ 可以使用索引，并且快速定位记录。like ‘%y’ 或 ‘%y%’，只是在二级索引树上遍历查找记录，并不能快速定位（扫描了整棵索引树）

index 类型表示”和全表扫描一样。只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210806163334582.png)

7.

~~~sql
查询学过"张三"老师授课的同学的信息

SELECT stu.* from student stu
JOIN score sc on stu.s_id = sc.s_id WHERE sc.c_id in (
	SELECT c_id from course where t_id = (
		SELECT t_id from teacher WHERE t_name = '张三'
	)
) ORDER BY stu.s_id ASC

create  INDEX idx_course_t_id on course(t_id)
建立完索引后，将ALL优化到了ref
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210807153333154.png)

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210807153352096.png)

8.

~~~sql
查询没学过"张三"老师授课的同学的信息

SELECT stu.* from student stu 
WHERE stu.s_id not in (
	SELECT stu2.s_id FROM student stu2 join score sc2 on stu2.s_id = sc2.s_id WHERE sc2.c_id in (
		SELECT c_id  from course WHERE t_id = (
			SELECT t_id from teacher WHERE t_name = '张三'
		)
	)
)
~~~

![image](https://cdn.jsdelivr.net/gh/chen-xing/figure_bed_02/cdn/20210808121025348.png)

9.

~~~sql
9.查询学过编号为"01"并且也学过编号为"02"的课程的同学的信息

SELECT stu.* FROM student stu,score sc1,score sc2
where stu.s_id = sc1.s_id and stu.s_id = sc2.s_id and sc1.c_id = '01' and sc2.c_id = '02'
~~~

10.

~~~sql
10. 查询学过编号为"01"但是没有学过编号为"02"的课程的同学的信息

SELECT stu.* from student stu 
WHERE stu.s_id in (SELECT s_id from score WHERE c_id = '01')
and stu.s_id not in (SELECT s_id from score WHERE c_id = '02')
~~~

11.

~~~sql
11. 查询没有学全所有课程的同学的信息

SELECT stu.* from student stu where s_id in (
	SELECT s_id from score WHERE s_id not in (
		SELECT a.s_id FROM score a 
		JOIN	score b on a.s_id = b.s_id and b.c_id = '02'
	  JOIN score c on a.s_id = c.s_id and c.c_id = '03'
	  WHERE a.c_id = '01'
	)
)
~~~

12.

~~~sql
查询至少有一门课与学号为"01"的同学所学相同的同学的信息

	SELECT stu.* from student stu WHERE s_id in(
		SELECT DISTINCT sc.s_id from score sc WHERE sc.c_id in(
			SELECT c_id from score  WHERE s_id = '01'
	)
)
~~~

