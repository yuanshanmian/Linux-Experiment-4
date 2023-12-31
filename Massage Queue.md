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
## 2.3、消息缓冲区(msgbuf) 
对发送消息，先预设一个msgbuf缓冲区并写入消息类型何内容，调用相应的发送函数。对读取消息，先分配一个msgbuf缓冲区。然后把消息读入该缓冲区
```
struct msgbuf { 
	long mtype; /* use this to represent different kinds of messages, allowing the applications to process them differently based on it */ 
	char mtext[1]; /* 消息正文 */ 
	};
```
## 2.4、消息队列基本操作
（1）相应头文件
```
#include <sys/types.h> /* 包含了一些基本数据类型的定义，通常用于在系统级别定义标准数据类型，如整数类型、进程ID等 */
#include <sys/ipc.h> /* 包含了与进程间通信（IPC）的键（key）相关的定义，以及用于创建和管理IPC对象的函数原型。key 是一个整数值，它用于唯一标识IPC对象，如消息队列 */
#include <sys/msg.h> /* 包含了与消息队列相关的定义和函数原型。它包括了用于创建、发送和接收消息的函数以及消息队列的属性和结构的定义，例如 struct msqid_ds，它用于描述消息队列的属性，如前面所述 */
```
（2）打开或创建消息队列
为了创建一个新的消息队列，或存取一个已经存在的队列，要使用msgget()系统调用
```
int msgget(key_t kye,int flag);
/* key：是一个函数，是创建/打开队列的键值，直接用常量指定或由ftok（）函数产生 */
/* flag: is an integer value that specifies various flags for the msgget operation. typically used like 'IPC_CREAT | 0666', means bitwise ORed with the permissions value (0666) to set the desired permissions for the message queue. */
/* Some commonly used flags include:*/
/* 	ipc_create：create a new message queue. If the specified key exists, it has no effect) */
/*	ipc_excl：exculsive. Typically used with ipc_create to ensure created, and it'll fail if the specified key exist */
/*	ipc_nowait：used to indicate that a particular operation should be non-blocking. If the desired condition unsatisfied, it will return immediately with an error. */
/*	or the '|' rusult of the above three, type is symbolic constant, all defined in <sys/ipc.h>*/
/* 0666: permission value means that the message queue will be created with read and write permissions for all users (readable and writable by the owner, the group, and others). */
```
（3）发送消息
```
int msgsnd ( int msqid, struct msgbuf *msgp, int msgsz,int msgflg ); 
/* msqid：是队列标识符，由msgget()调用返回 */
/* msgp：是一个指针，指向我们重新声明和装载的消息缓冲区 */
/* msgsz：包含了消息以字节为单位的长度，其中包括了消息类型的4个字节 */
/* msgflg：可以设置成0(忽略)，或者IPC_NOWAIT(非阻塞操作)、MSG_NOERROR(perform operation without generating error messages)、MSG_EXCEPT(接收与 mtype 不匹配的消息),如果为IPC_NOWAIT，且消息队列满，那么消息不写到队列中，并且控制权返回给调用进程(继续执行)。如果不指定IPC_NOWAIT，调用进程将挂起(阻塞)直到消息被写到队列中 */
```
（4）读取消息
```
int msgrcv ( int msqid, struct msgbuf *msgp, int msgsz,long mtype, int msgflg );
/* msqid：用来指定要检索的队列(必须由msgget()调用返回) */
/* msgp：是存放检索到消息的缓冲区的地址 */
/* msgsz：是消息缓冲区的大小，包括消息类型的长度(4字节) */
/* mtype：指定了消息的类型，如果传递给mytype参数的值为0，就可以不管类型，只返回队列中最早的消息 */
/* msgflg：可以设置成0(忽略)，或者IPC_NOWAIT(非阻塞操作)、MSG_NOERROR(不返回错误信息)、MSG_EXCEPT(接收与 mtype 不匹配的消息)，如果为IPC_NOWAIT，并且没有可取的消息，那么给调用进程返回ENOMSG错误消息，操作将立即返回，而不会阻塞当前进程。若不为IPC_NOWAIT，调用进程阻塞，直到一条消息到达队列并且满足msgrcv()的参数 */
/* return: the number of bytes in the message that was successfully received */
```
第一个参数用来指定要检索的队列(必须由msgget()调用返回)，第二个参数(msgp)，第三个参数(，第四个参数(。
（5）消息队列属性操作
```
int msgctl ( int msgqid, int cmd, struct msqid_ds *buf );
/* msgqid：是打开的消息队列id */
/* cmd：是规定的命令：IPC_STAT 读取消息队列的数据结构msqid_ds，并将其存储在buf指定的地址中；IPC_SET	设置消息队列的数据结构msqid_ds中的ipc_perm元素的值，这个值取自buf参数；IPC_RMID 从系统内核中移走消息队列 */
/* buf：don't like the msgbuf, msgbuf is used to store message content, this is used to store information about the message queue or to specify new attributes when using IPC_SET. */
```
（6）消息队列的应用
```
#include<sys/types.h>
#include<sys/msg.h>
#include<unistd.h>
#include<sys/ipc.h>
#include<stdio.h>
void msg_stat(int,struct msqid_ds);

int main()
{
	int gflags,sflags,rflags; /* operation flags like IPC_CREAT, IPC_EXCL */
	key_t key; /* call function key_t() */
	int msgid; /* message queue id return by msgget() */
	int reval; /* define a return value */

	struct msgsbuf
	{
		int mtype;
		char mtext[1]; /* text: type is char, lenth is 1 */
	}msg_sbuf; /* define a message send buffer msg_sbuf */

	struct msgmbuf{
	 	int mtype; /* message type is int */
	 	char mtext[10]; /* text: type is char, lenth is 10 */
	}msg_rbuf; /* define a message recive buffer msg_rbuf */

	struct msqid_ds msg_ginfo,msg_sinfo; /* define 2 message queue ID data structure for g and s */
	char *msgpath="/home/msgqueue"; /* specify path for message queue */
	key=ftok(msgpath,'a'); /* use msgpath to create a unique IPC key. 'proj_id' project identifier is the ASCLL value of the character 'a'. the result will be a integer */
	gflags=IPC_CREAT|IPC_EXCL; /* g flag used in message id create to create a new message queue with detection */
	msgid=msgget(key,gflags|00666); /* The 00666 permission value means that the message queue will be created with read and write permissions for all users (readable and writable by the owner, the group, and others).  */
	if(msgid==-1){ /* message queue ID creation failure */
	 printf("msg create error\n");
	 return;
	}

	msg_stat(msgid,msg_ginfo); /* call funtion msg_stat() for g to save current msqid_ds imformation */
	sflags=IPC_NOWAIT; /* s flag used in send message to do a particular operaion should be non-blocking */
	// content of s message
	msg_sbuf.mtype=10;
	msg_sbuf.mtext[0]='a';

	reval=msgsnd(msgid,&msg_sbuf,sizeof(msg_sbuf.mtext),sflags); /* send a message from msg_sbuf */
	if(reval==-1){ /* s message fail in sending failure */
	 printf("message send error\n");
	}

	msg_stat(msgid,msg_ginfo); /* call funtion msg_stat() for g to save current msqid_ds imformation */
	rflags=IPC_NOWAIT|MSG_NOERROR; /* r flag used in recive message to do a particular operaion should be non-blocking and failure operation will not send error messages */
	reval=msgrcv(msgid,&msg_rbuf,4,10,rflags); /* recive a message save in msg_rbuf */
	if(reval==-1){ /* failed to recive message*/
	 printf("read msg error\n");
	}
	else
 	printf("read from msg queue %d bytes\n",reval); /* print the number of bytes of recived message */

	msg_stat(msgid,msg_ginfo); /* call funtion msg_stat() for g to save current msqid_ds imformation */
	msg_sinfo.msg_perm.uid=8; /* update the owner of s message queue to user 8 in s message data structure buffer */
	msg_sinfo.msg_perm.gid=8; /* update the owner of s message queue to group 8 in s message data structure buffer */
	msg_sinfo.msg_qbytes=16388; /* update the maximun size of the message queue in the terms of the total size of all the messages to 16388 in s message data structure buffer */

	reval=msgctl(msgid,IPC_SET,&msg_sinfo); /* call funtion msgctl() to update message queue imformation */
	if(reval==-1){/* failed to update */
	 printf("msg set info error\n");
	 return;
	}

	msg_stat(msgid,msg_ginfo); /* call funtion msg_stat() for g to save current msqid_ds imformation */

	reval=msgctl(msgid,IPC_RMID,NULL); /* delete the message queue */
	if(reval==-1){/* failed to delete */
	 printf("unlink msg queue error\n");
	 return;
	}
}


void msg_stat(int msgid,struct msqid_ds msg_info)
{
 	int reval; /* define a return value */
	sleep(1); /* introduce a delay of one second */
 	reval=msgctl(msgid,IPC_STAT,&msg_info); /* get the msqid_ds information of msgid and save in msg_info */
 	if(reval==-1){ /* get the message queue imformation failed */
  	printf("get msg info error\n");
	return;
 }

 printf("\n");
 printf("current number of bytes on queue is %d\n",msg_info.msg_cbytes); /* print the number of bytes of all the message in message queue */
 printf("number of messages in queue is %d\n",msg_info.msg_qnum); /* print the number of the messages in the message queue */
 printf("max number of bytes on queue is %d\n",msg_info.msg_qbytes);/* print the maximun size of the message queue in the terms of the total size of all the messages */
 printf("pid of last msgsnd is %d\n",msg_info.msg_lspid); /* print the pid of the last sending message process */ 
 printf("pid of last msgrcv is %d\n",msg_info.msg_lrpid); /* print the pid of the last reciving message process */ 
 printf("last msgsnd time is%s",ctime(&(msg_info.msg_stime))); /* print the time of the last time sending message */ 
 printf("last msgrcv time is%s",ctime(&(msg_info.msg_rtime))); /* print the time of the last time reciving message */ 
 printf("last change time is%s",ctime(&(msg_info.msg_ctime))); /* print the time of the last time updating the message queue */ 
 printf("msg uid is%d\n",msg_info.msg_perm.uid); /* print Owner's user ID.  */
 printf("msg gid is%d\n",msg_info.msg_perm.gid); /* print  Owner's group ID.  */
}
```
