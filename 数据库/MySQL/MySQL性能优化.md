# MySQL性能优化

## 一、MySQL执行流程与架构

### 1、SQL的执行过程

1.通信协议（例）

- TCP
  - TCP/IP
  - UnixSocket
    - /var/lib/mysql下的mysql.sock(连接服务器时没有使用mysql -h去连接其他的服务器，只是在本机连接时使用)
- 四进四出
- 单工
- 通信方式：异步
  - MySQL客户端一般使用同步的方式去连接服务端，同步使用比较简单，异步需要在客户端编程难度较大，需要处理数据的交叉。
- 连接方式：长连接,长连接复用。一般放到一个连接池里，实现复用。（淘宝技术这十年）

```shell
--查看连接数
show global status like 'Thread%';
show global variables like 'wait_timeout';	--非交互式连接超时时间，如JDBC程序。
show global variables like 'interactive_timeout';	--交互式连接超时时间，如数据库连接工具。
show variables  like 'max_connections';	--允许连接的最大连接数。 default：151，max：100000

--修改参数
set (GLOABAL) max_connections=;
--动态修改（临时修改）
set autocommit = on;
(persist 8.0版本)：可以写到配置文件中

--永久修改
/etc/my.cnf(Linux)
my.ini(Windows)
--参数的两种级别
GLOABAL
SEESION

--允许最大的sql包
show global variables like 'max_allowed_packet';
```

2、SQL语句执行

```shell
--MySQL默认不对SQL语句进行缓存，因为MySQL对SQL语句的大小写比较敏感，易造成大量缓存。表的数据发生变化也会把缓存给释放掉。
show variables like 'query_cache%';
```

3、词法解析器(paser)：解析树

4、预处理：判断表名，权限解析等。

5、优化器(optimizer)：执行的步骤（CBO：Cost Based Optimizer基于成本最小的优化器）

6、执行计划(Execution Plans):

7、执行器

8、存储引擎(Storage Engine):每种表都有一种存储引擎，不同的存储引擎实现不一样，可以替换。

```
innoDB：事务、支持行级别的锁定、大幅度降低IO的效率，支持外键，MVCC
```

Server层、存储引擎层

3、更新的SQL语句

加载到内存缓冲区Buffer pool，Page 16K（局部性原理）

脏页：只在内存缓冲区中存在的数据，后台线程会自动将脏页写到磁盘（刷脏）。

```
show global status like 'innodb_buffer_pool%';
```

内存丢失咋办：redo log，大小固定，默认是48M，后面的内容会将前面的覆盖，只能做容灾恢复，不能作为数据恢复。（物理日志（在哪个数据页上做了啥修改），只在innodb上有）

写到日志比写到数据库文件快？这样的效率是不是低？redo log会比写磁盘快，因为使用了顺序写，减少了刷脏的频率。

undo log:为了实现数据库事务的原子性，逻辑日志（记录反向的日志）

Server层 binlog日志（逻辑日志）：记录DDL、DML，大小不固定，默认关闭。不能够依赖binlog，需要定期定时备份。

用于数据恢复和主从复制

## 二、深入剖析MySQL索引实现原理

```mysql
ALTER TABLE table_name ADD INDEX idx_name(field);
```

单列索引、联合索引（复合索引）

主键索引：特殊的唯一索引，不能为空。

唯一索引：不重复

全文索引（FULLTEXT KEY）：匹配内容，match(content),有最小字符的限制（默认为4），对非英文的词语匹配不是很好。（ES）

索引：

- 二分查找（有序数组）
- AVL树，左右子树的高度差不超过1（将数据节点设计为16K大小，会导致IO繁重---> B树）
- B树（多路平衡查找树）
- B+树