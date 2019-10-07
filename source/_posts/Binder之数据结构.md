---
title: Binder之数据结构
date: 2016-06-05 16:49:18
tags:
 - Android
 - Binder
 - 数据结构
categories:
 - Android
---
去年10月份，深入学习了Android的Binder进程间通信机制，感觉收获很大，但时隔数月，忘记了很多。当时主要是结合网上几篇文章以及老罗的博客来学习的，结合源码，基本能看懂。但是网上的那几篇文章都是单刀直入，没有对相关的数据结构进行介绍。这里先把之前学习过程中整理的Binder相关数据结构记录下来，后续学习的时候，结合原理讲解和数据结构，才能更快的看懂这块逻辑。
## 关于Binder，几篇不错的文章
1. [Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)
2. [Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)
3. [Android Bander设计与实现 - 实现篇](http://www.cnblogs.com/albert1017/p/3849585.html)
4. [老罗的binder系列文章](http://blog.csdn.net/luoshengyang/article/details/6618363)

<!-- more -->

## Binder数据结构
首先介绍下Binder驱动内部使用的数据结构。
### Binder 实体在驱动中的表述`binder_node`
`struct binder_node`结构是在第一次传输Binder实体时，由驱动程序在内核中创建的，隶属于提供Binder实体的进程。即`flat_binder_object`的type为BINDER_TYPE_BINDER or BINDER_TYPE_WEAK_BINDER时。

binder_node结构的成员如下所示：
``` c++
struct binder_node {
int debug_id;
struct binder_work work;
union {
	struct rb_node rb_node;
	struct hlist_node dead_node;
	};
struct binder_proc *proc;
struct hlist_head refs;
int internal_strong_refs;
int local_weak_refs;
int local_strong_refs;
void __user *ptr;
void __user *cookie;
unsigned has_strong_ref:1;
unsigned pending_strong_ref:1;
unsigned has_weak_ref:1;
unsigned pending_weak_ref:1;
unsigned has_async_transaction:1;
unsigned accept_fds:1;
unsigned min_priority:8;
struct list_head async_todo;
};
```
相关字段的含义如下所示：
1. int debug_id
	>标识一个Binder实体对象的身份，用来帮助调试Binder驱动程序。

2. struct binder_work work;
	>当Binder实体对象的引用计数发生改变时，Binder驱动会请求相应的Service组件修改其引用计数。此时，就会把该引用计数修改操作封装成一个binder_work工作项（即：将Binder实体对象成员变量work的type属性值设置为BINDER_WORK_NODE），添加到相应进程的todo队列中去等待处理。

3.  一个联合体成员
	``` c++
	 union {
			struct rb_node rb_node;
			struct hlist_node dead_node;
	 };
 ```
	>每个进程都会维护一棵红黑树（**binder_proc.nodes**），以Binder实体对象在用户空间的地址，即本结构的ptr成员为索引，来存储该进程所有的Binder实体。这样驱动程序可以根据Binder实体在用户空间的地址，快速定位到其位于内核中的节点。`rb_node`用于将本节点链入到该红黑树中。当该BInder实体对象销毁时，需要从该红黑树中删除节点。
	
	>若一个Binder实体对象的宿主进程已经死亡了，那么这个Binder实体对象就会通过它的成员变量`dead_node`保存在一个全局的hash列表中。

4. struct binder_proc *proc;
	>本成员指向binder_node节点所属的宿主进程，即提供该节点的进程。

5. struct hlist_head refs;
	>本成员是队列头，所有指向该Binder实体对象的引用（`binder_ref`）都链接在该队列里。这些引用属于不同的进程，通过该队列可以遍历指向该节点的所有引用。

6. void __user * ptr;
	>指向用户空间Binder实体的指针，来自于`flat_binder_object`的binder成员。

7. void __user *cookie;
	>指向用户空间的附加指针，来自于`flat_binder_object`的cookie成员。

8. unsigned has_async_transaction；
	> 该成员表明该binder_node所属进程binder_proc的todo队列中有异步交互任务尚未处理完成。（驱动程序会将所有发送往服务端的数据包暂存在接收进程或线程开辟的todo队列里)。对于异步交互，驱动程序做了适当的限流：如果todo队列里有异步交互任务尚未处理完成，则该成员置1，这将导致新到的异步交互任务被添加到本结构的成员asynch_todo队列中，而不直接送到所属进程的todo队列里。目的是为同步交互让路，避免长时间阻塞发送端。

9. struct list_head async_todo
	> 异步交互等待队列，用于分流发往所属进程todo队列的异步交互任务，和has_async_transaction属性配合使用。

10. unsigned accept_fds 
	>表明Binder实体对象是否同意接收文件方式的Binder，来自flat_binder_object中flags成员的第8bit位，代码中用FLAT_BINDER_FLAG_ACCEPTS_FDS取得。由于接收文件Binder会为进程自动打开一个文件，占用有限的文件描述符，Binder实体可以设置该标志位拒绝这种行为。

11. int min_priority
	>设置处理Binder请求的线程的最低优先级。发送线程将数据提交给接收线程处理时，驱动程序会将发送线程的优先级也赋予接收线程，使得数据即使跨了进程也能以同样优先级得到处理。不过如果发送线程优先级过低，接收线程将以预设的最小值运行。该域的值来自于flat_binder_object中flags成员的第0到第7 bit位，代码中用FLAT_BINDER_FLAG_PRIORITY_MASK取得。

12. int internal_strong_refs;
	>用以实现强指针的计数器：产生一个指向本节点的强引用，该计数就会加1。

13. int local_weak_refs;
	>驱动程序为传输中的Binder设置的弱引用计数。如果一个Binder打包在数据包中，从一个进程发送到另一个进程，驱动会为该Binder增加引用计数，直到接收进程通过BC_FREE_BUFFER，通知驱动程序释放该数据包的数据为止。

14. int local_strong_refs;
	> 驱动程序为传输中的Binder设置的强引用计数，同上。

15. 剩余的成员用于控制驱动程序与Binder实体所在进程（Service组件），交互式修改引用计数。
	>unsigned has_strong_ref;
	unsigned pending_strong_ref;
	unsigned has_weak_ref;
	unsigned pending_weak_ref




### Binder引用在驱动中的表述`binder_ref`
和Binder实体一样，Binder引用也是驱动程序根据传输数据中的`flat_binder_object`创建的，隶属于获得该引用的进程，用struct binder_ref结构体表示:
``` c++
struct binder_ref {
/* Lookups needed: */
/*   node + proc => ref (transaction) */
/*   desc + proc => ref (transaction, inc/dec ref) */
/*   node => refs + procs (proc exit) */
int debug_id;
struct rb_node rb_node_desc;
struct rb_node rb_node_node;
struct hlist_node node_entry;
struct binder_proc *proc;
struct binder_node *node;
uint32_t desc;
int strong;
int weak;
struct binder_ref_death *death;
};
```
相关字段解释如下：

1. int debug_id;
	>标识一个Binder引用的身份，用来帮助调试Binder驱动程序。

2. struct binder_proc *proc;
	>指向Binder引用的宿主进程

3. struct binder_node *node;
	>本引用所指向的Binder实体对象（binder_node）

4. uint32_t desc;
	> 表示一个句柄值，指向一个Binder引用对象。在Client进程的用户空间中，一个Binder引用对象是使用一个句柄值来描述的，即desc属性的值。

5. int strong;
	>强引用计数，用来维护Binder引用对象的生命周期。

6. int weak;
	>弱引用计数，用来维护Binder引用对象的生命周期。

7. struct hlist_node node_entry;
	>该属性作为节点链入所指向的Binder实体对象binder_node中的refs链表中。

8. struct rb_node rb_node_desc;
	>宿主进程有一棵红黑树，宿主进程所有的Binder引用，以引用号（即本结构的desc域）为索引链接入该红黑树中（`binder_proc.refs_by_desc`）。本成员作为链接到该红黑树的一个节点。

9. struct rb_node rb_node_node;
	>宿主进程有一棵红黑树，宿主进程所有的Binder引用，以对应Binder实体在内核中的内存地址（即本结构的node域）为索引链接入该红黑树中（`binder_proc.refs_by_node`）。本成员作为链接到该红黑树的一个节点。

10. struct binder_ref_death *death;
	> 该成员变量指向一个Service组件的死亡接收通知。当Client进程通过BC_REQUEST_DEATH_NOTIFICATION命令向Binder驱动程序注册一个它所引用的Service组件的死亡接收通知时，Binder驱动程序就会创建一个`binder_ref_death`结构体，然后保存在对应的Binder引用对象的成员变量death中。

	>这样，当该Binder引用所引用的Service组件死亡时，就会通过death变量指向的`binder_ref_death`，向Client进程发送通知。

	>该域不为空，表明Client进程订阅了对应Service组件销毁的“噩耗”。

关于和Binder引用相关的红黑树，和Binder实体一样，每个宿主进程（`binder_proc`）使用红黑树存放所有正在使用的引用。不同的是Binder引用可以通过两个键值索引：

- 对应Binder实体（`binder_node`）在内核中的地址。**注意这里指的是驱动创建于内核中的binder_node结构的地址，而不是Binder实体在用户进程中的地址**。实体在内核中的地址是唯一的，用做索引不会产生二义性；但实体可能来自不同用户进程，而实体在不同用户进程中的地址可能重合，不能用来做索引。驱动利用该红黑树可以在一个进程中快速查找某个Binder实体所对应的Binder引用（一个实体在一个进程中只建立一个引用）。

- 引用号（句柄值）。引用号是驱动程序为引用分配的一个32位标识，在一个进程内是唯一的，而在不同进程中可能会有同样的值，这和进程的文件描述符很类似。引用号将返回给应用程序，可以看作Binder引用在用户进程中的句柄。除了0号引用在所有进程里都固定保留给ServiceManager，其它值由驱动程序动态分配。向Binder实体发送数据包时，应用程序将引用号填入binder_transaction_data结构的target.handle域中表明该数据包的目的Binder。驱动程序根据该引用号在红黑树中找到对应的binder_ref结构，进而通过其node域知道目标Binder实体所在的进程及其它相关信息，实现数据包的路由。

### 内核缓冲区节点binder_buffer
Binder驱动程序会为每个正在使用binder进程间通信的进程分配内核缓冲区，用来接收Client进程发送来的请求数据。内核缓冲区会被划分为若干个小块。binder_buffer就表示一个内核缓冲区块，其结构体如下所示：
``` c++
struct binder_buffer {
struct list_head entry; /* free and allocated entries by address */
struct rb_node rb_node; /* free entry by size or allocated entry  by address */
unsigned free:1;
unsigned allow_user_free:1;
unsigned async_transaction:1;
unsigned debug_id:29;
struct binder_transaction *transaction;
struct binder_node *target_node;
size_t data_size;
size_t offsets_size;
uint8_t data[0];
};
```
相关字段的含义如下所示：

1. struct list_head entry;
	>进程的内核缓冲区列表中的一个节点，头节点为`binder_proc.buffers`。
	
2. struct rb_node rb_node;
	>进程（binder_proc）使用两个红黑树来分别保存那些正在使用的内核缓冲区，以及空闲的内核缓冲区。如果一个内核缓冲区是空闲的，则它的成员变量free为1，那么该成员变量	`rb_node`就是空闲内核缓冲区红黑树中的一个节点（binder_proc.free_buffers）；否则，成员变量free为0，该成员变量	`rb_node`就是正在使用内核缓冲区红黑树中的一个节点（binder_proc.allocated_buffers）。

3. unsigned free;
	>和rb_node属性配合使用，1表示空闲缓冲区，0表示正在使用缓冲区。

4. unsigned allow_user_free
	>目标Service组件在处理完任务后，若发现传给他的内核缓冲区的成员变量allow_user_free的值为1，那么该Service组件就会请求Binder驱动程序释放该内核缓冲区。

5. unsigned async_transaction;
	>1表示与该内核缓冲区关联的是异步事务，否则为0，表示为同步事务。Binder驱动程序限制了分配给异步事务的内核缓冲区大小，目的是为了保证同步事务可以优先得到内核缓冲区，以便可以快速的处理同步事务。

6. unsigned debug_id;
	>标识一个内核缓冲区的身份，帮助调试Binder驱动程序。

7. struct binder_transaction * transaction;
	>表示哪一个事务`binder_transaction`正在使用该内核缓冲区。

8. struct binder_node * target_node;
	> 表示哪一个Binder实体在使用该内核缓冲区。

9. size_t data_size;
	>真正的数据缓冲区的大小。

10. size_t offsets_size;
	>在data指向的数据缓冲区的后面，有一个偏移数组，记录了数据缓冲区中每一个binder对象在数据缓冲区的位置。而偏移数组的大小就保存在该offsets_size成员中。
	
11. uint8_t data[0];
	>指向真正的数据缓冲区，用来保存通信数据。

### Binder进程在内核中表述（Binder_proc）
该结构体用来描述一个正在使用Binder进程间通信机制的进程。当一个进程调用函数`open`来打开设备文件`/dev/binder`时，Binder驱动程序就会为它创建一个	`binder_proc`结构体，并且将它保存在一个全局的hash列表中，而成员变量`proc_node`就是该hash列表中的一个节点，其结构体如下所示：
``` c++
struct binder_proc {
struct hlist_node proc_node;
struct rb_root threads;
struct rb_root nodes;
struct rb_root refs_by_desc;
struct rb_root refs_by_node;
int pid;
struct vm_area_struct *vma;
struct mm_struct *vma_vm_mm;
struct task_struct *tsk;
struct files_struct *files;
struct hlist_node deferred_work_node;
int deferred_work;
void *buffer;
ptrdiff_t user_buffer_offset;

struct list_head buffers;
struct rb_root free_buffers;
struct rb_root allocated_buffers;
size_t free_async_space;

struct page **pages;
size_t buffer_size;
uint32_t buffer_free;
struct list_head todo;
wait_queue_head_t wait;
struct binder_stats stats;
struct list_head delivered_death;
int max_threads;
int requested_threads;
int requested_threads_started;
int ready_threads;
long default_priority;
struct dentry *debugfs_entry;
};
```
相关字段的含义如下所示：

1. struct hlist_node proc_node;
	 >全局`binder_proc`列表中的一个子节点。

2. struct rb_root threads;
	>红黑树根节点，以线程ID为关键字来组织一个进程的Binder线程池，节点为`binder_thread.rb_node`。

3. struct rb_root nodes;
	>红黑树根节点，以`binder_node.ptr`即Binder实体在用户空间的地址为索引，来组织一个进程的所有Binder实体对象。节点为`binder_node.rb_node`。

4. struct rb_root refs_by_desc;
	>红黑树根节点，以`binder_ref.desc`即Binder引用的句柄值为索引，来组织一个进程的所有Binder引用对象。节点为`binder_ref.rb_node_desc`。

5. struct rb_root refs_by_node;
	>红黑树根节点，以`binder_ref.node`即 Binder实体（binder_node）在内核中的地址为索引，来组织一个进程的所有Binder引用对象。节点为`binder_ref.rb_node_node`。

6. int pid;
	>进程（组）ID

7. struct task_struct * tsk;
	>该进程的任务控制块

8. struct files_struct * files;
	>该进程的打开文件结构体数组。

9. struct hlist_node deferred_work_node和int deferred_work;
	>Binder驱动程序将所有的延迟执行的工作项保存在一个Hash列表中。如果一个进程有延迟执行的工作项，那么成员变量deferred_work_node就刚好是该Hash列表中的一个节点，并且使用成员变量deferred_work来描述该延迟工作项的具体类型。

10. int * buffer、struct vm_area_struct * vma、size_t buffer_size和ptrdiff_t user_buffer_offset
	>这几个成员变量用来管理内核缓冲区。

	>进程打开设备文件`/dev/binder`之后，还必须调用函数`mmap`将它映射到进程的用户地址空间来，实际上是请求Binder驱动程序为它分配一块内核缓冲区，以便可以用来在进程间传输有效负荷数据，即`binder_transaction_data.data.buffer`指向的有效负荷数据。

	>该内核缓冲区的大小保存在成员变量`buffer_size`中。这些内核缓冲区有两个地址，其中一个是内核缓冲区地址，另外一个是用户空间地址。内核空间地址是在Binder驱动程序内部使用的，保存在成员变量`buffer`中，而用户空间地址是在应用程序进程内部使用的，保存在成员变量`vma`中。这两个地址相差一个固定的值，保存在成员变量`user_buffer_offset`中。这样，给定一个用户空间地址或者一个内核空间地址，Binder驱动程序就可以计算出另外一个地址值。

11. struct list_head buffers、struct rb_root free_buffers、struct rb_node allocated_buffers、unit32_t buffer_free和size_t free_async_space;
	>这几个成员变量用来管理被分割成小块的内核缓冲区。
	
	>Binder驱动程序为了方便管理分配的内核缓冲区，将它分成若干个小块。这些小块的内核缓冲区就是使用前面介绍的`binder_buffer`结构体来描述，它们保存在一个列表中，按照地址值从小到大的顺序来排列，成员变量`buffers`指向的便是该列表的头部，该列表的的节点就是`binder_buffer.entry`。

	>`buffers`列表中的小块内核缓冲区有的是正在使用的，即已经分配了物理页面；有的是空闲的，即还没有分配物理页面，它们分别组织在两个红黑树中。其中，空闲缓冲区保存在成员变量`free_buffers`所描述的红黑树中(以binder_buffer的数据缓冲区大小为关键字)，而正在使用缓冲区则保存在成员变量`allocated_buffers`所描述的红黑树中（以binder_buffer的内核空间地址为关键字），两个红黑树的节点都是`binder_buffer.rb_node`（因为一个binder_buffer要么是空闲的，要么是正在使用的，同一时刻仅仅会存在于一个红黑树中）。而成员变量`buffer_free`则保存了空闲内核缓冲区的大小；成员变量`free_async_space`保存了当前可以用来保存异步事务数据的内核缓冲区的大小。

12. int max_threads、int requested_threads和int requested_threads_started
	>每一个使用Binder进程间通信机制的进程都有一个Binder线程池，用来处理进程间通信请求，这个Binder线程池是由Binder驱动负责维护的。Binder线程池的线程，既可以由Server进程主动创建并注册到Binder线程池，也可以由Binder驱动主动请求Server进程注册更多的线程到Binder线程池。其中，前者的线程数不受限制，后者的线程数则受成员变量`max_threads`的限制，而成员变量`ready_threads`则表示进程当前的空闲Binder线程数目。

	>Binder驱动程序每一次主动请求进程注册一个线程时，都会将成员变量`requested_threads`的值加1，而当进程响应这个请求后，Binder驱动程序又会将该成员变量减1，同时将成员变量`requested_threads_started`的值加1，表示Bidner驱动程序已经主动请求进程注册了多少个线程到Binder线程池，此数目受成员变量`max_threads`的限制。

13. struct list_head todo、wait_queue_head_t wait和int ready_threads;
	>`todo`表示待处理的工作项队列。Binder线程池中空闲Binder线程会睡眠在由成员变量`wait`所描述的等待队列中，而成员变量`ready_threads`则表示进程当前空闲的Binder线程数目。当宿主进程的todo队列增加了新的工作项后，Binder驱动就会唤醒wait队列中的空闲线程，来处理新增的工作项。

14. long default_priority;
	>默认的线程优先级

15. struct binder_stats stats;
	>用来统计进程相关数据，例如：进程接收到的进程间通信请求的次数。

15. struct list_head delivered_death;
	>Binder驱动程序会将发送给进程的死亡通知，封装成一个类型为BINDER_WORK_DEAD_BINDER工作项，并保存在该成员变量所描述的队列中。当该进程处理完一个死亡通知时，会通知BInder驱动程序，从该队列中删除对应的工作项。

16. struct page ** pages;
	> 该成员变量是类型为struct page * 的一个数组，数组中的每一个元素都指向一个物理页面。Binder驱动一开始只会为进程的内核缓冲区分配一个物理页面，后面不够时，再继续分配。

### Binder线程池的线程（Bidner_thread）

Binder线程池中的线程使用`Bidner_thread`结构体来描述:
``` c++
struct binder_thread {
struct binder_proc *proc;
struct rb_node rb_node;
int pid;
int looper;
struct binder_transaction *transaction_stack;
struct list_head todo;
uint32_t return_error; /* Write failed, return error code in read buf */
uint32_t return_error2; /* Write failed, return error code in read */
/* buffer. Used when sending a reply to a dead process that */
/* we are also waiting on */
wait_queue_head_t wait;
struct binder_stats stats;
};
```
相关字段的含义如下所示：

1. struct binder_proc * proc;
	>指向宿主进程。
	
2. struct rb_node rb_node;
	>宿主进程用一个红黑树（`binder_proc.threads`）来组织Binder线程池中的线程，该成员变量就是该红黑树中的一个节点。
	
3. int pid;
	>线程ID
	
4. int looper;
	>线程状态，参见老罗书P153页。
	
5. struct binder_transaction * transaction_stack;
	> 当Binder驱动程序决定将一个事务交给一个Binder线程处理时，它会将该事务封装成一个`binder_transaction`结构体，并且将它添加到由成员变量`transaction_stack`所描述的一个事务堆栈中。
	
6. struct list_head todo 和wait_queue_head_t wait;
	>`todo`表示该线程待处理的工作项队列。成员变量`wait`表示线程等待（睡眠）队列（实际上，最多就只会包含该线程本身）。当Client进程的请求指定要由某一个线程处理时，才会将该工作项添加到该线程的todo队列，否则会添加到进程的todo队列。
	
7. uint32_t return_error和uint32_t return_error2;
	>当Binder线程在处理一个事务时，若出现了异常情况，那么Binder驱动就会将相应的错误码保存在其成员变量return_error和return_error2中，然后线程就会将这些错误返回给用户空间应用程序处理。

9. struct binder_stats stats;
	>统计Binder线程的数据，如：Binder线程接收到的进程间请求次数。


###  Binder事务Binder_transaction
该结构体用来描述进程间通信过程，称之为一个事务，其结构体如下所示：
``` c++
struct binder_transaction {
int debug_id;
struct binder_work work;
struct binder_thread *from;
struct binder_transaction *from_parent;
struct binder_proc *to_proc;
struct binder_thread *to_thread;
struct binder_transaction *to_parent;
unsigned need_reply:1;
/* unsigned is_dead:1; */	/* not used at the moment */
struct binder_buffer *buffer;
unsigned int	code;
unsigned int	flags;
long	priority;
long	saved_priority;
uid_t	sender_euid;
};
```
相关字段的含义如下所示：

1. int debug_id;
	>标识一个事务结构体的身份，用来帮助调试程序。
	
2. struct binder_work work;
	>和该事务相关联的工作项。当Binder驱动为目标进程或者线程创建一个事务时，就会将该事务的成员变量work的类型设置为BINDER_WORK_TRANSACTION,并且将它（binder_work）添加到目标进程或者线程的todo队列中去等待处理。

3. struct binder_thread *from;
	>发起事务的线程，称为源线程。

4. struct binder_transaction *from_parent;
	> 该事务依赖的另外一个事务。

5. struct binder_proc *to_proc和struct binder_thread *to_thread;
	>负责处理该事务的目标进程和目标线程。

6. struct binder_transaction *to_parent;
	> 目标线程下一个需要处理的事务。

7. unsigned need_reply；
	>1:同步事务  0：异步事务

8. struct binder_buffer *buffer;
	>buffer指向Binder驱动程序为该事务分配的一块内核缓冲区，它里面保存了进程间通信数据。

9. unsigned int code和unsigned int flags;
	>这些标志直接从进程间通信数据（`binder_transaction_data.code和binder_transaction_data.flags`）中拷贝过来，见后面的详解。
	
10. long	priority和long	saved_priority;
	>Binder线程在处理一个事务时，Binder驱动程序会修改Binder线程的优先级，以满足源线程和目标Service组件的要求。Binder驱动程序在修改一个线程的优先级之前，会将它原来的线程优先级保存在成员变量`saved_priority`中，以便线程处理完该事务后可以恢复到原来的线程优先级。

	>目标线程的优先级取决于目标进程的默认优先级，即`Binder_proc.default_priority`，和目标Binder实体的最低优先级，即`Binder_node.min_priority`。目标线程的优先级会取两者的最大值。

11. uid_t	sender_euid;
	> 源线程的用户ID

---

以上介绍的结构体都是在Binder驱动程序内部使用的，下面介绍直接和我们打交道的结构体。

### 和Binder驱动交互的数据结构
BInder驱动程序对外提供的函数包括：open、mmap和ioctl，其中ioctl函数是最重要的。Binder协议基本格式是（命令+数据），使用ioctl(fd, cmd, arg)函数实现交互。命令由参数cmd承载，数据由参数arg承载，随cmd不同而不同。

Binder协议的基本命令包括以下几个：
1. BINDER_SET_MAX_THREADS
	> 后面紧跟int max_threads；表示最大线程数（其实是Binder驱动主动要求Server进程创建的线程数）。通常是在创建Service组件时，通知Binder驱动，Server进程支持的最大线程数。该命令告知Binder驱动，Server进程的Binder线程池中最大的线程数。由于Client是并发向Server端发送请求的，Server端必须开辟线程池为这些并发请求提供服务。告知Binder驱动线程池的最大值，是为了让驱动发现线程数达到该值时，不要再命令Server端启动新的线程。

2. BINDER_SET_CONTEXT_MGR
	>后面紧跟一个整型参数，用来描述一个与Service Manager对应的Binder本地对象的地址，与Service Manager对应的Binder本地对象是一个虚拟的对象，且它的地址值为0。作用是将当前进程注册为`ServiceManager`。系统中同时只能存在一个ServiceManager。只要当前的ServiceManager没有调用close()关闭，Binder驱动就不会让别的进程可以成为ServiceManager。

3. BINDER_THREAD_EXIT
	> 后面不需要跟参数。作用是通知Binder驱动当前线程退出了。Binder驱动会为所有参与Binder通信的线程（包括Server线程池中的线程和Client发出请求的线程）建立相应的数据结构(binder_thread)。这些线程在退出时必须通知binder驱动释放相应的数据结构。

4. BINDER_VERSION
	> 后面不需要跟参数。作用是获得Binder驱动的版本号。

5. `BINDER_WRITE_READ`
	>后面紧跟`binder_write_read`结构体，用来描述进程间通信过程中所传输的数据。该命令是Binder通信命令中最重要的命令。作用是向Binder驱动写入或读取数据。下面对binder_write_read结构体进行详细的介绍。

#### 通信数据的载体Binder_write_read
其结构体如下所示：
``` c++
struct binder_write_read {
signed long	write_size;	/* bytes to write */
signed long	write_consumed;	/* bytes consumed by driver */
unsigned long	write_buffer;
signed long	read_size;	/* bytes to read */
signed long	read_consumed;	/* bytes consumed by driver */
unsigned long	read_buffer;
};
```
其中，成员变量write_size、write_consumed和write_consumed用来描述输入数据，即`从用户空间传输到Binder驱动程序的数据`，也是进程间通信的请求数据；而成员变量read_size、read_consumed和read_buffer则用来描述输出数据，即`从Binder驱动程序返回给用户空间的数据`，也是进程间通信的结果（返回）数据。

`write_buffer`指向一个用户空间缓冲区的地址，里面保存的请求数据就是要传输到Binder驱动的数据；`write_size`表示该缓冲区的大小；`write_consumed`**则表示Binder驱动程序从用户空间缓冲区中读取了多少字节的数据**。

`read_buffer`也是指向一个用户空间缓冲区的地址，里面保存的的内容即为Binder驱动程序返回给用户空间的进程间通信结果数据；`read_size`表示该缓冲区的大小；`read_consumed`**则表示Binder驱动程序写入了多少字节的数据给用户空间应用程序**。

从上文可知，`write_buffer`和`read_buffer`指向的缓冲区才是真正的有效负荷数据。

有效负荷数据的命令格式也是由`命令+数据`构成，并且支持多条命令连续存放，一起操作。下面来看下这里Binder支持的Cmd命令。
#### 适用于binder_write_read的子命令字
binder_write_read.write_buffer用于向Binder驱动写入数据。binder_write_read.read_buffer用于从Binder驱动读入数据。
write_buffer和read_buffer都是一个数组，数组的每个元素都由一个通信协议代码及其通信数据组成。协议代码又分成两部分，其中一种在输入缓冲区write_buffer中使用的称为`命令协议代码`，另一种在输出缓冲区read_buffer中使用的称为`返回协议代码`。

##### 命令协议代码
1. BC_TRANSACTION和BC_REPLY
	>BC_TRANSACTION用于Client向Server发送请求数据；BC_REPLY用于Server向Client发送回复（应答）数据。其后面紧接着一个binder_transaction_data结构体,表明要写入的数据。

2. BC_FREE_BUFFER
	>后面紧跟一个地址值，表示Binder实体的内核缓冲区地址。Binder驱动程序就是通过这个内核缓冲区将源进程的通信数据传递到目标进程的。当目标进程处理完源进程的通信请求后，它就会通过命令协议代码BC_FREE_BUFFER来通知Binder驱动程序来释放这块内存缓冲区。

3. BC_INCREFS、BC_DECREFS、BC_ACQUIRE、BC_RELEASE 
	>后面紧跟32位Binder引用号，表示Binder引用对象的句柄值。BC_ACQUIRE和BC_RELEASE命令协议分别用来增加和减少一个Binder引用对象的强引用计数。而BC_INCREFS和BC_DECREFS命令协议则分别用来增加和减少一个Binder引用对象的弱引用计数。

4. BC_INCREFS_DONE和BC_ACQUIRE_DONE
	>后面紧跟binder_ptr_cookie结构体，描述一个binder实体对象。Binder驱动程序第一次增加一个Binder实体对象的强引用计数或者弱引用计数时，会使用返回协议代码BR_INCREFS和BR_ACQUIRE请求对应的Server进程增加对应的Service组件的强引用计数或者弱引用计数。当Server进程处理完这两个请求后，就会分别使用命令协议代码BC_INCREFS_DONE和BC_ACQUIRE_DONE将操作结果返回给binder驱动程序。

5. BC_REGISTER_LOOPER 
	>后面不跟参数，当Binder驱动程序主动请求Server进程注册一个新的线程到它的Binder线程池中，来处理进程间通信请求之后，新创建的线程就会使用命令协议代码`BC_REGISTER_LOOPER`来通知Binder驱动程序，它准备就绪了。

6. BC_ENTER_LOOPER
	>后面不跟参数，当Server进程主动启动一个线程，并将自己注册到Binder驱动程序后，它接着就会使用命令协议代码`BC_ENTER_LOOPER`来通知Binder驱动程序，它已经准备就绪处理进程间通信请求了。

7. BC_EXIT_LOOPER
	>后面不跟参数，当一个线程要退出时，它就会使用命令协议代码BC_EXIT_LOOPER通知binder驱动，该线程退出主循环了，不再接收数据。

8. BC_REQUEST_DEATH_NOTIFICATION
	> 后面紧跟binder_ptr_cookie结构体，描述一个用来接收死亡通知的对象地址。ptr（指针）指向需要得到死亡通知的Binder引用。cookie（指针）指向与死亡通知相关的信息，Binder驱动会在发出死亡通知时，返回给发出请求的进程。该cookie值会保存在`binder_ref.death.cookie`中。当一个进程希望获得它所引用的Service组件的死亡接收通知时，就需要通过该命令协议代码向Binder驱动程序注册一个死亡接收通知。

9. BC_CLEAR_DEATH_NOTIFICATION
	>后面紧跟binder_ptr_cookie结构体，描述一个用来接收死亡通知的对象地址（上同）。当一个进程希望注销之前注册的一个死亡接收通知，就需要通过该命令协议代码向Binder驱动程序发送请求。Binder驱动程序就会创建一个binder_ref_death结构体，然后保存在对应的Binder引用对象的成员变量death中。

10. BC_DEAD_BINDER_DONE
	>后面紧跟一个指针，指向一个死亡接收通知结构体`binder_ref_death`的地址。当一个进程获得一个Service组件的死亡接收通知时，就会使用该命令协议代码来通知Binder驱动程序，它已经处理完该Service组件的死亡通知了。

11. BC_ACQUIRE_RESULT、BC_ATTEMPT_ACQUIRE 暂不支持。


##### 返回协议代码
1. BR_ERROR
	>后面紧跟一个整数，用来描述一个错误码。Binder驱动程序在处理应用程序进程发出的某个请求时，若发生了异常，就会通过该返回协议代码来通知应用程序进程。

2. BR_OK 
	> 后面不需要指定通信数据，Binder驱动程序成功处理了应用程序进程发出的某一个请求后，就会通过该返回协议代码来通知应用程序进程。

3. BR_TRANSACTION和BR_REPLY
	>后面紧跟binder_transaction_data结构体，当一个Client进程通过BC_TRANSACTION命令向一个Server进程发出进程间请求时，Binder驱动程序就会使用返回协议代码BR_TRANSACTION来通知Server进程来处理该请求。当Server进程处理完该进程间通信请求后，会通过BC_REPLY命令向Client进程返回处理结果，此时Binder驱动程序就会使用返回协议代码BR_REPLY将进程间通信处理结果返回给Client进程。

4. BR_DEAD_REPLY
	>后面不需要指定通信数据，Binder驱动程序在处理进程间通信请求时，如果发现目标进程or目标线程已经死亡，就会通过该返回协议代码来通知源进程。

5. BR_TRANSACTION_COMPLETE
	>后面不需要指定通信数据，当Binder驱动程序接收到应用程序进程给它发送的命令协议代码BC_TRANSACTION或者BC_REPLY后，它就会通过该返回协议代码来通知应用程序进程，该命令协议代码已经被接收了，正在分发给目标进程或者目标线程。

	>这和BR_REPLY不一样，是驱动程序告知发送方已经发送成功，而不是Server端返回请求数据。`所以，不管同步还是异步交互，接收方都能获得本消息`。

6. BR_INCREFS、BR_DECREFS、BR_ACQUIRE、BR_RELEASE
	>后面紧跟binder_ptr_cookie结构体，描述一个Binder实体对象。其中，BR_INCREFS、BR_DECREFS分别用来增加和减少一个Service组件的弱引用计数；而BR_ACQUIRE、BR_RELEASE则分别用来增加和减少一个Service组件的强引用计数。`只有提供Binder实体的进程才能收到这组消息。`

7. BR_SPAWN_LOOPER
	>后面不需要指定通信数据，该消息用于接收方线程池管理。当Binder驱动程序发现接收方所有线程都处于忙碌状态且线程池里的线程总数没有超过BINDER_SET_MAX_THREADS设置的最大线程数时，就会使用该返回协议代码，来通知接收方进程增加一个新的线程到Binder线程池中。

8. BR_FAILED_REPLY
	>后面不需要指定通信数据，当Binder驱动程序处理一个进程发出的BC_TRANSACTION命令协议时，若发生了异常情况（如：Binder引用号非法），就会通过该返回协议代码来通知源进程。

9. BR_DEAD_BINDER 
	>后面紧跟一个void类型的指针，即`binder_ref.death.cookie`值，表示注册接收死亡通知时的附加值。当Binder驱动程序检测到一个Service组件的死亡事件时，就会通过该返回协议码来通知相应的Client进程。

10. BR_CLEAR_DEATH_NOTIFICATION_DONE
	>后面紧跟一个void类型的指针，即`binder_ref.death.cookie`值，表示注册接收死亡通知时的附加值。当Client进程向Binder驱动注销之前注册的死亡接收通知时，Binder驱动程序执行完这个注销操作后。就会通过该返回协议码来通知源进程。

11. BR_NOOP
	>后面不需要指定通信数据，Binder驱动程序使用该返回协议码来通知应用程序进程执行一个空操作，它存在的目的是为了以后可以替换为BR_SPAWN_LOOPER。

12. BR_ACQUIRE_RESULT、BR_ATTEMPT_ACQUIRE、BR_FINISHED 尚未支持

上面提到了多次提到binder_ptr_cookie结构体，其结构如下所示：
``` c++
struct binder_ptr_cookie{
void * ptr;
void * cookie;
}
```
该结构体用来描述一个Binder实体对象或者一个Service组件的死亡接收通知。

当描述Binder实体时，成员变量ptr和cookie的含义等同于Binder_node的成员变量ptr和cookie，分别表示`指向用户空间Binder实体的指针`和`指向用户空间的附加指针`。

当描述Service组件的死亡接收通知时，成员变量ptr（指针）指向希望得到死亡通知的Binder引用。成员变量cookie（指针）指向与死亡通知相关的信息，该cookie值会保存在`binder_ref.death.cookie`中，Binder驱动会在发出死亡通知时，返回给发出请求的Client进程。

#### 进程间通信的真正数据载体Binder_transaction_data
Binder_transaction_data用来描述进程间通信过程中所传输的数据，其结构体如下所示：
``` c++
struct binder_transaction_data {
/* The first two are only used for bcTRANSACTION and brTRANSACTION,
* identifying the target and contents of the transaction.
*/
union {
	size_t	handle;	/* target descriptor of command transaction */
	void	*ptr;	/* target descriptor of return transaction */
} target;
void		*cookie;	/* target object cookie */
unsigned int	code;		/* transaction command */

/* General information about the transaction. */
unsigned int	flags;
pid_t		sender_pid;
uid_t		sender_euid;
size_t		data_size;	/* number of bytes of data */
size_t		offsets_size;	/* number of bytes of offsets */

/* If this transaction is inline, the data immediately
 * follows here; otherwise, it ends with a pointer to
 * the data buffer.
*/
union {
struct {
	/* transaction data */
	const void	*buffer;
	/* offsets from buffer to flat_binder_object structs */
	const void	*offsets;
	} ptr;
	uint8_t	buf[8];
} data;
;
```

1.  union {
    size_t  handle; 
    void    * ptr; 
} target;

	>成员变量target是一个联合体，用来描述一个目标Binder实体对象或者目标Binder引用对象。对于发送数据包的一方，该成员指明发送目的地。若描述的是一个Binder实体对象，那么它的成员变量ptr就指向Binder实体对象在用户空间的地址。若描述的是一个Binder引用对象，那么它的成员变量handle就表示指向Binder引用对象的句柄值。
	
	>若是Client进程向Server进程发送数据，那么需要填写handle句柄值（指向binder_node的binder_ref的成员变量的desc的值）。Binder驱动程序会自动将该句柄值转换为对应的Binder实体对象在用户空间的地址。这样，Server进程便可以直接将其当做对象指针来使用（通常是将其reinterpret_cast成相应类）。

	>反过来的转换，貌似没有应用场景哈？？？

2. void *cookie; 
	>Client进程忽略该成员；Binder驱动会自动赋值为binder_node.cookie的值（binder_node.cookie的值又来自于flat_binder_object.cookie的值）。驱动基本上不关心该成员。

3. unsigned int    code;
	>该成员存放收发双方约定的命令码，驱动完全不关心该成员的内容。通常是Server端定义的公共接口函数的编号。

4. unsigned int    flags;
	>flags是一个标志值，用来描述进程间通信行为特征。其取值如下所示：
	``` c++
	enum transaction_flags {
	 TF_ONE_WAY	= 0x01,	/* this is a one-way call: async, no return 若该位被设置为1，就表示这是一个异步的进程间通信过程*/
	 TF_ROOT_OBJECT	= 0x04,	/* contents are the component's root object 暂无用*/
	 TF_STATUS_CODE	= 0x08,	/* contents are a 32-bit status code 若该位被设置为1，就表示成员变量data所描述的数据缓冲区的内容是一个4字节的状态码*/
	 TF_ACCEPT_FDS	= 0x10,	/* allow replies with file descriptors 若该位被设置为0，就表示源进程不允许目标进程返回的结果数据中包含有文件描述符即文件Binder,因为收到一个文件形式的Binder，会自动为数据接收方打开一个文件，使用该标志位可以防止打开文件过多*/
	};
	```

5. pid_t  sender_pid和uid_t  sender_euid;
	>该成员存放发送方进程的PID和UID，由Binder驱动负责填入，接收方可以读取该成员获知发送方的身份，以便进行安全检查。

6. size_t  data_size和size_t   offsets_size
	>data_size表示成员变量`data.ptr.buffer`指向的数据缓冲区的大小，字节；offsets_size表示成员变量`data.ptr.offsets`指向的偏移数组的大小，字节。

7. union {
struct {
     const void  *buffer;
     const void  *offsets;
    } ptr;
    uint8_t buf[8];
} data;

	>data是一个联合体，表示通信数据缓冲区。若通信数据较小，就直接使用联合体内的数组buf来传输数据；若数据较大，就需要使用一块动态分配的数据缓冲区来传输数据了。这块动态分配的数据缓冲区通过一个结构体ptr来表示。其中，`ptr.buffer`指向一个数据缓冲区，他是真正用来保存通信数据的，他的大小由前面所描述的成员变量`data_size`来描述。当数据缓冲区中包含Binder对象时（Binder引用 or Binder实体），那么紧跟在这个数据缓冲区后面就会有一个偏移数组，用来描述数据缓冲区中每一个Binder对象相对于`ptr.buffer`的偏移量。`ptr.offsets`即指向这个偏移数组的首地址。偏移数组的大小由前面所述的`offsets_size`来表示。

	>数据缓冲区中的Binder对象使用`flat_binder_object`结构体来描述。


#### 数据缓冲区中的Binder对象flat_binder_object
Binder对象可以塞在数据包的有效数据中，从一个进程传递给另一个进程，这些传输中的Binder用结构flat_binder_object表示，如下表所示：
``` c++
struct flat_binder_object {
/* 8 bytes for large_flat_header. */
unsigned long		type;
unsigned long		flags;
/* 8 bytes of data. */
union {
	void		*binder;	/* local object */
	signed long	handle;		/* remote object */
};
/* extra data associated with local object */
void			*cookie;
};
```
相关字段的含义如下所示：

1. unsigned long		type;
	> 表明该Binder对象的类型，包括以下几种：
	``` c++
	enum {
	 //表示传递的是Binder实体，并且指向该实体的引用都是强类型
	 BINDER_TYPE_BINDER	= B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),
	 //表示传递的是Binder实体，并且指向该实体的引用都是弱类型；
	 BINDER_TYPE_WEAK_BINDER	= B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),
	 //表示传递的是Binder强类型的引用
	 BINDER_TYPE_HANDLE	= B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),
	 //表示传递的是Binder弱类型的引用
	 BINDER_TYPE_WEAK_HANDLE	= B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),
	 //表示传递的是文件形式的Binder
	 BINDER_TYPE_FD		= B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),
	};
	```

2. unsigned long   flags;
	> 一个标志位，只有当第一次传递Binder实体时有效(1、flat_binder_object表示Binder实体对象；2、第一次跨进程传递该Binder实体对象)，因为此刻Binder驱动需要在内核中创建相应的实体节点（binder_node），有些参数需要从该域取出：
		>>第0-7位：代码中用FLAT_BINDER_FLAG_PRIORITY_MASK取得，表示本Binder实体对象处理进程间通信请求时，所运行的线程应当具备的最小线程优先级。会保存在`binder_node.min_priority`中。
		>>第8位：代码中用FLAT_BINDER_FLAG_ACCEPTS_FDS取得，置1表示该Binder实体可以接收其它进程发过来的文件形式的Binder。由于接收文件形式的Binder会在本进程中自动打开文件，有些Server可以用该标志禁止该功能，以防打开过多文件。

3. void * cookie;
	>该域只对Binder实体有效，存放与该Binder实体有关的附加信息，保存在`binder_node.cookie`中，后续会由Binder驱动填充到`binder_transaction_data.cookie`成员变量中。	

4. union {
    void  * binder;    //local object
    signed long handle;     //remote object
};
	>当传递的是Binder实体时，使用binder域指向Binder实体在应用程序中的地址。
	>当传递的是Binder引用时，使用handle域存放Binder引用在进程中的句柄值。
	>在binder进程间传递时，binder驱动会动态的修改这两个值和type类型，详见下面的列表。


无论是Binder实体还是对实体的引用都从属与某个进程，所以该结构不能透明地在进程之间传输，必须经过驱动翻译。

例如当Server把Binder实体传递给Client时，在发送数据流中，flat_binder_object中的type是BINDER_TYPE_BINDER，binder指向Server进程的用户空间地址。如果透传给接收端将毫无用处，驱动必须对数据流中的这个Binder做修改：将type改成BINDER_TYPE_HANDLE；为这个Binder实体在接收进程中创建位于内核中的引用（binder_ref）并将引用号填入handle中。对于数据流中的Binder引用句柄，也要做同样转换。经过处理后，接收进程从数据流中取得的Binder引用句柄才是有效的，才可以将其填入数据包binder_transaction_data的target.handle域，向Binder实体发送请求。

这样做也是出于安全性考虑：应用程序不能随便猜测一个引用号填入target.handle中就可以向Server请求服务了，因为Binder驱动并没有为你在内核中创建对应的Binder引用，必定会被驱动拒绝。唯有经过身份认证确认合法后，由“权威机构”（Binder驱动）亲手授予你的Binder引用（句柄）才能使用。

下表总结了当flat_binder_object结构穿过Binder驱动时，驱动所做的操作：

| Binder 类型（ `flat_binder_object.type `） |在发送方的操作 |在接收方的操作 |
| -------- | ----- | ---- |
|BINDER_TYPE_BINDER、BINDER_TYPE_WEAK_BINDER|只有Server进程才能发送该类型的Binder。如果是第一次发送Binder实体，Binder驱动将在内核中创建对应的实体节点（Binder_node），并保存binder，cookie，flag域。|如果是第一次接收该Binder实体，Binder驱动将在内核中创建对应的引用节点（Binder_ref），并将handle域替换为新建的引用号，将type域替换为BINDER_TYPE_(WEAK_)HANDLE|
|BINDER_TYPE_HANDLE、BINDER_TYPE_WEAK_HANDLE|获得Binder引用的进程都能发送该类型Binder。Binder驱动根据handle域提供的引用号，查找在内核的Binder引用（Binder_ref）。如果找到，则说明引用号合法，否则拒绝该发送请求。|如果收到的Binder实体位于接收进程中：将binder域替换为保存在binder_node节点中的binder值（Binder实体在用户空间的地址）；cookie替换为保存在binder_node节点中的cookie值；type替换为BINDER_TYPE_(WEAK_)BINDER。如果收到的Binder实体不在接收进程中：若是第一次接收，则创建Binder实体在内核中的Binder引用（binder_ref），并将handle域替换为新建的引用号（每个进程都会有独立的引用号）|
|BINDER_TYPE_FD|验证handle域中提供的打开文件号是否有效，无效则拒绝该发送请求。|在接收方创建新的打开文件号，并将其与给定的文件描述结构绑定。|

##### 文件形式的 Binder
除了通常意义上用来通信的Binder，还有一种特殊的Binder：文件Binder。这种Binder的基本思想是：将文件看成Binder实体，进程打开的文件号看成Binder的引用。一个进程可以将它打开文件的文件号传递给另一个进程，从而另一个进程也打开了同一个文件，就象Binder的引用在进程之间传递一样。

一个进程打开一个文件，就获得与该文件绑定的打开文件号。**从Binder的角度，linux在内核创建的打开文件描述结构struct file是Binder的实体，打开文件号是该进程对该实体的引用。** 既然是Binder，那么就可以在进程之间传递，故也可以用flat_binder_object结构，将文件Binder通过数据包发送至其它进程，只是结构中type域的值为BINDER_TYPE_FD，表明该Binder是文件Binder。而结构中的handle域则存放文件在发送方进程中的打开文件号。

我们知道打开文件号是个局限于某个进程的值，一旦跨进程就没有意义了。若要跨进程同样需要驱动做转换。驱动在接收Binder的进程空间创建一个新的打开文件号，将它与已有的打开文件描述结构struct file勾连上，从此该Binder实体又多了一个引用。新建的打开文件号覆盖flat_binder_object中原来的文件号（handle域）交给接收进程。接收进程利用它可以执行read()，write()等文件操作。

传个文件为啥要这么麻烦，直接将文件名用Binder传过去，接收方用open()打开不就行了吗？其实这还是有区别的。首先对同一个打开文件共享的层次不同：使用文件Binder打开的文件共享linux VFS中的struct file，struct dentry，struct inode结构，这意味着一个进程使用read()/write()/seek()改变了文件指针，另一个进程的文件指针也会改变；而如果两个进程分别使用同一文件名打开文件则有各自的struct file结构，从而各自独立维护文件指针，互不干扰。其次是一些特殊设备文件要求在struct file一级共享才能使用，例如android的另一个驱动ashmem，它和Binder一样也是misc设备，用以实现进程间的共享内存。一个进程打开的ashmem文件只有通过文件Binder发送到另一个进程才能实现内存共享，这大大提高了内存共享的安全性，道理和Binder增强了IPC的安全性是一样的。
