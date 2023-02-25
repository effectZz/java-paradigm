## 题目 01- 完成 ReadView 案例，解释为什么 RR 和 RC 隔离级别下看到查询结果不一致

 要求：

- 完成**案例 01- 读已提交 RC 隔离级别下的可见性分析**
- 完成**案例 02- 可重复读 RR 隔离级别下的可见性分析**
- 用通俗易懂的方式记录整个案例过程，可以画图与截图
- 做完案例给出结论，并对结论进行分析

回答范式：

1. 案例 01- 读已提交 RC 隔离级别下的可见性分析
   - 目标
   - 操作步骤
   - 实践过程
   - 结论
2. 案例 02- 可重复读 RR 隔离级别下的可见性分析
   - 目标
   - 操作步骤
   - 实践过程
   - 结论
3. 结论分析



**测试数据库使用为MySQL8**

RC：

| 时间\（线程） | 事务1        | 事务2                | 事务3              |
| ------------- | ------------ | -------------------- | ------------------ |
| T1            | 开启事务     | 开启事务             | 开启事务           |
| T2            | 更新name“关” | （查询结果不变）     | （查询结果不变）   |
| T3            |              | 更新name“赵”（阻塞） | （查询结果不变）   |
| T4            | 提交事务     |                      | 查询结果name为”关“ |
| T5            |              | 提交事务             | 查询结果name为”赵“ |
| T6            |              |                      | 提交事务           |

T2时刻事务1更新之后，事务 2 3查询的数值不变。

T3时事务2更新，事务2阻塞，事务3查询的数值不变。

T4时提交事务1，事务3结果变化为事务1提交的结果。

T5时提交事务2，事务3结果变化为事务2提交的结果。

T6时提交事务3，事务3结果变化为事务2提交的结果。



总结：**使用RC隔离级别的事务在每次查询开始时都会生成一个独立的ReadView。**



RC---SQL:

```sql
# 创建表
CREATE TABLE `tab_user` (
`id` int(11) NOT NULL,
`name` varchar(100) DEFAULT NULL,
`age` int(11) NOT NULL,
`address` varchar(255) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB;
# 新增数据
Insert into tab_user(id,name,age,address) values (1,'小明',18,'北京');
Insert into tab_user(id,name,age,address) values (2,'小李',22,'石家庄');
Insert into tab_user(id,name,age,address) values (3,'小黑',21,'阿富汗');

# 事务01
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
-- 设置数据库的隔离级别
set session transaction isolation level read committed;
SELECT * FROM tab_user;
BEGIN;
UPDATE tab_user SET name = '关' WHERE id = 1;
COMMIT;

# 事务02
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
-- 设置数据库的隔离级别
set session transaction isolation level read committed;
BEGIN;
# 更新了一些别的表的记录
UPDATE tab_user SET name = '赵' WHERE id = 1;
SELECT * FROM tab_use;
COMMIT;


# 事务03
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
-- 设置数据库的隔离级别
set session transaction isolation level read committed;
BEGIN;
SELECT * FROM tab_user; 
COMMIT;
```



RR：

| 时间\（线程） | 事务1                | 事务2                            | 事务3             |
| ------------- | -------------------- | -------------------------------- | ----------------- |
| T1            | 开启事务             | 开启事务                         | 开启事务          |
| T2            | 更新name“阿强”       | （查询结果不变）                 | （查询结果不变）  |
| T3            | 查询结果name为“阿强” | 删除id为2（查询结果缺id为2数据） | （查询结果不变）  |
| T4            | 查询结果name为“阿强” | 删除id为1（阻塞）                | （查询结果不变）  |
| T5            | 提交事务             | （id为1和2的数据全被删除）       | （查询结果不变）  |
| T6            | 查询结果name为“阿强” | 提交事务                         | （查询结果不变）  |
| T7            | （id仅剩3的数据）    | （id仅剩3的数据）                | 提交事务          |
| T8            | （id仅剩3的数据）    | （id仅剩3的数据）                | （id仅剩3的数据） |

T2时刻事务1更新之后，事务 2 3查询的数值不变。

T3时事务1结果为事务1修改的数据，事务2删除数据，事务2查询的数据为删除id2后的数据，事务3查询的数据不变。

T4时事务1结果为事务1修改的数据，事务2删除id为1时阻塞（行锁），事务3查询的数据不变。

T5时提交事务1，事务2结果变化为id 1，2均被删除，事务3结果不变。

T6时事务1结果为事务1修改的数据，提交事务2,提交以后事务1和事务2数据一致，事务3结果不变。

T7时事务1和2的查询结果一样，事务3结果不变。

