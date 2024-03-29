# 索引
## overview
提高查询效率的一种数据结构，像书的目录一样
## 索引类型
支持快速查询的数据结构有很多，比如有哈希表，搜索树，有序数组等。
* 哈希表：Hash表是一个以Key-Value形式存储的数据结构，可以在O(1)时间查询到指定Key的value值，所以，良好的Hash算法可以提升Hash表的性能，如果key的hash值重复，那就向后面拉一个链表。但是最大的问题是不支持范围查询，因为hash表示无序的。它适用于等值查询。
* 有序数组：有序数组最大的问题是删除和添加是一个很重的操作，所以适用于静态的场景，比如存储的是一个某一年杭州市的人口数量。
* 搜索树：搜索树查询某个值的时间复杂度为对数级别，当然，维护树的成本也是对数级别，但是二叉搜索树做索引，树过高，会增加磁盘的IO次数，所以使用了N叉树。

索引根据叶子结点存储的内容又可以分为：主键索引和普通索引
* 主键索引：叶子结点存储的是整行的数据，InnoDB中，主键索引也是聚簇索引
* 非主键索引：叶子结点存储的是主键的值，InnoDB中，非主键索引也是二级索引

二级索引搜索，需要比主键索引多扫描一棵树，因为二级索引查到主键的地址后，还要再去主键的索引树上去找整行的数据，这个过程也叫回表。
## 为什么使用B+树做索引
数据结构那么多，为什么选取了B+树来做索引呢？这个问题还是需要回到MySQL的底层，索引是用来提高查询效率的，那么MySQL需要被提升的地方在哪呢？答案是磁盘的IO，加入我们使用一个平衡树做索引的数据结构，那么查询的时间复杂度大约为对数级别，乍一听觉得很合理，但是并不是这样。因为索引是要落实到磁盘持久化的，数据量增大，树的高度会越来越高，每一次寻址就是一次磁盘的IO，磁盘IO就是MySQL最大的性能瓶颈，
## 索引维护
MySQL利用索引提高查询效率，那么一定要维护索引这种结构，因为一个数据的删除或插入，都有可能影响索引的存储结构，比如索引新插入了一个值1000，这个值位于已有的500和1500中间，那么就需要在这两个值中间进行插入，如果要插入进去的这个数据页满了，那么就要申请新的数据页，将一部分数据挪过去，这个过程称之为页分裂，那么自然有页合并。这也是有性能开销的。

问题的根源在于，我们无序的插入一个值，就可能有这样的事情发生。在建表的时候，有一个叫自增主键的东西，这是一个递增的操作，就不会有页分裂的性能开销，这是出于性能的考虑。另外自增主键如果是整形，只需要4个字节；长整型只需要8个字节，如果是一个长度24的UUID，那么就需要大约20多个字节。二级索引存储的是主键的值，叶子结点的主键越小，自然占用的空间就越小。

那么有没有用其他业务字段而不使用自增主键的情况呢？
* 这个业务字段就是主键索引
* 没有其他的二级索引

这就是Key-Value的情况，索引一般情况下，还是考虑用自增主键作为主键索引。
## 建设高性能索引
### 加了索引未生效
   1. 索引列是计算表达式的一部分
```sql
SELECT id FROM xxx WHERE age + 1 = 17
```
索引列是计算式的一部分，会导致全表扫描

2. 索引列是函数表达式的一部分

```sql
SELECT xxx FROM table WHERE data(created) < 7
```
在created字段上建索引，索引失效
3. 隐式类型转换

如果字段a类型是varchar，并且在字段a上建立索引

```sql
SELECT xxx FROM xxx WHERE a = 110
```
查询条件中a被写为int类型，导致MySQL发生隐式转换，a字段索引失效
4. MySQL主动选择不走索引

```sql
SELECT * FROM table WHERE age = 50;
```
看着是一个很普通的sql语句，age字段加了索引，但是什么情况会导致索引失效呢?答案就是MySQL主动选择了全表扫描，因为MySQL在选择索引时需要评估选择索引后回表带来的性能损耗，不走索引就不会回表，所以MySQL在自己评估后选择了不走索引，从而导致了全表扫描。可以使用force index(age)强制使用索引，也可以limit缩小查询的数量。有条件可以重新建立索引，引导MySQL选择合适的索引
### 覆盖索引
我们要查询的值，已经在索引树上，我们称为覆盖索引。覆盖索引可以减少树的搜索次数，提升查询性能，所以覆盖索引是一个常用的性能优化手段。
### 最左前缀原则
如果你要查的是所有名字第一个字是“张”的人，你的 SQL 语句的条件是"where name like ‘张 %’"。这时，你也能够用上这个索引，查找到第一个符合条件的记录是 ID3，然后向后遍历，直到不满足条件为止。

不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。这个最左前缀可以是联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符。

基于上面对最左前缀索引的说明，我们来讨论一个问题：在建立联合索引的时候，如何安排索引内的字段顺序。

