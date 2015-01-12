#InnoDB文件之参数文件

数据库的配置文件，一般叫做my.cnf，可以用如下的命令来查找。
<pre>mysql --help|grep my.cnf</pre>


my.cnf配置项是一个键/值对。
通过下面的命令来查看配置
<pre>
show variables;
</pre>


mysql配置文件参数分为：

- 动态参数，可以在MySQL运行中进行修改。

- 静态参数，在MySQL运行中无法修改，可以理解为只读参数。



修改动态参数

在当前session有效
<pre>
set @@session.参数名称=值
</pre>

//在全局里面都有效
<pre>
set @@global.参数名称=值
</pre>


查看参数的值

全局参数的值
<pre>
SELECT @@global.参数名称;
</pre>


当前session的值
<pre>
SELECT @@session.参数名称 
</pre>