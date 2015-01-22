#MySQL文件之socket文件和pid文件

##socket文件

MySQL在Unix下面本地连接采用UNIX域套接字的方式，这种方式需要一个socket文件，可以用下面的命令查看socket文件的路径。
```
SHOW VARIABLES LIKE "%socket%";
```
该文件一般在/tmp目录下面

##pid文件
MySQL实例启动的时候会把自己的进程id写入都一个文件中，这个文件就是pid文件。可以用下面的命令查看。
```
show variables like "%pid%";
```

