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

- binlog-format


