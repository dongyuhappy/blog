# InnoDB行记录格式

InnoDB表记录是以行的形式存储的。这意味着页中保存的数据是一行行的数据。

可以使用下面的命令来查看表的行格式

```
show table status like "表名称"。
```
InnoDB行记录格式就是数据库页中行记录的组织规则。主要有两种：

- Compact，最常用的行记录的格式。

- Redundant，兼容之前的行记录格式而保留的。


## Compact行记录格式
下图显示了commpact行的存储方式

![image](../pic/Snip20150124_8.png)

### 非null字段变长字段长度列表
Compact行记录格式首部是一个列表，这个列表存放的是**非null**字段的**长度**列表，并且是按照数据库字段的**倒序**放置的。若列的长度小于255个字节用一个1字节表示其长度，否者用2个字节表示。由此可以看出变长字段的最大长度是:**Math.pow(2,2*8)=65535**。


### null标识位

该行数据是否有null值，有则用1表示，否者为0。该部分占一个字节

###记录头信息

该部分固定占5个字节，几个重要的头信息为：

1. deleted_flag,该行是否已经被删除

2. min_rec_flag，该记录是否被预先定义为最小记录

3. n_owned，拥有的记录数

4. heap_no,索引堆中该条记录的排序记录

5. record_type，记录类型。

6. next_record，页中下一条记录的相对位置。


### 每个列(数据库字段)数据

null不占该部分的任何空间。


每行数据除了用户自定义的字段列以外，还有两个隐藏的列:

1. 数据id列，占6个字节，
2. 回滚指针列，占7个字节，
3. 主键列，如果InnoDB表没有定义主键的话。

## 行数据溢出

InnoDB会将某些数据存储在真正的数据页之外。

记录可变字段长度的信息最多只会占2个字节(8位)，由此可知，varchar的最大长度为`Math.pow(2,2*8)=65535`,也就是65535。但是MySQL的varchar字段是没发存65535个字节的，创建下表：

```
create table mytest(a varchar(65535)) charset=latin1 engine=innodb;
```

会得到如下的错误信息：

```
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. You have to change some columns to TEXT or BLOBs
```
实际上InnoDB的vachar最大长度为65532个字节。

下面我们来创建一个varchar的长度为65532个字节的表
```
create table mytest(a varchar(65532)) charset=latin1 engine=innodb;
```
如果是SQL_MODE为1的话上面的创建表语句将会得到警告。a 字段的类型也会变成mediumtext。

注意：上面创建的表的charset是latin1,那如果改成utf8或者gbk呢，会怎样呢？

先设置了为严格模式。

```
set @@session.sql_mode='strict_trans_tables'; 
```

```
mysql> create table mytest(a varchar(65532)) charset=gbk engine=innodb;
ERROR 1074 (42000): Column length too big for column 'a' (max = 32767); use BLOB or TEXT instead
```
```
mysql> create table mytest(a varchar(65532)) charset=utf8 engine=innodb;   
ERROR 1074 (42000): Column length too big for column 'a' (max = 21845); use BLOB or TEXT instead
```

可以看到同样的varchar长度在不同的charset下面，支持的max长度不一样。这说明不同的charset的可以存储的数据长度不一样，varchar（n），这里的n指的是单字节。

MySQL官方手册定义的varchar的最大长度为65535指**所有varchar列的长度加起来最大为65535**，所以下面的创建表语句还是会报错

```
create table mytest(a varchar(65530), b varchar(3)) charset=latin1 engine=innodb;   
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. You have to change some columns to TEXT or BLOBs

```


**即使varchar最大能存65532字节的数据，我们知道存储引擎的页为16KB，也就是说每页最多能存16384个字节的数据。因为单页是存不下65532个字节数据的。这时就发生了数据溢出。**

普通情况下数据是存在B-tree的node中的，数据溢出后，数据存放的页类型为Uncompress BLOB页。

**如果一页能存放下两行数据，那么数据就不会被放到BLOB页，否则就会放到BLOB页。发生行溢出的行会保存数据的前768个字节在当前页**
。



## char的行结构存储

vachar是变长字符类型，其定义的长度为但是是**字节**，cahr是固定类型的字符类型，其定义的但是是**字符**。

可以使用CHAR_LENGHT查看字符的长度，LENGHT查看字节的长度。


如果表的charset为多字节的字符集，那么char在InnoDB的内部会被转化为varchar。
