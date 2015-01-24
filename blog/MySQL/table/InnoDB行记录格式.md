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

![image](../pic/Snip20150124_8.png)