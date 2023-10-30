# Linux-Experiment-4
linux继承了unix v的进程间通信机制（IPC是Inter-Process Communication），即消息队列、信号量和共享主存。通常把消息、信号量和共享主存统称为system v ipc对象或ipc资源。系统在进程请求ipc对象时在内核动态创建一个ipc数据结构并初始化，该结构定义了对该ipc对象访问的许可权， 每个对象都具有同样类型的接口-函数，这组函数为应用程序提供3种服务：
 
通过信号量实现与其他进程的同步
 
通过消息队列以异步方式为通信频繁但数据量少的进程间通信提供服务
 
通过共享主存，为大数据量的进程间通信提供服务
 
# 1、共性描述
## 1.1、公有的数据结构
每一类ipc资源都有一个ipc_ids结构的全局变量用来描述同一类资源的共用数据，三种ipc对象对应的三个全局变量分别是semid_ds,msgid_ds和shmid_ds。
ipc_ids结构定义如下：
```
struct ipc_ids{
  int size;   /*entries数组的大小，，即可以管理的IPC对象的最大数量*/
  int in_use;	/*entries数组已使用的元素个数，即已经被使用的IPC对象的数量*/
  int max_id; /*记录分配的最大IPC对象标识符*/
  unsigned short  seq;  /*seq和seq_max这些字段用于生成唯一的IPC对象标识符*/
  unsigned short seq_max;
  struct semaphore sem; /*sem是用于同步访问 ipc_ids 结构的信号量。它确保只有一个进程可以在某一时刻修改这个结构*/
  spinlock_t ary; 		/*ary：这是一个自旋锁，用于控制对 entries 数组的访问。自旋锁是一种同步机制，用于防止多个进程同时访问共享资源*/
  struct ipc_id* entries;  /*entries：这是一个指向 ipc_id 结构的指针数组，用于存储各个IPC对象的信息*/
};

struct ipc_id{struct ipc_perm *p;};
```
每个ipc对象都有一个ipc_perm数据结构，用于描述其属性：

```
struct ipc_perm
{
    __key_t __key;	/* ipc键*/
    __uid_t uid;                /* Owner's user ID.  */
    __gid_t gid;                /* Owner's group ID.  */
    __uid_t cuid;        /* Creator's user ID.  */
    __gid_t cgid;        /* Creator's group ID.  */
    unsigned short int mode;/* Read/write permission.  */
    unsigned short int __seq;/* Sequence number.  */
};
```
每一个ipc对象都有一个32位的ipc键和一个ipc标识符，ipc标识符由内核分配给ipc对象，在系统内部惟一，IPC机制提供了相应的编程接口根据IPC键值获取标识符。对于一个IPC对象，在打开时返回一个标识符，而关闭后再次打开同一个IPC对象时，该标识符将顺序加1，而键是ipc对象的外部表示，可由程序员选择。不同的进程直接使用key去创建IPC对象容易引起混淆，并且不同的应用之间可能会因为使用同一个key而产生冲突。为此，Linux系统提供了如下机制产生惟一的关键字。
1）创建IPC对象时，指定关键字为IPC_PRIVATE。通过该参数创建的IPC对象的关键字值是0，所以无法在其他进程中通过关键字对该对象进行访问，只能通过返回的标识符进行访问。
2）调用函数ftok（）产生一个惟一的关键字值。通过IPC进行通信的进程，只需要按照相同的参数调用ftok即可产生惟一的参数。通过该参数可有效解决关键字的产生及惟一性问题。
