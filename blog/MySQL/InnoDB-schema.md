# InnoDB架构体系 
*本文所讨论的InnoDB的版本为1.0.x*

InnoDB所做的工作主要分为三大块：  

- 启动管理InnoDB的后台线程  

- 内存 

- 数据以及日志文件的落地

##后台线程

后台线程主要负责刷新缓冲池的数据,并且把已经修改的缓冲池刷新到磁盘文件里面保存.  


InnoDB存储引擎是多线程模型.因为后台有多个不同的后台线程,用来处理多个不同的任务,他们主要分为:

- Master Thread,主要负责将数据异步刷新到磁盘.  

- IO Thread,处理IO请求。

- Purge Thread，用来清理已经提交事务的undo log。

- Page Cleaner Thread ，刷新脏页。

### Master Thread

Master Thread具有最高线程的优先级别，在其内部有多个循环组成：

- 主循环(loop)  
- 后台循环(backgroup loop)  
- 刷新循环(flush loop)  
- 暂停循环(suspend loop)  

Master Thread会根据数据库运行的状态在这些循环中进行切换。

####主循环

主循环有两大部分的操作--每10一次的和每秒一次的。
用伪代码可以这样描述为:

<pre>
void main(){
	loop();
}



//主循环
void loop(){

	//每秒进行一次的操作
	for(int i = 0;i < 10;i++){
		sleep(1)
		//刷新重做日志
		if(last_one_second_io < 5){
			//最多进行5个插入缓冲的合并
		}
		if(脏页比例 > 设置的最大脏页比例){
			//刷新100个脏页到磁盘
		}
		if(没有用户活动){
			goto backgroupLoop()
		}
	}

	//每10秒执行一次的操作
	if(最近10秒的IO次数少于200){
		//刷新100个脏页
	}

	//合并5个插入缓冲
	//日志缓冲刷新到磁盘
	//删除undo页

	if(脏页比例>70%){
		//刷新100个脏页
	}else{
		//刷新10个
	}
	if(没有活动用户){
		goto backgroupLoop();
	}





}

void backgroupLoop(){
	//删除无用的undo页
	//合并20个插入缓冲

	if(有活动用户){
		goto loop();
	}else{
		goto flushLoop()
	}

}

void flushLoop(){
	//刷新100个脏页

	if(脏页比例 > 设置的最大脏页比例){
		goto flushLoop()
	}else{
		goto suspendLoop();
	}
}

void suspendLoop(){
	//等待事件
	if(有活动用户){
		goto loop();
	}

}







</pre>

每秒一次的操作有:

1. 重做日志刷新到磁盘。（一定）  
2. 合并插入缓冲（可能，当前一秒内IO次数少于5次。）  
3. 至多刷新100个InnoDB缓冲池中的脏页。（可能，脏页的数量大于innodb_max_dirty_pages_pct的配置）  
4. 当前没有活动用户，切换到background loop。（可能）  

每10秒一次的操作有：

1. 刷新100个脏页到磁盘（可能，判断过去10秒内的IO操作是否少于200次）
2. 合并至多5个插入缓冲。（总是）
3. 将日志缓冲文件刷新到磁盘。（总是）
4. 删除无用的undo页。（总是）
5. 刷新100个或者10个脏页到磁盘。（总是）


####backgroup loop

当前没有用户活动称为数据库空闲，这时会进入background loop。

执行的操作有：

1. 删除无用的undo页。
2. 合并20个插入缓冲。
3. 跳到主循环。
4. 不断刷新100个页，直到符合条件。（可能，跳到flush loop中完成）。

如果flush loop中也没有事情可做则进入suspend_loop，将Master Thread挂起，等待事件发生。如果启动了innodb引擎，却没有任何的innodb表，那么Master Thread线程将被挂起。




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


## 内存

innodb存储引擎由以下三个部分组成:   

- 缓冲池(buffer pool),用innodb_buffer_pool_size来设置

- 重做日志(redo log buffer),用innodb_log_buffer_size设置

- 额外内存池(additional memory pool) ,用innodb_additional_mem_pool_size设置。


###缓冲池  



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


