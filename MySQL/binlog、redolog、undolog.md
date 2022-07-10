# binlog, undo log, redo log
## binlog
binlog是MySQL记录数据库写操作的日志，持久化在磁盘中，记录的内容可以简单理解为SQL语句。binlog存在于server层，也就是说所有的引擎都支持binlog，binlog是通过追加的方式写入的，当文件写满时，MySQL会创建新的文件保存日志。

binlog的两种使用场景：
1. 主从复制：master端开启binlog，binlog发送到slave端，slave重放binlog来完成数据一致
2. 数据恢复：binlog可以用来恢复数据，比如误删除表等

binlog需要将日志持久化进入磁盘，这个操作是什么时候进行的呢？MySQL可以通过sync_binlog参数控制biglog的刷盘时机，取值为0-N。

* 0:系统自行判断何时写入磁盘
* 1:每次提交事务都将binlog写入磁盘
* N:每N个事务，才会被写入磁盘

## redo log
事务的四大特性中有一个持久性，最好的做法就是每次事务提交后，将数据写入磁盘，但是这样做会有很大的性能问题，具体来说redo log记录的是在某个数据页上做了什么修改，这样就可以满足顺序IO，提升性能。

