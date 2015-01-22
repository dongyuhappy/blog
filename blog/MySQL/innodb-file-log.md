#MySQL文件之日志文件


MySQL常见的日志文件分为：

- 错误日志(error log)

- 慢查询日志(slow query log)

- 二进制日志(binlog)

- 查询日志(log)


##错误日志

错误日志文件多MySQL启动，运行，关闭过程进行了记录。该文件不仅记录了错误信息，页记录了警告信息或者正确的信息。通过下面的命令查看错误日志文件的位置：
<pre>
SHOW VARIABLES LIKE 'log_error';
</pre>

错误日志文件是定位问题的最好信息来源。


##慢查询日志

可以通过设置阀值，来定位可能存在问题的SQL语句。默认情况下，慢查询日志是关闭的。
可以通过下面的配置将其开启
<pre>
log_slow_queries=ON #开启慢查询日志
</pre>

慢查询阀值的设置参数为
<pre>
long_query_time=10 #默认为10秒，当sql语句的执行时间超过10s才会记录，等于10秒都不会记录
</pre>

查看慢查询日志的文件地址
<pre>
SHOW VARIABLES like "slow_query_log_file";
</pre>

没有使用索引查询的SQL语句也会被记录，默认这个功能是关闭的，打开方式为：
<pre>
SET @@global.log_queries_not_using_index=on;#默认是off
</pre>

限制未使用索引查询而进入慢查询日志文件的频率：
<pre>
SET @@gloabl.log_ throttle_ queries_ not_ using_ indexes=0 #每分钟记录的慢查询的条数，0表示没有限制
</pre>

慢查询日志的格式化查看
<pre>
mysqldumpslow 慢查询日志文件地址
</pre>


慢查询日志输出格式

默认是FILE方式，也可以改为TABLE的方式，通过下面查询设置,如果设置为TABLE方式，那么慢查询日志将会被记录下mysql.slow_log表里面

<pre>
SET @@global.log_output='TABLE';
</pre>

可以用下列语句来测试慢查询记录

<pre>
select sleep(10);
</pre>

##查询日志

记录了所有对数据库的请求信息的日志，配置参数为
<pre>
general_log = on 
</pre>

配置项与slow_log类似。


##二进制日志

记录了对MySQL所有的更改操作，不包括select和show命令的sql。即使是更改操作没有使数据发生变化，也有可能进入到binlog文件里面。

binlog文件的主要作用

- 恢复(recovery)

- 复制(replication) 

- 审计(audit)

binlog的相关配置，可以用下面的命令查看
<pre>
show variables like "%bin%";
</pre>


binlog默认是关闭的，使用下面的命令启动
<pre>
set @@global.lon_bin=on;
</pre>


下面这些参数影响着binlog的记录行为：

- max_binlog_size,

- binlog_cache_size

- sync_binlog

- binlog-do-db

- binlog-ignore-db

- log-slave-update

- binlog_format


###max_bin_log_size

指定单个文件的最大值，如果超过该值，则产生新的二进制文件。


###binlog_cache_size

当使用事物表的存储引擎时，所有未提交的二进制日志会被记录到一个缓存中去，等待该事物提交时直接将缓冲中得二进制日志写入二进制日志文件。该缓冲的大小就是binlog_cache_size决定的。默认为32k，binlog_cache_size是基于当前session，也就是每个session都会分配一个binlog_cache_size大小的缓冲。

可以通过查看 binlog_cache_use,binlog_cache_disk_use的状态来判断binlog_cache_size是否合适。

- binlog_cache_use，使用缓冲写二进制日志的次数

- binlog_cache_disk_size，使用临时文件写二进制日志的次数。


### sync_binlog
 
默认情况下，二进制日志并不是每次都是在写的时候同步到磁盘。当数据库所在操作系统发生宕机的时候，可能会有最后一部分数据没有写入到二进制文件中。

该sync_binlog的值表示每写缓冲多少次就同步到磁盘。sync_binlog=1表示采用同步写的方式写入到磁盘。该只默认为0。

sync_binlog=1会导致未提交的时候被写入到binlog中，若此时宕机，由于没有commit，该事物会被回滚掉，但是binlog已经记录，下次恢复的时候，不能回滚，可以通过innodb_support_xa=1来设置。

###binlog-do-db和binlog-ignore-db

表示需要记录和忽略记录binlog的数据库，默认为空。

###log-slave-update

如果当前数据库的角色为slave，默认它将不会从master复制binlog到自己的binlog里面


###bin_format

配置binlog文件的格式，默认为STATEMENT,可配置的值有STATEMENT，ROW,MIXED。

- STATEMENT，日志的逻辑SQL语句。事物的隔离级别为REPEATABLE READ级别。
- ROW,记录表的行更改情况，事物的隔离级别为 READ COMMITTED。binlog日志文件的会更大。
- MIXED，默认使用STATEMENT，下列情况会使用ROW：
	- 表的存储引擎为NDB。
	- 使用了UUID,USER,CURRENT_USER,FOUND_ROWS,ROW_COUNT等不确定函数
	- 使用了INSERT DELAY
	- 使用了用户自定义函数
	- 使用了临时表



###查看binlog

使用mysqlbinlog工具查看
```
mysqlbinlog --start-position=开始位置 binlog路径
```