####缓冲池中数据页的类型  

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
		
		//free列表
		private List<Page> free = new ArrayList<Page>();
		//...........
	}

```


**为了减少数据库内部资源竞争，提高数据库的整体性能，可以通过innodb_buffer_pool_instances来配置缓冲池的实例个数,默认为1个**


####InnoDB对缓冲池的管理 


数据库缓冲池是通过[LRU](http://baike.baidu.com/link?url=lo84x-2KIn2pqUKytKrHSUAUa-5KnQXh6Pp6BznOCU-5zFMs1b05DV7SUa1PW2GqN0grs2QWCXwbDDJ1SeVL_q)(Last Recent Used)算法来进行管理的,即使用最频繁的在最前端，使用最少的在末端。当缓冲池满的时候，会首先释放LRU列表中尾端的页。

缓冲池中的页的默认大小为16k，InnoDB的对LRU算法做了一些优化，这种优化主要体现在两个方面：  

- 可以使用```innodb_old_blocks_pct```控制新页加入到LRU列表的位置。  

- 使用 ```innodb_old_blocks_time```控制新页多久才会被加入到LRU列表的首端。单位为毫秒。


**新读取到的页，不会立马放到LRU列表的首部，二是放到LRU列表的midpoint位置**，midpoint位置可以使用```innodb_old_blocks_pct```控制，值为百分比的整数，例如37表示，距离列表尾端的37%处。midpoint之前的列表称为new，之后的称为old。new列表中的页为活跃的热点数据。


**换句话说，新读取到的会首先放到距离尾端的百分之```innodb_old_blcoks_pct```的位置，然后至少会在old列表部分停留```innodb_old_blocks_time```毫秒。**


下面总结下InnoDB对缓冲池管理的整个流程

1. 数据库启动，根据```innodb_buffer_pool_size```设置的大小分配内存。这时候LRU数据列表为空。

2. 有新的数据页需要缓存，首先检查Free列表是否有空闲页，如果有从Free列表删除该页，放入到LRU数据列表，如果没有，删除LRU列表末端的页。

3. 新的页被放入```innodb_old_blocks_pct```位置。经过```innodb_old_blocks_time```毫秒后，新页被放入LRU数据列表的new端。当页从LRU数据列表的old部分加入到new部分成为page made young。而因为```innodb_old_blocks_time```没有从LRU数据列表的old部分进入new部分，成为page not made young。

用javascript可以这样表示为  


<pre>
var dataList = list
var freeList = list
var newPage;
if(freeList.length > 0){
	//free列表可以申请到新页
	 newPage = freeList.getFreePage();
}else{
	//free列表已经申请不到页，从LRU列表old端溢出一页
	newPage = dataList.getLastPage();
	newPage.clear();
}
//newPage的赋值操作。

add(dataList,page,midpoint);//加入到LRU列表的midpoint处

//经过innodb_old_blcoks_time秒后，newPage进入到LRU列表的new端
</pre>





#### 查看innodb数据存储引擎的运行状态


```
show engine innodb status;
```


解释下上面命令执行输出的结果

- 文件IO信息
<pre>
I/O thread 0 state: waiting for i/o request (insert buffer thread)
I/O thread 1 state: waiting for i/o request (log thread)
-----------读线程------------
I/O thread 2 state: waiting for i/o request (read thread)
I/O thread 3 state: waiting for i/o request (read thread)
I/O thread 4 state: waiting for i/o request (read thread)
I/O thread 5 state: waiting for i/o request (read thread)
-----------读线程------------
I/O thread 6 state: waiting for i/o request (write thread)
I/O thread 7 state: waiting for i/o request (write thread)
I/O thread 8 state: waiting for i/o request (write thread)
I/O thread 9 state: waiting for i/o request (write thread)
Pending normal aio reads: 0 [0, 0, 0, 0] , aio writes: 0 [0, 0, 0, 0] ,
 ibuf aio reads: 0, log i/o's: 0, sync i/o's: 0
Pending flushes (fsync) log: 0; buffer pool: 0
278 OS file reads, 5 OS file writes, 5 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
</pre>


- 内存池相关信息

<pre>
//innodb_buffer_pool_size配置的大小
Total memory allocated 137363456; in additional pool allocated 0
Dictionary memory allocated 48554
Buffer pool size   8191 //分配缓冲池中总页数
Free buffers       7926 //Free列表中页的数量
Database pages     265 //LRU列表中的页的数量
Old database pages 0 //未修改页的数量
Modified db pages  0 //脏页数量
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0 //新页从old到new端的数量，新叶还未从old到new端的数量
0.00 youngs/s, 0.00 non-youngs/s
Pages read 265, created 0, written 1
0.00 reads/s, 0.00 creates/s, 0.00 writes/s

Buffer pool hit rate 915 / 1000, //这个参数很重要，表示缓存命中率，这个值通常不能小于95%
young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/sLRU len: 265, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
</pre>


####InnoDB的压缩页

1.0.x开始支持压缩页的功能，原本16K的页面会被压缩为8k,4k,2k,1k。对于非16K页面的LRU列表用unzip_LRU列表进行管理，所以 总页面 = LRU列表页长度 + unzip_LRU列表长度


####脏页 

在LRU列表中的也被修改后，称为脏页。缓冲池中的页与磁盘上的页不一致。InnoDB通过checkpoint机制将脏页刷新到磁盘。脏页既存在于LRU列表也会存在Flush列表。


###重做日志 
InnoDB首先将重做日志放到这个缓冲区，然后再按照一定频率刷新到磁盘上面，重做日志缓冲区不需要设置很大，一般情况下每秒都会刷新到磁盘上的日志文件。

下面的三种操作都会在日志缓冲区中的数据刷新到磁盘上的日志文件里面： 

- Master Thread每1秒都会把重做日志刷新到磁盘。

- 每次的事物提交

- 当重做日志缓冲池空间小于1/2。


### 额外的内存池

在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域的内存不够，会从缓冲中申请。



##数据以及日志文件的落地

###CheckPoint技术

为了缓解CPU速度和磁盘速度之间的鸿沟，页的操作都首先在内存里面进行。即内存中的数据要比磁盘上面的数据新。数据库需要将内存中的新版本数据刷新到磁盘，这个刷新操作不是每次都进行。为了避免发生数据丢失的情况，当前事物数据库系统普遍都采用了 Write Ahead Log策略。**即事物提交时，先写重做日志(redolog)，然后再修改页。**对于数据库宕机导致数据丢失时，都是通过重做日志来来完成数据恢复的。


CheckPoint主要解决下面三个问题： 

- 缩短数据库的恢复时间

- 缓冲池不够用的时候刷新脏页到磁盘

- 重做日志不可用时，刷新脏页。

有了checkPoint技术，那么在数据库在恢复数据的时候就不需重做所有的redolog，只需要重做checkPoint之后的redolog进行恢复即可。

对于InnoDB而言，它是通过LSN（long sequence Number）来标记版本的。每个页都有LSN，重做日志也有LSN，CheckPoint也有LSN。

checkPoint所做的事情就是把缓冲池中的数据刷新到磁盘，不同之处在于每次刷新多少页，每次从哪里取脏页，以及什么时候触发CheckPoint，在InonoDB里面分为两种CheckPoint：

- Sharp CheckPoint，数据关闭时把所有的脏页都刷新到磁盘，这是默认方式，配置参数为```innodb_fast_shutdown=1```

- Fuzzy CheckPoint，只刷新一部分脏页到磁盘。在InnoDB内部使用的这种CheckPoint。

可能出现Fuzzy CheckPoint的几种情况：

- Master Thread CheckPoint，每秒或者10秒的速度刷新，这个过程是异步的。

- FLUSH_LRU_LIST CheckPoint，当LRU列表的空闲页数少于n个时，会移除LRU尾端的页，那么这个时候会进行checkPoint。可以通过```innodb_lru_scan_depth```来控制LRU列表中空闲页的数量。1.2.x之前这个检查操作在用户查询线程中进行，之后在page cleaner线程中进行。

- Async/Sync Flush CheckPoint。重做日志文件 不可用的情况下才进行。

- Dirty Page too much CheckPoint，缓冲池中的脏页太多，可以通过```innodb_max_dirty_pages_pct```来设置。这是一个百分百参数。





