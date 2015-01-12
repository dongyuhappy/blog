#InnoDB特性之刷新邻接页

##原理

当刷新一个脏页时候，会检测该页所在区(extent)的所有页，如果有脏页，那么一起刷新。

##配置
    innodb_flush_neighbors #进行开关