这里我们的评估标准是，索引的复用能力。因为可以支持最左前缀，所以当已经有了 (a,b) 这个联合索引后，一般就不需要单独在 a 上建立索引了。因此，第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。
### 索引下推
```sql
select * from tuser where name like '张 %' and age=10 and ismale=1;
```
MySQL5.6之前，定位到以张开头后，开始回表查年龄和性别进行对比，5.6之后引入索引下推，在搜索到张开头后，对索引中包含的字段做判断，不满足的先过滤掉，减少回表的次数。
### 唯一索引和普通索引的选择
有一张市民信息表，存有身份证号、姓名等字段，我们使用自增主键。但是我们少不了一种情况，就是通过身份证号获取公民的姓名
```sql
select name from user_info where id_card='1'
```
那么我们有两种选择，给身份证号建唯一索引/普通索引，业务上已经保证了身份证号一定唯一，所以两种选择都可以。但是，从性能的角度出发，一定是选择普通索引。为什么？
#### 查询过程
如果是唯一索引，在找到id_card = 1这条记录后，会立刻停止检索。

对于普通索引来说，找到id_card = 1这条记录后，会继续向下搜索，直到第一个不满足id_card = 1的记录。

但是仅仅多查一条记录的开销，是非常非常小的。所以，选择两个的任意一个，对查询的影响是非常小的
#### 更新过程
* change buffer：更新一个数据时，如果数据页在内存中就直接更新，数据页没在内存中，在不影响一致性的前提下，会将更新的操作缓存在change buffer中，下次需要查询这个数据页的时候，将数据读入内存，然后执行change buffer中有关这个页的操作，这样就能保证数据的正确性。除了访问到需要的数据页时执行merge操作外，后台也会线程定期merge。

对于唯一索引来说，它不需要用到change buffer，因为唯一性的约束，所以要查询id_card = 1这个值是否已经存在，这个操作只能加载到内存操作，但是都已经加载到内存了，所以直接更新内存会更快。

对于普通索引来说，可以用到change buffer

所以在更新频繁的表里，普通索引的性能优于唯一索引，change buffer带来的性能提升是很明显的，并且数据读入内存是要占用buffer pool的，这种方式还能避免占用内存，提高内存利用率。
change buffer 用的是 buffer pool 里的内存，因此不能无限增大。

change buffer 的大
小，可以通过参数 innodb_change_buffer_max_size 来动态设置。这个参数设置为 50 的
时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

### change buffer使用场景
* 写多读少，对于写多读少的场景来说，页面在更新后立刻访问到的概率较小，此时使用change buffer效果较好，比如日志系统。
* 写少读多，不适合change buffer
如果更新之后立刻伴随查询，那么需要关闭change buffer，其他情况change buffer越大越好。
### change buffer和redo log
```sql
update table set xxx = 1, yyy = 2 where id = 4;
```
执行更新语句时：

* 更新的页在内存中，直接更新内存
* 不在内存中，记录change buffer，记录“要更新xxx行”这个信息
* 上述的两个操作记录进入redo log
* 事务结束，总共写一次磁盘，两次内存
```sql
select * from table where xxx = 1;
```
执行查询语句时：
* 查询的记录在内存中，直接返回，结果一定是正确的。虽然此时磁盘上的数据还是旧的，WAL之后读数据不需要刷新磁盘并读盘。
* 查询的记录不在内存中，将数据读入内存，应用change buffer中的内容，返回正确数据

所以两个的主要区别就是：change buffer提升的是读性能（减少随机读磁盘的IO消耗），redo log提升的是写性能（转为顺序写）
### MySQL选错索引
有些表可能有多个索引，那么MySQL在执行的时候选择哪个索引，这就是优化器的工作，但是，有可能一条执行很快的语句，因为MySQL选错了索引，导致了语句执行的很慢。

```sql
CREATE TABLE `t` (
 `id` int(11) NOT NULL,
 `a` int(11) DEFAULT NULL,
 `b` int(11) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `a` (`a`),
 KEY `b` (`b`)
) ENGINE=InnoDB；
```
往表 t 中插入 10 万行记录，取值按整数递增，即：(1,1,1)，(2,2,2)，(3,3,3) 直到 (100000,100000,100000)。
接下来，我们分析一条 SQL 语句：
```sql
select * from t where a between 10000 and 20000;
```
### 优化器的优化逻辑
优化器选择一个索引执行的目的，是选择最佳的索引，以最小的代价去执行这个语句，在数据库里，扫描行数、是否适用临时表、是否排序等因素综合判断。但是在上面的这个简单的SQL语句中，没有临时表和排序，那么一定是MySQL在判断行数时出了问题。
### 扫描行数的判断
一个索引上的不同值的个数，称之为“基数”。MySQL也是通过采样统计估算行数的，InnoDB会默认选择N个数据页，统计这些页面上的不同值，得到一个平均值，乘以索引的页面树，得到基数
## 

## 参考
《MySQL实战45讲》