T8时提交了事务3，最终查询结果为id 1，2均被删除。



总结：**对于使用RR 隔离级别的事务来说，只会在第一次**
**执行查询语句时生成一个ReadView ，之后的查询就不会重复生成了。**



RR---SQL:

```sql
# 创建表
CREATE TABLE `tab_user` (
`id` int(11) NOT NULL,
`name` varchar(100) DEFAULT NULL,
`age` int(11) NOT NULL,
`address` varchar(255) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB;
# 新增数据
Insert into tab_user(id,name,age,address) values (1,'小明',18,'北京');
Insert into tab_user(id,name,age,address) values (2,'小李',22,'石家庄');
Insert into tab_user(id,name,age,address) values (3,'小黑',21,'阿富汗');

# 事务01
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
SELECT * FROM tab_user;
BEGIN;
UPDATE tab_user SET name = '阿强' WHERE id = 1;
COMMIT;

# 事务02
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
BEGIN;
# 更新了一些别的表的记录
DELETE FROM tab_user WHERE id = 2;
SELECT * FROM tab_use;
DELETE FROM tab_user WHERE id = 1;
SELECT * FROM tab_user; 
COMMIT;


# 事务03
-- 查询事务隔离级别：
SELECT @@TRANSACTION_ISOLATION;
BEGIN;
SELECT * FROM tab_user; 
COMMIT;
```



## 题目 02- 什么是索引？

**要点：**

1.优点是什么？

​		大幅提高系统查询的速度

2.缺点是什么？

​	索引的存储需要占用磁盘空间，索引的创建和维护需要时间（增加数据的插入或删除时间）

 3.索引分类有哪些？特点是什么？

​	聚集索引（主键索引）查询效率十分快

​	辅助索引：需要回表再查询一次主键索引

​		辅助索引包含：普通索引、唯一索引、复合索引、全文索引

4.索引创建的原则是什么？

- ​	经常出现在Where子句中的字段或者经常查询的字段应建立索引
- ​	索引应该建在小字段上，对于大的文本字段甚至超长字段，不要建索引
- ​	组合索引要匹配最左原则

5.有哪些使用索引的注意事项？

​	索引数量不要创建太多

​	复合索引要按照字段在查询条件中出现的频度建立

6.如何知道 SQL 是否用到了索引？

​	在查询语句前面加上 explain ，查询结果字段上type表示查询的类型 ，possible_keys表示可能会用到的索引列，主要是key列显示的此时是用到的什么索引。

7.请你解释一下索引的原理是什么？「重点」

​	用于快速查找记录的一种数据结构。需要额外开辟空间和数据维护工作。可以加快检索速度，但是同时也会降低增删改操作速度，索引维护需要代价。

- 说清楚为什么要用 B+Tree

  - B+树在B树的基础上改造，B+树和B树最主要的区别在于：B树是非叶子节点和叶子节点都会存储数据，而B+树是只有叶子节点才会存储数据，非叶子节点只存储键值。B+树在同数据量下磁盘I/0次数更少，查询效率更稳定。

  

## 题目 03- 什么是 MVCC？

**要点：**

1. Redo 日志
2. ReadView
3. 如何判断可见性



MVCC：直译多版本并发控制，核心思想是读不加锁，读写不冲突。MVCC 只在 **Read Commited（读已提交） 和 Repeatable Read（可重读读） **两种隔离级别下工作。



undo log：

在InnoDB 中 MVCC 的实现方式为：每一行记录都有三个隐藏列：**DATA_TRX_ID**、**DATA_ROLL_PTR**、**DB_ROW_ID**。

DATA_TRX_ID：记录最近更新这条行记录的事务 ID

DATA_ROLL_PTR：表示指向该行回滚段的指针，InnoDB 通过这个指针找到之前版本的数据。

DB_ROW_ID：主键，如果有自定义主键，那么该值就是主键，如果没有，那么就会默认生成一个隐藏列作为主键。



redo log：

redo log 是InnoDB存储引擎独有的，它让MySQL拥有了崩溃恢复能力。比如 MySQL 实例挂了或宕机了，重启时，InnoDB存储引擎会使用redo log恢复数据，保证数据的持久性与完整性。



ReadView：

InnoDB在实现MVCC时用到的一致性读视图，用于支持读提交和可重复读隔离级别的实现，作用是执行期间判断版本链中的哪个版本是当前事务可见的。



可见性：

通过主键查找记录，根据记录版本里的DB_TRX_ID与Read View读视图进行判断

配合DB_ROLL_PTR回滚指针和undo log来找到当前事务可见的数据记录。