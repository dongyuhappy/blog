#InnoDB逻辑存储结构

##主键
在InnoDB引擎的表中，每张表都有一个主键(primary key)，如果没有显式的定义主键，那么InnoDB将会按照下面的方式选择或者创建主键：

1. 判断表是否有非空得唯一索引，如果有则用该表定义的第一个非空唯一索引字段作为主键。

2. 如果没有，则自动创建一个大小为6个字节的指针作为索引。

也就是说无论如何InnoDB的表一定会有一个主键。


##InnoDB逻辑存储结构

InnoDB表的所有数据都存放在tablespace里面，tablespace又由段(segment)，区(extent),页(page),行（row）组成。



##TableSpace(表空间)

tablespace是InnoDB表逻辑存储结构的最高层，所有的数据都放在tablespace里面。如果```innodb_file_per_table```设置为on的话，tablespace将会为每张InnoDB表建立一个tablespace文件，每张表的独立数据会被存放到这个单独的tablespace里面，共享的数据还是会被存放在共享的tablespace文件里面。


每张表的tablespace里面主要存放的数据有：

1. 数据
2. 索引
3. 插入缓冲bitmap页

共享的tablespace里面主要包括：

1. undo
2. 插入缓冲索引页
3. 系统事物信息
4. 二次写缓冲（double write buffer）。 



根据上面的表述可知，即使是开始了innnodb_file_per_table，共享的tablespace文件的大小还是会增长。

##段(segment)


tablespace由段组成，主要的段包括：数据段，索引段，回滚段。




##区(extent)

区是由连续的页组成的空间，在任何情况下页的大小都是1m。为了保证区中页的连续性，InnoDB一次从磁盘申请4~5个区。默认情况下page的大小为16kb，也就是说一个区中有64个连续的页。

从InnoDB1.x开始引擎压缩页，新增参数innodb_page_size参数来设置默认页的大小为4k或者8k。


如果用户开始了innodb_file_per_table后，创建表的默认大小为96kb。区中是64个连续的页组成的，创建表的大小至少是1MB才对啊？其实这是因为每个段的开始时，先用32个页大小的碎片页来存放数据的，在使用完这些页后才是64个连续页的申请，这样做的目的是对于一些小表或者undo类的段，可以在开始的时候申请较少的空间。




##页（page）

页是磁盘管理的最小单位，默认是16kb，可以通过innodb_page_size来设置页的默认大小，若设置完成所有的表中的页的大小都为innodb_page_size。

在InnoDB中，常见的页的类型有：

- 数据页（b-tree node）
- undo页（undo log page）
- 系统页(System page)
- 事物数据数据页(transaction system page)
- 插入缓冲位图页(insert buffer bitmap)
- 插入缓冲空闲表页(insert buffer free list)
- 未压缩二进制大对象页(uncompressed blob page)
- 压缩二进制大对象页(compressed blob page) 


##行(row)

InnoDB表的数据是按照行存放的，每页最多存放的row为7992.



