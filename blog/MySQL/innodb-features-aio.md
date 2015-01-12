#InnoDB特性之AIO

##优势 

- 可以在发出一个IO请求后立即发出下一个IO请求。 

- 可以进行IO Merge的操作。可以将多个IO合并成一个IO。例如需要访问的页(space,page_no)为(8,6),(8,7),(8,8),由于每个页的大小为16KB，AIO会判断到三个页连续的，则会把3个AIO操作合并成一个。


##实现方式

- 1.1.x之前是innodb模拟的AIO的方式

- \>=1.1.x开始使用系统的AIO，native AIO的方式。因为需要libaio库的支持。

- 支持AIO的平台有Windows和linux，而mac osx不支持。

##配置方式

```
innodb_use_native_aio=on #默认值为on
```

##应用的范围
InnoDB中，read ahead的方式读取操作都是通过AIO完成，脏页的刷新操作全部是由AIO完成的。