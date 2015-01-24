#InnoDB存储引擎文件

除了MySQL表本身的一些文件外 ，每中存储引擎还有自己独立的文件，InnoDB独立的存储文件包括日志文件(redolog)和表空间文件(tablespace)


##表空间文件(tablespace)

InnoDB将数据存在tablespace文件里面。

在默认的配置下面会初始化一个名为ibdata1大小为10M的tablespace文件。所有InnoDB引擎的数据都会存放在这个文件里面。

但是，可以通过设置```innodb_file_per_table=on```，将给每个独立的InnoDB类型的表创建一个名为“表名.idb”的独立tablespace文件。   
**这些独立的tablespace文件仅存储了该表的数据，索引，插入缓冲BITMAP等信息，其余的信息还存放在ibdata1文件里面。**


##重做日志文件

在默认情况下，在InnoDB存储引擎的数据目录下面都会有两个名为ib_logfile0和ib_logfile1的文件，这两个就是 redo log file。

每个InnoDB至少有一个重做文件日志组，每组文件下面至少有两个重做日志文件，默认为ib_logfile0和ib_logfile1。

为了提高可靠性，可以设置多个镜像日志组，将文件放在不同的磁盘上面，提高日志文件的高可用性。在日志组中每个文件的大小一致，并以循环的方式写入。

###影响redo log file的配置参数

innodb_log_file_size，指定每个redo log file文件的大小。

innodb_log_files_in_group,设置每组重做日志文件的数量，默认为2
