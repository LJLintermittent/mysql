## ID列

这一列总是包含一个编号，标识select所属的行。如果在语句当中没有子查询或者联合查询，那么只会有唯一的SELECT。于是每一行在这个列都将显示一个1

MySQL将select查询分为简单查询和复杂查询，复杂查询可以分为三大类：简单子查询，在from中的子查询和union查询。

注意：MySQL5.6允许解释非select查询。

## SELECT_TYPE列

这一列显示了对应行是简单还是复杂select，（如果是复杂select，那么是三种复杂查询的哪一种）

simple值意味着查询不包括子查询或联合查询，如果查询中有任何复杂的子部分，则最外层部分标记为primary

subquery：包含在select列表中的子查询的中的select（不在from语句中的）

derived：derived用来表示包含在from子句中的子查询中的select，MySQL会递归执行并将结果放到一个临时表，MySQL内部将其称为派生表，因为该临时表是从子查询中派生出来的。

union：在union中的第二个和随后的select被标记为union。

