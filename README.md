# Linux-Experiment-4
linux继承了unix v的进程间通信机制（IPC是Inter-Process Communication），即消息队列、信号量和共享主存。通常把消息、信号量和共享主存统称为system v ipc对象或ipc资源。系统在进程请求ipc对象时在内核动态创建一个ipc数据结构并初始化，该结构定义了对该ipc对象访问的许可权， 每个对象都具有同样类型的接口-函数，这组函数为应用程序提供3种服务：
 
通过信号量实现与其他进程的同步
 
通过消息队列以异步方式为通信频繁但数据量少的进程间通信提供服务
 
通过共享主存，为大数据量的进程间通信提供服务
 
# 1、共性描述
## 1.1、公有的数据结构
每一类ipc资源都有一个ipc_ids结构的全局变量用来描述同一类资源的共用数据，三种ipc对象对应的三个全局变量分别是semid_ds,msgid_ds和shmid_ds（具体结构在本文2.1介绍）。
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

2）调用函数ftok（）产生一个惟一的关键字值。通过IPC进行通信的进程，只需要按照相同的参数调用ftok即可产生惟一的参数。通过该参数可有效解决关键字的产生及惟一性问题。通过该参数可有效解决关键字的产生及惟一性问题。

## 1.2、linux提供的ipc函数
（1）ipc对象的创建函数
```
semget（）：获得信号量的ipc标识符
msgget（）：获得消息队列的ipc标识符
shmget（）：获得共享主存的ipc标识符
```
（2）ipc资源控制函数

创建ipc对象后，用户可以通过下面的库函数对ipc对象进行控制
```
semctl（）：对信号量资源进行控制
msgctl（）：对消息队列进行控制
shmctl（）：对共享主存进行控制
```
（3）资源操作函数
```
semop（）：用于对信号量资源进行操作，获得或释放一个ipc信号量
msgsnd（）及msgrcv（）:发送和接收一个ipc消息
shmat（）及shmdt（）:分别将一个ipc共享主存区附加到进程的虚拟地址空间，以及把共享主存区从进程的虚拟地址空间剥离出去
```
# 2、消息队列
Linux 中的消息可以被描述成在内核地址空间的一个内部链表，每一个消息队列由一个IPC的标识号唯一的标识。Linux 为系统中所有的消息队列维护一个 msg_queue 链表，该链表中的每个指针指向一个 msgid_ds 结构，该结构完整描述对应的一个消息队列。
## 2.1、消息队列结构(msgid_ds) 
```
struct msqid_ds { 
	struct ipc_perm msg_perm; /* 用于权限和拥有者信息 */ 
	struct msg *msg_first; /* 队列上第一条消息，即链表头 */ 
	struct msg *msg_last; /* 队列中的最后一条消息，即链表尾 */ 
	time_t msg_stime; /* 发送给队列的最后一条消息的时间 */ 
	time_t msg_rtime; /* 从消息队列接收到的最后一条消息的时间 */ 
	time_t msg_ctime; /* 最后修改队列的时间*/ 
	ushort msg_cbytes; /*队列上所有消息总的字节数 */ 
	ushort msg_qnum; /*在当前队列上消息的个数 */ 
	ushort msg_qbytes; /* 队列最大的字节数 */ 
	ushort msg_lspid; /* 发送最后一条消息的进程的pid */ 
	ushort msg_lrpid; /* 接收最后一条消息的进程的pid */ 
	};
```
## 2.2、消息结构(msg) 
内核把每一条消息存储在以msg结构为框架的队列中
```
struct msg { 
	struct msg *msg_next; /* 队列上的下一条消息 */ 
	long msg_type; /*消息类型*/ 
	char *msg_spot; /* 消息正文的地址 */ 
	short msg_ts; /* 消息正文的大小 */ 
	};
```
