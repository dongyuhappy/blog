# InnoDB架构体系 
*本文所讨论的InnoDB的版本为1.0.x*

InnoDB所做的工作主要分为三大块：  
- 启动管理InnoDB的后台线程  
- 缓冲池的管理  
- 数据已经日志文件的落地

##后台线程

后台线程主要负责刷新缓冲池的数据,并且把已经修改的缓冲池刷新到磁盘文件里面保存.  


InnoDB存储引擎是多线程模型.因为后台有多个不同的后台线程,用来处理多个不同的任务,他们主要分为:

- Master Thread,主要负责将数据异步刷新到磁盘.  

- IO Thread,处理IO请求。

- Purge Thread，用来清理已经提交事务的undo log。

- Page Cleaner Thread ，刷新脏页。

### Master Thread
// TODO



### IO Thread  

IO线程主要分为四类:  


1. read,读线程，默认是4个线程,使用 ```innodb_read_io_threads```来调节数量   
	
2. write,写线程，通过```innodb_write_io_threads```来设置   
	
3. insert buffer，对于非唯一索引，辅助索引的修改操作并且时时更新索引的叶子页，而是在若干对同一页面的更新缓存起来，合并为一次性的更新操作。   

4. log   


###purge Thread

事务被提交后，所使用的undolog可能不再需要，使用PurgeThread来回收已经使用并分配undo也。

可以在mysql的配置文件中开启
```
innodb_purge_threads=1 #purge线程的数量

```

###page Cleaner Thread

刷新脏页的线程。


##缓冲池

InnoDB是基于磁盘存储的，并将其中的记录数据按照页的方式进行管理的，可以理解为InnoDB是基于磁盘的数据库系统。由于CPU的速度比磁盘的速度快很多，可以通过内存池的技术来提高数据库的性能。


缓冲池就是一块内存区域，通过内存的速度来弥补磁盘的速度对数据库性能的影响。  

读取页的流程可以用下面的伪代码来表示:

```
if(page in cache){  
	//读取的页在内存池中，从内存中读取  
	read cache[page]  
}else{  
	//读取的页不在内存池中，先把也加载到内存池(下次读取相同的页就可以直接从内存池中获取)，然后读取。 
	load page into cache  
	read page  
}

```  

对于页的修改操作为:   

1. 先修改内存池中页的数据，被修改的页称为**脏页**  

2. 然后在以一定的频率把修改页的数据刷新到磁盘保存。 不是每次页数据修改都会刷新到磁盘而是以checkpoint的机制来刷新到磁盘的。  


**综上所诉，缓冲池的大小，直接影响着数据库的整体性能。可以通过配置文件中的 innodb_buffer_pool_size来设置缓冲池的大小**


###缓冲池中数据页的类型  

- 数据页(data page)
  
- 索引页(index page)

- undo页

- 插入缓冲(insert buffer)

- 字适应hash索引页(adaptive hash index)

- InnoDB锁结构信息(lock info)

- 数据字典信息(data dictionary)



 缓冲池的基本模型可以使用下面的java代码来：  



```

	class Page{
		//innodb的数据记录是按照页的方式来进行管理的
	}
	
	 // 缓冲池
	 //缓冲池的大小由innodb_buffer_pool_size来决定
	 //缓冲池里面的list，使用LRU算法来管理
	 
	class Pool{
		//数据页
		private List<Page> dataPageList = new ArrayList<Page>();
		
		//索引页
		private List<Page>  indexPageList = new ArrayList<Page>();
		//...........
	}

```


**为了减少数据库内部资源竞争，提高数据库的整体性能，可以通过innodb_buffer_pool_instances来配置缓冲池的实例个数,默认为1个**


###InnoDB对缓冲池的管理 


数据库缓冲池是通过[LRU](http://baike.baidu.com/link?url=lo84x-2KIn2pqUKytKrHSUAUa-5KnQXh6Pp6BznOCU-5zFMs1b05DV7SUa1PW2GqN0grs2QWCXwbDDJ1SeVL_q)(Last Recent Used)算法来进行管理的,即使用最频繁的在最前端，使用最少的在末端。当缓冲池满的时候，会首先释放LRU列表中尾端的页