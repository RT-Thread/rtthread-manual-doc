# POSIX接口 #

POSIX Threads简称Pthreads，POSIX是"Portable Operating System Interface"（可移植操作系统接口） 的缩写，POSIX是IEEE Computer Society为了提高不同操作系统的兼容性和应用程序的可移植性而制定的一套标准。Pthreads是线程的POSIX标准，被定义在POSIX.1c, Threads extensions (IEEE Std1003.1c-1995)标准里，该标准定义了一套C程序语言的类型、函数和常量。定义在pthread.h头文件和一个线程库里，大约有100个API，所有API都带有"pthread_"前缀，可以分为4大类：

* 线程管理（Thread management）：包括线程创建（creating）、分离（detaching）、连接（joining）及设置和查询线程属性的函数等。

* 互斥锁（Mutex）："mutual exclusion"的缩写，用了限制线程对共享数据的访问，保护共享数据的完整性。包括创建、销毁、锁定和解锁互斥锁及一些用于设置或修改互斥量属性等函数。

* 条件变量（Condition variable）：用于共享一个互斥量的线程间的通信。包括条件变量的创建、销毁、等待和发送信号（signal）等函数。

* 读写锁（read/write lock）和屏障（barrier）：包括读写锁和栏杆的创建、销毁、等待及相关属性设置等函数。

* POSIX信号量（semaphore）和Pthreads一起使用，但不是Pthreads标准定义的一部分，被定义在POSIX.1b, Real-time extensions (IEEE Std1003.1b-1993)标准里。因此信号量相关函数的前缀是"sem\_"而不是"pthread\_"。

* 消息队列（Message queue）和信号量一样，和Pthreads一起使用，也不是Pthreads标准定义的一部分，被定义在IEEE Std 1003.1-2001标准里。消息队列相关函数的前缀是"mq\_"。

-----------------------------------------------------------------------
               函数前缀   函数组
-------------------------  ----------------------------------------------
  pthread\_                线程本身和各种相关函数
  pthread\_attr\_          线程属性对象
  Pthread\_mutex\_         互斥锁
  pthread\_mutexattr\_     互斥锁属性对象
  pthread\_cond\_          条件变量
  pthread\_condattr\_      条件变量属性对象
  pthread\_rwlock\_        读写锁
  pthread\_rwlockattr\_    读写锁属性对象
  pthread\_spin\_          自旋锁
  pthread\_barrier\_       栏杆
  pthread\_barrierattr\_   栏杆属性对象
  sem\_                    信号量
  mq\_                     消息队列
-----------------------------------------------------------------------
绝大部分Pthreads的函数执行成功则返回0值，不成功则返回一个包含在\<errno.h\>头文件中的错误代码。很多操作系统都支持Pthreads，比如Linux、MacOS X、 Android 和Solaris，因此使用Pthreads的函数编写的应用程序有很好的可移植性，可以在很多支持Pthreads的平台上直接编译运行。

## 在RT-Thread中使用POSIX ##

在RT-Thread中使用POSIX API接口包括几个部分：libc（例如newlib），file system，pthread等。需要在rtconfig.h中打开相关的选项：
```c
    #define RT_USING_LIBC
    #define RT_USING_DFS
    #define RT_USING_DFS_DEVFS
    #define RT_USING_PTHREADS
```
RT-Thread实现了Pthreads的大部分函数和常量，按照POSIX标准定义在pthread.h、mqueue.h、semaphore.h和sched.h头文件里。Pthreads是libc的一个子库，RT-Thread中的Pthreads是基于RT-Thread内核函数的封装，使其符合POSIX标准。后续章节会详细介绍RT-Thread中实现的Pthreads函数及相关功能。

## 线程 ##

### 线程句柄 ###
```c
    typedef rt\_thread\_t pthread\_t;
```
pthread\_t是rt\_thread\_t类型的重定义，定义在pthread.h头文件里。rt\_thread\_t是RT-Thread的线程句柄（或线程标识符），是指向线程控制块的指针。在创建线程前需要先定义一个pthread\_t类型的变量。每个线程都对应了自己的线程控制块，线程控制块是操作系统用于控制线程的一个数据结构，它存放了线程的一些信息，例如优先级，线程名称和线程堆栈地址等。线程控制块及线程具体信息在RT-Thread编程手册的线程调度与管理一章有详细的介绍。

### 创建线程 ###

**函数原型**

```c
    int pthread_create (pthread_t *tid, const pthread_attr_t *attr, void *(*start) (void *), void *arg);
```
-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
           tid  指向线程句柄(线程标识符)的指针，不能为NULL

          attr  指向线程属性的指针，如果使用NULL，则使用默认的线程属性

          star 	线程入口函数地址
		  
		  arg	传递给线程入口函数的参数	  
-----------------------------------------------------------------------
**函数返回**

创建成功返回0，参数无效返回EINVAL，动态分配内存失败返回ENOMEM。
此函数创建一个线程，主要是对rt\_thread\_init()函数的封装。此函数会动态分配POSIX线程数据块和RT-Thread线程控制块，并把线程控制块的起始地址（线程ID）保存在tid指向的内存里，此线程标识符可用于在其他线程中操作此线程。并把attr指向的线程属性、start指向的线程入口函数及入口函数参数arg保存在线程数据块和线程控制块里。如果线程创建成功，线程就进入就绪态，参与系统的调度，如果线程创建失败，则会释放之前线程占有的资源。关于线程属性及相关函数会在线程高级编程一章里有详细介绍，一般情况下采用默认属性就可以。

### 创建线程示例代码 ###

这个程序会初始化2个线程，它们拥有共同的入口函数，但是它们的入口参数不相同，相同的优先级，相同优先级的线程是按照时间片轮转调度。

```c

#include <pthread.h>
#include <unistd.h>
#include <stdio.h>

/* 线程控制块 */
static pthread_t tid1;
static pthread_t tid2;

/* 函数返回值检查 */
static void check_result(char* str,int result)
{
	if (0 == result)
	{
		printf("%s successfully!\n",str);
	}
	else
	{
		printf("%s failed! error code is %d\n",str,result);
	}	
}
/* 线程入口函数*/
static void* thread_entry(void* parameter)
{
    int count = 0;
    int no = (int) parameter; /* 获得线程的入口参数 */

    while (1)
    {
        /* 打印输出线程计数值 */
        printf("thread%d count: %d\n", no, count ++);

        sleep(2);	/* 休眠2秒 */
    }
}

/* 用户应用入口 */
int test()
{
    int result;
	
	/* 创建线程1,属性为默认值，入口函数是thread_entry，入口函数参数是1 */
	result = pthread_create(&tid1,NULL,thread_entry,(void*)1);
	check_result("thread1 created",result);

	/* 创建线程2,属性为默认值，入口函数是thread_entry，入口函数参数是2 */
	result = pthread_create(&tid2,NULL,thread_entry,(void*)2);
	check_result("thread2 created",result);
	
	return 0;
}


```
打开PuTTY查看程序的打印信息：


### 线程脱离 ###
--------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------------------------------------------------------
  函数原型   int [[[]{#OLE_LINK33 .anchor}]{#OLE_LINK53 .anchor}]{#OLE_LINK52 .anchor}pthread\_detach (pthread\_t thread);
  参数       描述
  thread     线程句柄（线程标识符）
  返回值     只返回0，总是成功

调用此函数，如果
thread线程没有结束，则将thread线程属性的分离状态设置为detached，当thread线程结束时，系统将自动回收thread线程占用的资源。如果thread线程已经结束，将立刻回收thread线程占用的资源。

使用方法：子线程调用pthread\_detach(pthread\_self())（pthread\_self()返回当前调用线程的线程句柄），或者其他线程调用pthread\_detach(thread\_id)。关于线程属性的分离状态会在线程高级编程里详细介绍。

注意：一旦线程属性的分离状态设置为detached，该线程不能被pthread\_join()函数等待或者重新被设置为detached。

### 线程脱离示例代码

这个程序会初始化2个线程，它们拥有相同的优先级，相同优先级的线程是按照时间片轮转调度。2个线程都会被设置为脱离状态，2个线程循环打印3次信息后自动退出，退出后系统将会自动回收其资源。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 线程1入口函数\*/

static void\* thread1\_entry(void\* parameter)

{

int i;

printf(\"i\'m thread1 and i will detach myself!\\n\");

pthread\_detach(pthread\_self()); /\*线程1脱离自己\*/

for (i = 0;i \< 3;i++) /\* 循环打印3次信息 \*/

{

printf(\"thread1 run count: %d\\n\",i);

sleep(2); /\* 休眠2秒 \*/

}

printf(\"thread1 exited!\\n\");

}

/\* 线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int i;

for (i = 0;i \< 3;i++) /\* 循环打印3次信息 \*/

{

printf(\"thread2 run count: %d\\n\",i);

sleep(2); /\* 休眠2秒 \*/

}

printf(\"thread2 exited!\\n\");

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 创建线程1,属性为默认值，分离状态为默认值joinable,

入口函数是thread1\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\* 创建线程2,属性为默认值，分离状态为默认值joinable,

入口函数是thread2\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

pthread\_detach(tid2); /\* 脱离线程2 \*/

return 0;

}

等待线程结束
------------

  头文件                                                  \#include\<pthread.h\>
  ------------------------------------------------------- ----------------------------------------------------------------------------------------------------------------------------------
  函数原型                                                int [[[]{#OLE_LINK34 .anchor}]{#OLE_LINK46 .anchor}]{#OLE_LINK45 .anchor}pthread\_join (pthread\_t thread, void \*\*value\_ptr);
  参数                                                    描述
  thread                                                  线程句柄
  value\_ptr                                              用户定义的指针，用来存储被等待线程的返回值地址，可由函数pthread\_join()获取
  返回值                                                  描述
  0                                                       成功
  [[]{#OLE_LINK83 .anchor}]{#OLE_LINK82 .anchor}EDEADLK   死锁，线程join自己
  EINVAL                                                  无效参数，Join一个分离状态为detached的线程
  [[]{#OLE_LINK85 .anchor}]{#OLE_LINK84 .anchor}ESRCH     No such process，找不到thread线程

此函数会使调用该函数的线程以阻塞的方式等待线程分离属性为joinable的thread线程运行结束，并获得thread线程的返回值，返回值的地址保存在value\_ptr里，并释放thread线程占用的资源。

pthread\_join()和pthread\_detach()函数功能类似，都是在线程结束后用来回收线程占用的资源。线程不能等待自己结束，thread线程的分离状态必须是joinable，一个线程只对应一次pthread\_join()调用。分离状态为joinable的线程仅当有其他线程对其执行了pthread\_join()后，它所占用的资源才会释放。因此为了避免内存泄漏，所有会结束运行的线程，分离状态要么已设为detached，要么使用pthread\_join()来回收其占用的资源。

### 等待线程结束示例代码

[]{#OLE_LINK40
.anchor}这个程序会初始化2个线程，它们拥有相同的优先级，相同优先级的线程是按照时间片轮转调度。2个线程属性的分离状态为默认值joinable，线程1先开始运行，循环打印3次信息后结束。线程2调用pthread\_join()阻塞等待线程1结束，并回收线程1占用的资源，然后线程2每隔2秒钟会打印一次信息。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块\*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 线程1入口函数 \*/

static void\* thread1\_entry(void\* parameter)

{

int i;

for (int i = 0;i \< 3;i++) /\* 循环打印3次信息 \*/

{

printf(\"thread1 run count: %d\\n\",i);

sleep(2); /\* 休眠2秒 \*/

}

printf(\"thread1 exited!\\n\");

}

/\* 线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int count = 0;

void\* thread1\_return\_value;

/\* 阻塞等待线程1运行结束 \*/

pthread\_join(tid1,NULL);

/\* 线程2打印信息开始输出 \*/

while(1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread2 run count: %d\\n\",count ++);

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 创建线程1,属性为默认值，分离状态为默认值joinable,

入口函数是thread1\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\* 创建线程2,属性为默认值，分离状态为默认值joinable,

入口函数是thread2\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

return 0;

}

线程退出
--------

  头文件       \#include\<pthread.h\>
  ------------ -----------------------------------------------------------------------------------------------------------------------------------------------------------
  函数原型     void [[[]{#OLE_LINK39 .anchor}]{#OLE_LINK55 .anchor}]{#OLE_LINK54 .anchor}pthread\_exit(void \*[[]{#OLE_LINK57 .anchor}]{#OLE_LINK56 .anchor}value\_ptr);
  参数         描述
  value\_ptr   指向线程返回值的指针，如果线程分离状态为joinable,则可由pthread\_join()获取
  返回值       此函数没有返回值

线程调用此函数会终止执行，如同进程调用exit()函数一样，并返回一个指向线程返回值的指针。线程退出由线程自身发起。

-   若线程的分离状态为joinable，线程退出后该线程占用的资源并不会被释放，必须调用pthread\_join()函数释放线程占用的资源。

### 线程退出示例代码

这个程序会初始化2个线程，它们拥有相同的优先级，相同优先级的线程是按照时间片轮转调度。2个线程属性的分离状态为默认值joinable，线程1先开始运行，打印一次信息后休眠2秒，之后打印退出信息然后结束运行。线程2调用pthread\_join()阻塞等待线程1结束，并回收线程1占用的资源，然后线程2每隔2秒钟会打印一次信息。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值核对函数 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 线程1入口函数\*/

static void\* thread1\_entry(void\* parameter)

{

int count = 0;

while(1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread1 run count: %d\\n\",count ++);

sleep(2); /\* 休眠2秒 \*/

printf(\"thread1 will exit!\\n\");

pthread\_exit(0); /\* 线程1主动退出 \*/

}

}

/\* 线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int count = 0;

/\* 阻塞等待线程1运行结束 \*/

pthread\_join(tid1,NULL);

/\* 线程2开始输出打印信息 \*/

while(1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread2 run count: %d\\n\",count ++);

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 创建线程1,属性为默认值，分离状态为默认值joinable,

入口函数是thread1\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\* 创建线程2,属性为默认值，分离状态为默认值joinable,

入口函数是thread2\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

return 0;

}

互斥锁
======

互斥锁又叫相互排斥的信号量，是一种特殊的二值信号量。互斥锁用来保证共享资源的完整性，保证在任一时刻，只能有一个线程访问该共享资源，线程要访问共享资源，必须先拿到互斥锁，访问完成后需要释放互斥锁。嵌入式的共享资源包括内存、IO、SCI、SPI等，如果两个线程同时访问共享资源可能会出现问题，因为一个线程可能在另一个线程修改共享资源的过程中使用了该资源，并认为共享资源没有变化。

互斥锁的操作只有两种上锁或解锁，同一时刻只会有一个线程持有某个互斥锁。当有线程持有它时，互斥量处于闭锁状态，由这个线程获得它的所有权。相反，当这个线程释放它时，将对互斥量进行开锁，失去它的所有权。当一个线程持有互斥量时，其他线程将不能够对它进行解锁或持有它。

对互斥锁的主要操作包括：调用pthread\_mutex\_init()初始化一个互斥锁，调用pthread\_mutex\_destroy()销毁互斥锁，调用pthread\_mutex\_lock()对互斥锁上锁，调用pthread\_mutex\_lock()对互斥锁解锁。

使用互斥锁会导致一个潜在问题是线程优先级翻转。在RT-Thread操作系统中实现的是优先级继承算法。优先级继承是指，提高某个占有某种资源的低优先级线程的优先级，使之与所有等待该资源的线程中优先级最高的那个线程的优先级相等，然后执行，而当这个低优先级线程释放该资源时，优先级重新回到初始设定。因此，继承优先级的线程避免了系统资源被任何中间优先级的线程抢占。有关优先级反转的详细信息请参考RT-Thread编程手册任务间同步及通信一章互斥量一节有详细介绍。

互斥锁控制块
------------

每个互斥锁对应一个互斥锁控制块，包含对互斥锁进行的控制的一些信息。创建互斥锁前必须先定义一个pthread\_mutex\_t类型的变量，
pthread\_mutex\_t是pthread\_mutex的重定义，pthread\_mutex数据结构定义在pthread.h头文件里，数据结构如下：

struct pthread\_mutex

{

pthread\_mutexattr\_t attr; /\* 互斥锁属性 \*/

struct rt\_mutex lock; /\* RT-Thread互斥锁控制块 \*/

};

typedef struct pthread\_mutex pthread\_mutex\_t;

rt\_mutex是RT-Thread
内核里定义的一个数据结构，定义在rtdef.h头文件里，数据结构如下：

struct rt\_mutex

{

struct rt\_ipc\_object parent; /\* 继承自ipc\_object类 \*/

rt\_uint16\_t value; /\* 互斥锁的值 \*/

rt\_uint8\_t original\_priority; /\* 持有线程的原始优先级 \*/

rt\_uint8\_t hold; /\* 互斥锁持有计数 \*/

struct rt\_thread \*owner; /\* 当前拥有互斥锁的线程 \*/

};

typedef struct rt\_mutex\* rt\_mutex\_t; /\*
rt\_mutext\_t为指向互斥锁结构体的指针 \*/

互斥锁初始化
------------

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int [[[[]{#OLE_LINK51             |
|                                   | .anchor}]{#OLE_LINK41             |
|                                   | .anchor}]{#OLE_LINK58             |
|                                   | .anchor}]{#OLE_LINK17             |
|                                   | .anchor}pthread\_mutex\_init(pthr |
|                                   | ead\_mutex\_t                     |
|                                   | \*[[]{#OLE_LINK50                 |
|                                   | .anchor}]{#OLE_LINK42             |
|                                   | .anchor}mutex,                    |
|                                   |                                   |
|                                   | const pthread\_mutexattr\_t       |
|                                   | \*attr);                          |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| mutex                             | 互斥锁句柄，不能为NULL            |
+-----------------------------------+-----------------------------------+
| attr                              | 指向互斥锁属性的指针，若该指针NULL，则使用默认的属性。 |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，mutex为NULL或者初始化失败 |
+-----------------------------------+-----------------------------------+

此函数会初始化mutex互斥锁，并根据attr指向的互斥锁属性对象设置
mutex属性，成功初始化后互斥锁处于未上锁状态，线程可以获取，此函数是对rt\_mutex\_init()函数的封装。

除了调用pthread\_mutex\_init()函数创建一个互斥锁，还可以用宏PTHREAD\_MUTEX\_INITIALIZER来静态初始化互斥锁，方法：pthread\_mutex\_t
mutex =
PTHREAD\_MUTEX\_INITIALIZER（结构体常量），等同于调用pthread\_mutex\_init()时attr指定为NULL。

关于互斥锁属性及相关函数会在线程高级编程一章里有详细介绍，一般情况下采用默认属性就可以。

销毁互斥锁
----------

  头文件     \#include\<pthread.h\>
  ---------- -------------------------------------------------------------------------------------------------------
  函数原型   int [[]{#OLE_LINK62 .anchor}]{#OLE_LINK61 .anchor}pthread\_mutex\_destroy(pthread\_mutex\_t \*mutex);
  参数       描述
  mutex      互斥锁句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，mutex为空或者mutex已经被销毁过
  EBUSY      繁忙，互斥锁正在被使用

此函数会销毁mutex互斥锁。销毁后互斥锁mutex处于未初始化状态。销毁以后互斥锁的属性和控制块参数将不在有效，但可以调用pthread\_mutex\_init()对销毁后的互斥锁重新初始化。但不需要销毁使用宏PTHREAD\_MUTEX\_INITIALIZER静态初始化的互斥锁。

当确定互斥锁没有被锁住，且没有线程阻塞在该互斥锁上，才可以销毁该互斥锁。

阻塞方式对互斥锁上锁
--------------------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------------------------------------------------------------------
  函数原型   int [[[]{#OLE_LINK59 .anchor}]{#OLE_LINK65 .anchor}]{#OLE_LINK64 .anchor}pthread\_mutex\_lock(pthread\_mutex\_t \*mutex);
  参数       描述
  mutex      指向互斥锁控制块的指针，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，mutex为NULL等
  EDEADLK    死锁，互斥锁mutex不为嵌套锁的情况下线程重复调用


此函数对mutex互斥锁上锁，此函数是对rt\_mutex\_take()函数的封装。如果互斥锁mutex还没有被上锁，那么申请该互斥锁的线程将成功对该互斥锁上锁。如果互斥锁mutex已经被当前线程上锁，且互斥锁类型为嵌套锁，则该互斥锁的持有计数加1，当前线程也不会挂起等待（死锁），但线程必须对应相同次数的解锁。如果互斥锁mutex被其他线程上锁持有，则当前线程将被阻塞，一直到其他线程对该互斥锁解锁后，等待该互斥锁的线程将按照先进先出的原则获取互斥锁。

非阻塞方式对互斥锁上锁
----------------------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------
  函数原型   int pthread\_mutex\_trylock(pthread\_mutex\_t \*mutex);
  参数       描述
  mutex      指向互斥锁控制块的指针，不能为NULL
  返回值     参数
  0          成功
  EINVAL     无效参数，mutex为NULL
  EDEADLK    死锁，互斥锁mutex不为嵌套锁的情况下线程重复调用
  EBUSY      繁忙，互斥锁mutex已经被其他线程上锁

[[]{#OLE_LINK25 .anchor}]{#OLE_LINK24
.anchor}pthread\_mutex\_trylock()函数是pthread\_mutex\_lock()函数的非阻塞版本。区别在于如果互斥锁mutex已经被上锁，线程不会被阻塞，而是马上返回错误码。

互斥锁解锁
----------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------
  函数原型   int pthread\_mutex\_unlock(pthread\_mutex\_t \*mutex);
  参数       描述
  mutex      指向互斥锁控制块的指针，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，mutex为NULL等
  EPERM      操作不允许，解锁其他线程持有的类型为检错锁的互斥锁mutex

调用此函数给mutex互斥锁解锁，是对rt\_mutex\_release()函数的封装。当线程完成共享资源的访问后，应尽快释放占有的互斥锁，使得其他线程能及时获取该互斥锁。只有已经拥有互斥锁的线程才能释放它，每释放一次该互斥锁，它的持有计数就减1。当该互斥量的持有计数为零时（即持有线程已经释放所有的持有操作），互斥锁才变为可用，等待在该互斥锁上的线程将按先进先出方式被唤醒。如果线程的运行优先级被互斥锁提升，那么当互斥锁被释放后，线程恢复为持有互斥锁前的优先级。

互斥锁示例代码
--------------

[[]{#OLE_LINK69 .anchor}]{#OLE_LINK68
.anchor}这个程序会初始化2个线程，它们拥有相同的优先级，2个线程都会调用同一个printer()函数输出自己的字符串，printer()函数每次只输出一个字符，之后休眠1秒，调用printer()函数的线程同样也休眠。如果不使用互斥锁，线程1打印了一个字符，休眠后执行线程2，线程2打印一个字符，这样就不能完整的打印线程1和线程2的字符串，打印出的字符串是混乱的。如果使用了互斥锁保护2个线程共享的打印函数printer()，线程1拿到互斥锁后执行printer()打印函数打印一个字符，之后休眠1秒，这是切换到线程2，因为互斥锁已经被线程1上锁，线程2将阻塞，直到线程1的字符串打印完整后主动释放互斥锁后线程2才会被唤醒。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 互斥锁控制块 \*/

static pthread\_mutex\_t mutex;

/\* 线程共享的打印函数 \*/

static void printer(char\* str)

{

while(\*str != 0)

{

putchar(\*str); /\* 输出一个字符 \*/

str++;

sleep(1); /\* 休眠1秒 \*/

}

printf(\"\\n\");

}

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\*线程入口\*/

static void\* thread1\_entry(void\* parameter)

{

char\* str = \"thread1 hello RT-Thread\";

while (1)

{

pthread\_mutex\_lock(&mutex); /\* 互斥锁上锁 \*/

printer(str); /\* 访问共享打印函数 \*/

pthread\_mutex\_unlock(&mutex); /\* 访问完成后解锁 \*/

sleep(2); /\* 休眠2秒 \*/

}

}

static void\* thread2\_entry(void\* parameter)

{

char\* str = \"thread2 hi world\";

while (1)

{

pthread\_mutex\_lock(&mutex); /\* 互斥锁上锁 \*/

printer(str); /\* 访问共享打印函数 \*/

pthread\_mutex\_unlock(&mutex); /\* 访问完成后解锁 \*/

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 初始化一个互斥锁 \*/

pthread\_mutex\_init(&mutex,NULL);

/\*创建线程1,线程入口是thread1\_entry,
属性参数为NULL选择默认值，入口参数是NULL\*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\*创建线程2,线程入口是thread2\_entry,
属性参数为NULL选择默认值，入口参数是NULL\*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

return 0;

}

![](media/image3.png){width="6.0in" height="3.5in"}

![](media/image4.png){width="6.0in" height="3.5in"}

条件变量
========

条件变量其实就是一个信号量，用于线程间同步。条件变量用来阻塞一个线程，当条件满足时向阻塞的线程发送一个条件，阻塞线程就被唤醒，条件变量需要和互斥锁配合使用，互斥锁用来保护共享数据。

条件变量可以用来通知共享数据状态。比如一个处理共享资源队列的线程发现队列为空，则此线程只能等待，直到有一个节点被添加到队列中，添加后在发一个条件变量信号激活等待线程。

条件变量的主要操作包括：调用pthread\_cond\_init()对条件变量初始化，调用pthread\_cond\_destroy()销毁一个条件变量，调用pthread\_cond\_wait()等待一个条件变量，调用pthread\_cond\_signal()发送一个条件变量。

条件变量控制块
--------------

每个条件变量对应一个条件变量控制块，包括对条件变量进行操作的一些信息。初始化一个条件变量前需要先定义一个pthread\_cond\_t条件变量控制块。pthread\_cond\_t是pthread\_cond结构体类型的重定义，定义在pthread.h头文件里。

struct pthread\_cond

{

[]{#OLE_LINK26 .anchor}pthread\_condattr\_t attr; /\* 条件变量属性 \*/

struct rt\_semaphore sem; /\* RT-Thread信号量控制块 \*/

};

typedef struct pthread\_cond pthread\_cond\_t;

rt\_semaphore是RT-Thread内核里定义的一个数据结构，是信号量控制块，定义在rtdef.h头文件里

struct rt\_semaphore

{

struct rt\_ipc\_object parent;/\*继承自ipc\_object类\*/

rt\_uint16\_t value; /\* 信号量的值 \*/

};

初始化条件变量
--------------

  头文件     \#include\<pthread.h\>
  ---------- --------------------------------------------------------------------------------------
  函数原型   int pthread\_cond\_init(pthread\_cond\_t \*cond, const pthread\_condattr\_t \*attr);
  参数       描述
  cond       条件变量句柄，不能为NULL
  attr       指向条件变量属性的指针，若为NULL则使用默认属性值
  返回值     描述
  0          成功
  EINVAL     无效参数，cond或者attr无效等

此函数会初始化cond条件变量，并根据attr指向的条件变量属性设置其属性，此函数是对rt\_sem\_init
()函数的一个封装，基于信号量实现。初始化成功后条件变量处于不可用状态。

还可以用宏PTHREAD\_COND\_INITIALIZER静态初始化一个条件变量，方法：pthread\_cond\_t
cond =
PTHREAD\_COND\_INITIALIZER（结构体常量），等同于调用pthread\_cond\_init()时attr指定为NULL。attr一般设置NULL使用默认值即可，具体会在线程高级编程一章介绍。

销毁条件变量
------------

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------------
  函数原型   int pthread\_cond\_destroy(pthread\_cond\_t \*cond);
  参数       描述
  cond       条件变量句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，cond为NULL
  EBUSY      资源繁忙

此函数会销毁cond条件变量，销毁后cond处于未初始化状态。销毁之后条件变量的属性及控制块参数将不在有效，但可以调用pthread\_cond\_init()或者静态方式重新初始化。

销毁条件变量前需要确定没有线程被阻塞在该条件变量上，也不会等待获取、发信号或者广播。

阻塞方式获取条件变量
--------------------

  头文件     \#include\<pthread.h\>
  ---------- ----------------------------------------------------------------------------------------------------------------------------
  函数原型   int [[]{#OLE_LINK63 .anchor}]{#OLE_LINK60 .anchor}pthread\_cond\_wait(pthread\_cond\_t \*cond, pthread\_mutex\_t \*mutex);
  参数       描述
  cond       条件变量句柄，不能为NULL
  mutex      指向互斥锁控制块的指针，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，获取条件变量失败

此函数会以阻塞方式获取cond条件变量。线程等待条件变量前需要先将mutex
互斥锁锁住，此函数首先判断条件变量是否可用，如果不可用则初始化一个条件变量，之后解锁mutex互斥锁，然后尝试获取一个信号量，当信号量值大于零时，表明信号量可用，线程将获得信号量，也就获得该条件变量，相应的信号量值会减1。如果信号量的值等于零，表明信号量不可用，线程将阻塞直到信号量可用，之后将对mutex互斥锁再次上锁。

指定阻塞时间获取条件变量
------------------------

+-----------+-------------------------------------------------------+
| 头文件    | \#include\<pthread.h\>                                |
+===========+=======================================================+
| 函数原型  | int pthread\_cond\_timedwait(pthread\_cond\_t \*cond, |
|           |                                                       |
|           | pthread\_mutex\_t \*mutex,                            |
|           |                                                       |
|           | const struct timespec \*abstime);                     |
+-----------+-------------------------------------------------------+
| 功能      | 初始化/回收pthread\_cond\_t结构                       |
+-----------+-------------------------------------------------------+
| 参数      | 描述                                                  |
+-----------+-------------------------------------------------------+
| cond      | 条件变量句柄，不能为NULL                              |
+-----------+-------------------------------------------------------+
| mutex     | 互斥锁句柄，不能为NULL                                |
+-----------+-------------------------------------------------------+
| abstime   | 指定的等待时间，单位是操作系统时钟节拍（OS Tick）     |
+-----------+-------------------------------------------------------+
| 返回值    | 描述                                                  |
+-----------+-------------------------------------------------------+
| 0         | 成功                                                  |
+-----------+-------------------------------------------------------+
| ETIMEDOUT | 超时                                                  |
+-----------+-------------------------------------------------------+
| EINVAL    | 无效参数                                              |
+-----------+-------------------------------------------------------+

此函数和pthread\_cond\_wait()函数唯一的差别在于，如果条件变量不可用，线程将被阻塞abstime时长，之后如果条件变量依然不可用，函数将直接返回ETIMEDOUT错误码，线程将会被唤醒进入就绪态。

发送满足条件信号量
------------------

  头文件     \#include\<pthread.h\>
  ---------- -------------------------------------------------------------------------------------------------------------------------------------------------
  函数原型   int [[[[]{#OLE_LINK67 .anchor}]{#OLE_LINK66 .anchor}]{#OLE_LINK30 .anchor}]{#OLE_LINK29 .anchor}pthread\_cond\_signal(pthread\_cond\_t \*cond);
  参数       描述
  cond       指向条件变量控制块的指针，不能为NULL
  返回值     只返回0，总是成功

此函数会发送一个信号且只唤醒一个等待cond条件变量的线程，是对rt\_sem\_release()函数的封装，也就是发送一个信号量。当信号量的值等于零，并且有线程等待这个信号量时，将唤醒等待在该信号量线程队列中的第一个线程，由它获取信号量。否则将把信号量的值加1。

广播
----

  头文件     \#include\<pthread.h\>
  ---------- --------------------------------------------------------
  函数原型   int pthread\_cond\_broadcast(pthread\_cond\_t \*cond);
  参数       描述
  cond       指向条件变量控制块的指针，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数

调用此函数将唤醒所有等待cond条件变量的线程。

条件变量示例代码
----------------

这个程序是一个生产者消费者模型，有一个生产者线程，一个消费者线程，它们拥有相同的优先级。生产者每隔2秒会生产一个数字，放到head指向的链表里面，之后调用pthread\_cond\_signal()给消费者线程发信号，通知消费者线程链表里面有数据。消费者线程会调用pthread\_cond\_wait()等待生产者线程发送信号。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

\#include \<stdlib.h\>

/\* 静态方式初始化一个互斥锁和一个条件变量 \*/

static pthread\_mutex\_t mutex = PTHREAD\_MUTEX\_INITIALIZER;

static pthread\_cond\_t cond = PTHREAD\_COND\_INITIALIZER;

/\* 指向线程控制块的指针 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 生产者生产的结构体数据，存放在链表里 \*/

struct node

{

int n\_number;

struct node\* n\_next;

};

struct node\* head = NULL; /\* 链表头,是共享资源 \*/

/\* 消费者线程入口函数 \*/

static void\* consumer(void\* parameter)

{

struct node\* p\_node = NULL;

pthread\_mutex\_lock(&mutex); /\* 对互斥锁上锁 \*/

while (1)

{

while (head == NULL) /\* 判断链表里是否有元素 \*/

{

pthread\_cond\_wait(&cond,&mutex); /\* 尝试获取条件变量 \*/

}

/\*

pthread\_cond\_wait()会先对mutex解锁，

然后阻塞在等待队列，直到获取条件变量被唤醒，

被唤醒后，该线程会再次对mutex上锁，成功进入临界区。

\*/

p\_node = head; /\* 拿到资源 \*/

head = head-\>n\_next; /\* 头指针指向下一个资源 \*/

/\* 打印输出 \*/

printf(\"consume %d\\n\",p\_node-\>n\_number);

free(p\_node); /\* 拿到资源后释放节点占用的内存 \*/

}

pthread\_mutex\_unlock(&mutex); /\* 释放互斥锁 \*/

return 0;

}

/\* 生产者线程入口函数 \*/

static void\* product(void\* patameter)

{

int count = 0;

struct node \*p\_node;

while(1)

{

/\* 动态分配一块结构体内存 \*/

p\_node = (struct node\*)malloc(sizeof(struct node));

if (p\_node != NULL)

{

p\_node-\>n\_number = count++;

pthread\_mutex\_lock(&mutex); /\* 需要操作head这个临界资源，先加锁 \*/

p\_node-\>n\_next = head;

head = p\_node; /\* 往链表头插入数据 \*/

pthread\_mutex\_unlock(&mutex); /\* 解锁 \*/

printf(\"produce %d\\n\",p\_node-\>n\_number);

pthread\_cond\_signal(&cond); /\* 发信号唤醒一个线程 \*/

sleep(2); /\* 休眠2秒 \*/

}

else

{

printf(\"product malloc node failed!\\n\");

break;

}

}

}

int rt\_application\_init()

{

int result;

/\*
创建生产者线程,属性为默认值，入口函数是product，入口函数参数为NULL\*/

result = pthread\_create(&tid1,NULL,product,NULL);

check\_result(\"product thread created \",result);

/\* 创建消费者线程,属性为默认值，入口函数是consumer，入口函数参数是NULL
\*/

result = pthread\_create(&tid2,NULL,consumer,NULL);

check\_result(\"consumer thread created \",result);

return 0;

}

读写锁
======

读写锁也称为多读者单写者锁。读写锁把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。同一时间只能有一个线程可以占有写模式的读写锁,
但是可以有多个线程同时占有读模式的读写锁。读写锁适合于对数据结构的读次数比写次数多得多的情况，因为读模式锁定时可以共享,
写模式锁定时意味着独占。

读写锁通常是基于互斥锁和条件变量实现的。一个线程可以对一个读写锁进行多次读写锁定，同样必须有对应次数的解锁。

读写锁的主要操作包括：调用pthread\_rwlock\_init()初始化一个读写锁，写线程调用pthread\_rwlock\_wrlock()对读写锁写锁定，读线程调用pthread\_rwlock\_rdlock()对读写锁读锁定，当不需要使用此读写锁时调用pthread\_rwlock\_destroy()销毁读写锁。

读写锁控制块
------------

每个读写锁对应一个读写锁控制块，包括对读写锁进行操作的一些信息。pthread\_rwlock\_t是pthread\_rwlock数据结构的重定义，定义在pthread.h头文件里。在创建一个读写锁之前需要先定义一个pthread\_rwlock\_t类型的数据结构。

struct pthread\_rwlock

{

pthread\_rwlockattr\_t attr; /\* 读写锁属性 \*/

pthread\_mutex\_t rw\_mutex; /\* 互斥锁 \*/

pthread\_cond\_t rw\_condreaders; /\* 条件变量，供读者线程使用 \*/

pthread\_cond\_t rw\_condwriters; /\* 条件变量，供写者线程使用 \*/

int rw\_nwaitreaders; /\* 读者线程等待计数 \*/

int rw\_nwaitwriters; /\* 写者线程等待计数 \*/

int rw\_refcount; /\* 读写锁值，为0值：未上锁,为 -1值: 被写者线程锁定,
大于0值：被读者线程锁定数量 \*/

};

typedef struct pthread\_rwlock pthread\_rwlock\_t; /\* 类型重定义 \*/

读写锁初始化
------------

  头文件     \#include\<pthread.h\>
  ---------- ----------------------------------------------------------------------------------------------------------------------
  函数原型   int []{#OLE_LINK70 .anchor}pthread\_rwlock\_init (pthread\_rwlock\_t \*rwlock, const pthread\_rwlockattr\_t \*attr);
  参数       描述
  rwlock     读写锁句柄，不能为NULL
  attr       指向读写锁属性的指针，RT-Thread不使用此变量
  返回值     描述
  0          成功
  EINVAL     无效参数，rwlock为NULL

此函数会初始化一个rwlock读写锁。此函数使用默认值初始化读写锁控制块的信号量和条件变量，相关计数参数初始为0值。初始化后的读写锁处于未上锁状态。

还可以使用宏PTHREAD\_RWLOCK\_INITIALIZER来静态初始化读写锁，方法：pthread\_rwlock\_t
mutex =
PTHREAD\_RWLOCK\_INITIALIZER（结构体常量），等同于调用pthread\_rwlock\_init
()时attr指定为NULL。[[]{#OLE_LINK87 .anchor}]{#OLE_LINK86
.anchor}attr一般设置NULL使用默认值即可，具体会在线程高级编程一章介绍。

销毁读写锁
----------

  头文件     \#include\<pthread.h\>
  ---------- -------------------------------------------------------------
  函数原型   int pthread\_rwlock\_destroy (pthread\_rwlock\_t \*rwlock);
  参数       描述
  rwlock     读写锁句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，rwlock为NULL等
  EBUSY      繁忙，读写锁目前正在被使用或者有线程等待该读写锁
  EDEADLK    死锁

此函数会销毁一个rwlock读写锁，对应的会销毁读写锁里的互斥锁和条件变量。销毁之后读写锁的属性及控制块参数将不在有效，但可以调用pthread\_rwlock\_init
()或者静态方式重新初始化读写锁。

读写锁读锁定
------------

### 阻塞方式对读写锁读锁定

  []{#_Hlk499043751 .anchor}头文件   \#include\<pthread.h\>
  ---------------------------------- ----------------------------------------------------------------------------------------------------------
  函数原型                           int [[]{#OLE_LINK72 .anchor}]{#OLE_LINK71 .anchor}pthread\_rwlock\_rdlock (pthread\_rwlock\_t \*rwlock);
  参数                               描述
  rwlock                             读写锁句柄，不能为NULL
  返回值                             描述
  0                                  成功
  EINVAL                             无效参数，rwlock为NULL等
  EDEADLK                            死锁

读者线程可以调用此函数来对rwlock读写锁进行读锁定。如果读写锁没有被写锁定并且没有写者线程阻塞在该读写锁上，读写线程将成功获取该读写锁。如果读写锁已经被写锁定，读者线程将会阻塞，直到写锁定该读写锁的线程解锁。

### 非阻塞方式对读写锁读锁定

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------
  函数原型   int pthread\_rwlock\_tryrdlock (pthread\_rwlock\_t \*rwlock);
  参数       描述
  rwlock     读写锁句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，rwlock为NULL
  EBUSY      繁忙，读写锁目前被写锁定或者有写着线程阻塞在该读写锁上
  EDEADLK    死锁

此函数和pthread\_rwlock\_rdlock()函数的不同在于，如果读写锁已经被写锁定，读者线程不会被阻塞，而是返回一个错误码EBUSY。

### 指定阻塞时间对读写锁读锁定

  头文件      \#include\<pthread.h\>
  ----------- --------------------------------------------------------------------------------------------------
  函数原型    int pthread\_rwlock\_timedrdlock (pthread\_rwlock\_t \*rwlock, const struct timespec \*abstime);
  参数        描述
  rwlock      读写锁句柄，不能为NULL
  abstime     指定的等待时间，单位是操作系统时钟节拍（OS　Tick）
  返回值      描述
  0           成功
  EINVAL      无效参数，rwlock无效等
  EDEADLK     死锁
  ETIMEDOUT   超时，阻塞了abstime时长依然没有获取读写锁

此函数和pthread\_rwlock\_rdlock()函数的不同在于，如果读写锁已经被写锁定，读者线程将会阻塞指定的abstime时长，函数将返回错误码ETIMEDOUT，线程将会被唤醒进入就绪态。

读写锁写锁定
------------

### 阻塞方式写锁定读写锁

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------------------
  函数原型   int pthread\_rwlock\_wrlock (pthread\_rwlock\_t \*rwlock);
  参数       描述
  rwlock     读写锁句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，rwlock为NULL等
  EDEADLK    死锁

写者线程调用此函数对rwlock读写锁进行写锁定。写锁定读写锁类似互斥量，同一时刻只能有一个线程写锁定读写锁。如果没有线程锁定该读写锁，即读写锁值为0，调用此函数的写者线程将会写锁定读写锁，其他线程此时都不能获取读写锁，如果已经有线程锁定该读写锁，即读写锁值不为0，则写线程将被阻塞，直到读写锁解锁。

### 非阻塞方式写锁定读写锁

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------
  函数原型   int pthread\_rwlock\_trywrlock (pthread\_rwlock\_t \*rwlock);
  参数       描述
  rwlock     读写锁句柄，不能为NULL
  返回值     描述
  0          成功
  EINVAL     无效参数，rwlock为NULL
  EBUSY      繁忙，读写锁目前被写锁定或者有写着线程阻塞在该读写锁上
  EDEADLK    死锁

此函数和pthread\_rwlock\_wrlock()函数唯一的不同在于，如果已经有线程锁定该读写锁，即读写锁值不为0，则调用该函数的写者线程会直接返回一个错误代码，线程不会被阻塞。

### 指定阻塞时长写锁定读写锁

+-----------+----------------------------------------------------------------+
| 头文件    | \#include\<pthread.h\>                                         |
+===========+================================================================+
| 函数原型  | int pthread\_rwlock\_timedwrlock (pthread\_rwlock\_t \*rwlock, |
|           |                                                                |
|           | const struct timespec \*abstime);                              |
+-----------+----------------------------------------------------------------+
| 参数      | 描述                                                           |
+-----------+----------------------------------------------------------------+
| rwlock    | 读写锁句柄，不能为NULL                                         |
+-----------+----------------------------------------------------------------+
| abstime   | 指定的等待时间，单位是操作系统时钟节拍（OS Tick）              |
+-----------+----------------------------------------------------------------+
| 返回值    | 描述                                                           |
+-----------+----------------------------------------------------------------+
| 0         | 成功                                                           |
+-----------+----------------------------------------------------------------+
| EINVAL    | 无效参数，rwlock无效等                                         |
+-----------+----------------------------------------------------------------+
| EDEADLK   | 死锁                                                           |
+-----------+----------------------------------------------------------------+
| ETIMEDOUT | 超时，阻塞了abstime时长依然没有获取读写锁                      |
+-----------+----------------------------------------------------------------+

此函数和pthread\_rwlock\_wrlock()函数唯一的不同在于，如果已经有线程锁定该读写锁，即读写锁值不为0，调用线程阻塞指定的abstime时长后，函数将返回错误码ETIMEDOUT，线程将会被唤醒进入就绪态。

读写锁解锁
----------

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------------------------------
  函数原型   int []{#OLE_LINK73 .anchor}pthread\_rwlock\_unlock (pthread\_rwlock\_t \*rwlock);
  参数       描述
  rwlock     读写锁句柄，不能为NULL

此函数可以对rwlock读写锁解锁。线程对同一个读写锁加锁多次，必须有同样次数的解锁，若解锁后有多个线程等待对读写锁进行锁定，系统将按照先进先出的规则激活等待的线程。

读写锁示例代码
--------------

这个程序有2个读者线程，一个写着线程。2个读者线程先对读写锁读锁定，之后休眠2秒，这是其他的读者线程还是可以对该读写锁读锁定，然后读取共享数据。

\#include \<pthread.h\>

\#include \<sched.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t reader1;

static pthread\_t reader2;

static pthread\_t writer1;

/\* 共享数据book \*/

static int book = 0;

/\* 读写锁 \*/

static pthread\_rwlock\_t rwlock;

/\* 函数结果检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\*线程入口\*/

static void\* reader1\_entry(void\* parameter)

{

while (1)

{

pthread\_rwlock\_rdlock(&rwlock); /\* 尝试读锁定该读写锁 \*/

printf(\"reader1 read book value is %d\\n\",book);

sleep(2); /\* 线程休眠2秒，切换到其他线程运行 \*/

pthread\_rwlock\_unlock(&rwlock); /\* 线程运行后对读写锁解锁 \*/

}

}

static void\* reader2\_entry(void\* parameter)

{

while (1)

{

pthread\_rwlock\_rdlock(&rwlock); /\* 尝试读锁定该读写锁 \*/

printf(\"reader2 read book value is %d\\n\",book);

sleep(2); /\* 线程休眠2秒，切换到其他线程运行 \*/

pthread\_rwlock\_unlock(&rwlock); /\* 线程运行后对读写锁解锁 \*/

}

}

static void\* writer1\_entry(void\* parameter)

{

while (1)

{

pthread\_rwlock\_wrlock(&rwlock); /\* 尝试写锁定该读写锁 \*/

book++;

printf(\"writer1 write book value is %d\\n\",book);

pthread\_rwlock\_unlock(&rwlock); /\* 对读写锁解锁 \*/

sleep(2); /\* 线程休眠2秒，切换到其他线程运行 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 默认属性初始化读写锁 \*/

pthread\_rwlock\_init(&rwlock,NULL);

/\*创建reader1线程,线程入口是reader1\_entry,
线程属性为默认值，入口参数为NULL\*/

result = pthread\_create(&reader1,NULL,reader1\_entry,NULL);

check\_result(\"reader1 created\",result);

/\*创建reader2线程,线程入口是reader2\_entry,
线程属性为默认值，入口参数为NULL\*/

result = pthread\_create(&reader2,NULL,reader2\_entry,NULL);

check\_result(\"reader2 created\",result);

/\*创建writer1线程,线程入口是writer1\_entry,
线程属性为，入口参数为NULL\*/

result = pthread\_create(&writer1,NULL,writer1\_entry,NULL);

check\_result(\"writer1 created\",result);

return 0;

}

屏障
====

屏障是多线程同步的一种方法。barrier意为屏障或者栏杆，把先后到达的多个线程挡在同一栏杆前，直到所有线程到齐，然后撤下栏杆同时放行。先到达的线程将会阻塞，等到所有调用pthread\_barrier\_wait()函数的线程（数量等于屏障初始化时指定的count）都到达后，这些线程才会由阻塞状态进入就绪状态再次参与系统调度。

屏障是基于条件变量和互斥锁实现的。主要操作包括：调用pthread\_barrier\_init()初始化一个屏障，其他线程调用pthread\_barrier\_wait()，所有线程到期后线程唤醒进入准备状态，屏障不在使用调用pthread\_barrier\_destroy()销毁一个屏障。

屏障控制块
----------

创建一个屏障前需要先定义一个pthread\_barrier\_t
屏障控制块。pthread\_barrier\_t是pthread\_barrier结构体类型的重定义，定义在pthread.h头文件里。

struct pthread\_barrier

{

int count; /\*指定的等待线程个数\*/

pthread\_cond\_t cond; /\* 条件变量 \*/

pthread\_mutex\_t mutex; /\* 互斥锁 \*/

};

typedef struct pthread\_barrier pthread\_barrier\_t;

创建屏障
--------

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int [[]{#OLE_LINK75               |
|                                   | .anchor}]{#OLE_LINK74             |
|                                   | .anchor}pthread\_barrier\_init(pt |
|                                   | hread\_barrier\_t                 |
|                                   | \*barrier,                        |
|                                   |                                   |
|                                   | const pthread\_barrierattr\_t     |
|                                   | \*attr,                           |
|                                   |                                   |
|                                   | unsigned count);                  |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| attr                              | 指向屏障属性的指针，传入NULL，则使用默认值，非NULL必须使用 |
|                                   | PTHREAD\_PROCESS\_PRIVATE         |
+-----------------------------------+-----------------------------------+
| barrier                           | 屏障句柄                          |
+-----------------------------------+-----------------------------------+
| count                             | 指定的等待线程个数                |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，barrier或者attr无效     |
+-----------------------------------+-----------------------------------+

此函数会创建一个barrier屏障，并根据默认的参数对屏障控制块的条件变量和互斥锁初始化，初始化后指定的等待线程个数为count个，必须对应count个线程调用pthread\_barrier\_wait()。attr一般设置NULL使用默认值即可，具体会在线程高级编程一章介绍。s

销毁屏障
--------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------
  函数原型   int pthread\_barrier\_destroy(pthread\_barrier\_t \*barrier);
  参数       描述
  barrier    指向栏杆控制卡的指针
  返回值     描述
  0          成功
  EINVAL     无效参数

此函数会销毁一个barrier屏障。销毁之后屏障的属性及控制块参数将不在有效，但可以调用pthread\_barrier\_init
()重新初始化。

等待屏障
--------

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------------------------------------------------------------------------------
  函数原型   int [[[]{#OLE_LINK76 .anchor}]{#OLE_LINK38 .anchor}]{#OLE_LINK37 .anchor}pthread\_barrier\_wait(pthread\_barrier\_t \*barrier);
  barrier    指向栏杆控制卡的指针
  返回值     描述
  0          成功
  EINVAL     无效参数，barrier为NULL等
  EDEADLK    死锁

此函数同步等待在barrier前的线程，由每个线程主动调用，若屏障等待线程个数count不为0，count将减1，若减1后count为0，表明所有线程都已经到达栏杆前，所有到达的线程将被唤醒重新进入就绪状态，参与系统调度。若减一后count不为0，表明还有线程没有到达屏障，调用的线程将阻塞直到所有线程到达屏障。

屏障示例代码
------------

此程序会创建3个线程，初始化一个屏障，屏障等待线程数初始化为3。3个线程都会调用pthread\_barrier\_wait()等待在屏障前，当3个线程都到齐后，3个线程进入就绪态，之后会每隔2秒打印输出计数信息。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

static pthread\_t tid3;

/\* 屏障控制块 \*/

static pthread\_barrier\_t barrier;

/\* 函数返回值检查函数 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\*线程1入口函数\*/

static void\* thread1\_entry(void\* parameter)

{

int count = 0;

printf(\"thread1 have arrived the barrier!\\n\");

pthread\_barrier\_wait(&barrier); /\* 到达屏障，并等待其他线程到达 \*/

while (1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread1 count: %d\\n\",count ++);

/\* 休眠2秒\*/

sleep(2);

}

}

/\*线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int count = 0;

printf(\"thread2 have arrived the barrier!\\n\");

pthread\_barrier\_wait(&barrier);

while (1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread2 count: %d\\n\",count ++);

/\* 休眠2秒\*/

sleep(2);

}

}

/\* 线程3入口函数 \*/

static void\* thread3\_entry(void\* parameter)

{

int count = 0;

printf(\"thread3 have arrived the barrier!\\n\");

pthread\_barrier\_wait(&barrier);

while (1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread3 count: %d\\n\",count ++);

/\* 休眠2秒\*/

sleep(2);

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

pthread\_barrier\_init(&barrier,NULL,3);

/\*创建线程1,线程入口是thread1\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\*创建线程2,线程入口是thread2\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

/\*创建线程3,线程入口是thread3\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid3,NULL,thread3\_entry,NULL);

check\_result(\"thread3 created\",result);

}

信号量
======

信号量可以用于进程与进程之间，或者进程内线程之间的通信。每个信号量都有一个不会小于0的信号量值，对应信号量的可用数量。调用sem\_init()或者sem\_open()给信号量值赋初值，调用sem\_post()函数可以让信号量值加1，调用sem\_wait()可以让信号量值减1，如果当前信号量为0，调用sem\_wait()的线程被挂起在该信号量的等待队列上，直到信号量值大于0，处于可用状态。

根据信号量的值（代表可用资源的数目）的不同，POSIX信号量可以分为：

二值信号量：信号量的值只有0和1，初始值指定为1。这和互斥锁一样，若资源被锁住，信号量的值为0，若资源可用，则信号量的值为1。相当于只有一把钥匙，线程拿到钥匙后，完成了对共享资源的访问后需要解锁，把钥匙再放回去，给其他需要此钥匙的线程使用。使用方法和互斥锁一样，等待信号量函数必须和发送信号量函数成对使用，不能单独使用，必须先等待后发送。

计数信号量：信号量的值在0到一个大于1的限制值（POSIX指出系统的最大限制值至少要为32767）。该计数表示可用信号量个数。此时，发送信号量函数可以被单独调用发送信号量，相当于有多把钥匙，线程拿到一把钥匙就消耗了一把，使用过的钥匙不必在放回去。

POSIX信号量又分为有名信号量和无名信号量：

有名信号量：其值保存在文件中，一般用于进程间同步或互斥。

无名信号量：其值保存在内存中，一般用于线程间同步或互斥。

RT-Thread操作系统的POSIX信号量主要是基于RT-Thread内核信号量的一个封装，主要还是用于系统内线程间的通讯。使用方式和RT-Thread内核的信号量差不多。

信号量控制块
------------

每个信号量对应一个信号量控制块，创建一个信号量前需要先定义一个sem\_t
信号量控制块。sem\_t是posix\_sem结构体类型的重定义，定义在semaphore.h头文件里。

struct posix\_sem

{

rt\_uint16\_t refcount;

rt\_uint8\_t unlinked;

rt\_uint8\_t unamed;

[[]{#OLE_LINK8 .anchor}]{#OLE_LINK5 .anchor}rt\_sem\_t sem; /\*
RT-Thread 信号量 \*/

struct posix\_sem\* next; /\* 指向下一个信号量控制块 \*/

};

typedef struct posix\_sem sem\_t;

rt\_sem\_t是RT-Thread信号量控制块，定义在rtdef.h头文件里。

struct rt\_semaphore

{

struct rt\_ipc\_object parent;/\*继承自ipc\_object类\*/

rt\_uint16\_t value; /\* 信号量的值 \*/

};

/\* rt\_sem\_t是指向semaphore结构体的指针类型 \*/

typedef struct rt\_semaphore\* rt\_sem\_t;

无名信号量
----------

无名信号量的值保存在内存中，一般用于线程间同步或互斥。在使用之前，必须先调用sem\_init()初始化。

### 无名信号量初始化

  头文件     \#include\<semaphore.h\>
  ---------- --------------------------------------------------------------------------------------
  函数原型   int []{#OLE_LINK77 .anchor}sem\_init(sem\_t \*sem, int pshared, unsigned int value);
  参数       描述
  sem        信号量句柄
  value      信号量初始值，表示信号量资源的可用数量
  pshared    RT-Thread未实现参数
  返回值     执行成功则返回0，否则返回-1

此函数初始化一个无名信号量sem，根据给定的或默认的参数对信号量相关数据结构进行初始化，并把信号量放入信号量链表里。初始化后信号量值为给定的初始值value。此函数是对rt\_sem\_create()函数的封装。

### 销毁无名信号量

  头文件     \#include\<semaphore.h\>
  ---------- ---------------------------------
  函数原型   int sem\_destroy(sem\_t \*sem);
  参数       描述
  sem        信号量句柄
  返回值     成功则返回0，否则返回-1

此函数会销毁一个无名信号量sem，并释放信号量占用的资源。

有名信号量
----------

有名信号量，其值保存在文件中，一般用于进程间同步或互斥。两个进程可以操作相同名称的有名信号量。RT-Thread操作系统中的有名信号量实现和无名信号量差不多，都是设计用于线程间的通信，使用方法也类似。

### 创建或打开有名信号量

  头文件     \#include\<semaphore.h\>
  ---------- -------------------------------------------------------------------------------------------------------
  函数原型   sem\_t \*sem\_open(const char \*[[]{#OLE_LINK28 .anchor}]{#OLE_LINK27 .anchor}name, int oflag, \...);
  参数       描述
  name       信号量名称
  oflag      信号量的打开方式
  返回值     成功则返回信号量句柄，否则返回NULL

此函数会根据信号量名字name创建一个新的信号量或者打开一个已经存在的信号量。Oflag的可选值有0、O\_CREAT或O\_CREAT\|O\_EXCL。如果Oflag设置为O\_CREAT则会创建一个新的信号量。如果Oflag设置O\_CREAT\|O\_EXCL，如果信号量已经存在则会返回NULL，如果不存在则会创建一个新的信号量。如果Oflag设置为0，信号量不存在则会返回NULL。

### 分离有名信号量

  头文件     \#include\<semaphore.h\>
  ---------- -------------------------------------
  函数原型   int sem\_unlink(const char \*name);
  参数       描述
  name       信号量名称
  返回值     成功,返回0，若信号量不存在则返回-1

[[]{#OLE_LINK13 .anchor}]{#OLE_LINK12
.anchor}此函数会根据信号量名称name查找该信号量，若信号量存在，则将该信号量标记为分离状态。之后检查引用计数，若值为0，则立即删除信号量，若值不为0，则等到所有持有该信号量的线程关闭信号量之后才会删除。

### 关闭有名信号量

  头文件     \#include\<semaphore.h\>
  ---------- -----------------------------------------------------------------------------
  函数原型   int [[]{#OLE_LINK32 .anchor}]{#OLE_LINK31 .anchor}sem\_close(sem\_t \*sem);
  参数       描述
  sem        信号量句柄，不能为NULL
  返回值     成功返回0，否则返回-1

当一个线程终止时，会对其占用的信号量执行此关闭操作。不论线程是自愿终止还是非自愿终止都会执行这个关闭操作，相当于是信号量的持有计数减1。若减1后持有计数为0且信号量已经处于分离状态，则会删除sem信号量并释放其占有的资源。

获取信号量值
------------

  头文件     \#include\<semaphore.h\>
  ---------- ----------------------------------------------
  函数原型   int sem\_getvalue(sem\_t \*sem, int \*sval);
  参数       描述
  sem        信号量句柄，不能为NULL
  sval       保存获取的信号量值地址,不能为NULL
  返回值     成功则返回0，否则返回-1

此函数可以获取sem信号量的值，并保存在sval指向的内存里，可以知道信号量的资源数量。

阻塞方式获取信号量
------------------

  头文件     \#include\<semaphore.h\>
  ---------- ---------------------------------------------------------------------------
  函数原型   int [[]{#OLE_LINK16 .anchor}]{#OLE_LINK9 .anchor}sem\_wait(sem\_t \*sem);
  参数       描述
  sem        信号量句柄
  返回值     成功则返回0，否则返回-1

线程调用此函数获取信号量，是rt\_sem\_take(sem，RT\_WAITING\_FOREVER)函数的封装。若信号量值大于零，表明信号量可用，线程获得信号量，信号量值减1。若信号量值等于0，表明信号量不可用，线程阻塞进入挂起状态，并按照先进先出的方式排队等待，直到信号量可用。

非阻塞方式获取信号量
--------------------

  头文件     \#include\<semaphore.h\>
  ---------- ---------------------------------
  函数原型   int sem\_trywait(sem\_t \*sem);
  参数       描述
  sem        信号量句柄
  返回值     成功则返回0，否则返回-1

此函数是sem\_wait()函数的非阻塞版，是rt\_sem\_take(sem,
0)函数的封装。当信号量不可用时，线程不会阻塞挂起，而是直接返回。

指定阻塞时间获取信号量
----------------------

  头文件         \#include\<semaphore.h\>
  -------------- -------------------------------------------------------------------------
  函数原型       int sem\_timedwait(sem\_t \*sem, const struct timespec \*abs\_timeout);
  参数           描述
  sem            指向信号量控制块的指针
  abs\_timeout   指定的等待时间，单位是操作系统时钟节拍（OS Tick）
  返回值         成功则返回0，否则返回-1

此函数和sem\_wait()函数的区别在于，若信号量不可用，线程将阻塞abs\_timeout时长，函数返回-1，线程将被唤醒由阻塞态进入就绪态。

发送信号量
----------

  头文件     \#include\<semaphore.h\>
  ---------- ------------------------------
  函数原型   int sem\_post(sem\_t \*sem);
  参数       描述
  sem        信号量句柄
  返回值     成功则返回0，否则返回-1

此函数将释放一个sem信号量，是rt\_sem\_release()函数的封装。若等待该信号量的线程队列不为空，表明有线程在等待该信号量，第一个等待该信号量的线程将由挂起状态切换到就绪状态，等待系统调度。若没有线程等待该信号量，该信号量值将加1。

无名信号量使用示例代码
----------------------

信号量使用的典型案例是生产者消费者模型。一个生产者线程和一个消费者线程对同一块内存进行操作，生产者往共享内存填充数据，消费者从共享内存读取数据。

此程序会创建2个线程，2个信号量，一个信号量表示共享数据为空状态，一个信号量表示共享数据不为空状态，一个互斥锁用于保护共享资源。生产者线程生产好数据后会给消费者发送一个full\_sem信号量，通知消费者线程有数据可用，休眠2秒后会等待消费者线程发送的empty\_sem信号量。消费者线程等到生产者发送的full\_sem后会处理共享数据，处理完后会给生产者线程发送empty\_sem信号量。程序会这样一直循环。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

\#include \<stdlib.h\>

\#include \<semaphore.h\>

/\* 静态方式初始化一个互斥锁用于保护共享资源\*/

static pthread\_mutex\_t mutex = PTHREAD\_MUTEX\_INITIALIZER;

/\* 2个信号量控制块，一个表示资源空信号，一个表示资源满信号 \*/

static sem\_t empty\_sem,full\_sem;

/\* 指向线程控制块的指针 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 生产者生产的结构体数据，存放在链表里 \*/

struct node

{

int n\_number;

struct node\* n\_next;

};

struct node\* head = NULL; /\* 链表头,是共享资源 \*/

/\* 消费者线程入口函数 \*/

static void\* consumer(void\* parameter)

{

struct node\* p\_node = NULL;

while (1)

{

sem\_wait(&full\_sem);

pthread\_mutex\_lock(&mutex); /\* 对互斥锁上锁, \*/

while (head != NULL) /\* 判断链表里是否有元素 \*/

{

p\_node = head; /\* 拿到资源 \*/

head = head-\>n\_next; /\* 头指针指向下一个资源 \*/

/\* 打印输出 \*/

printf(\"consume %d\\n\",p\_node-\>n\_number);

free(p\_node); /\* 拿到资源后释放节点占用的内存 \*/

}

pthread\_mutex\_unlock(&mutex); /\* 临界区数据操作完毕，释放互斥锁 \*/

sem\_post(&empty\_sem); /\* 发送一个空信号量给生产者 \*/

}

}

/\* 生产者线程入口函数 \*/

static void\* product(void\* patameter)

{

int count = 0;

struct node \*p\_node;

while(1)

{

/\* 动态分配一块结构体内存 \*/

p\_node = (struct node\*)malloc(sizeof(struct node));

if (p\_node != NULL)

{

p\_node-\>n\_number = count++;

pthread\_mutex\_lock(&mutex); /\* 需要操作head这个临界资源，先加锁 \*/

p\_node-\>n\_next = head;

head = p\_node; /\* 往链表头插入数据 \*/

pthread\_mutex\_unlock(&mutex); /\* 解锁 \*/

printf(\"produce %d\\n\",p\_node-\>n\_number);

sem\_post(&full\_sem); /\* 发送一个满信号量给消费者 \*/

}

else

{

printf(\"product malloc node failed!\\n\");

break;

}

sleep(2); /\* 休眠2秒 \*/

sem\_wait(&empty\_sem); /\* 等待消费者发送空信号量 \*/

}

}

int rt\_application\_init()

{

int result;

sem\_init(&empty\_sem,NULL,0);

sem\_init(&full\_sem,NULL,0);

/\*
创建生产者线程,属性为默认值，入口函数是product，入口函数参数为NULL\*/

result = pthread\_create(&tid1,NULL,product,NULL);

check\_result(\"product thread created \",result);

/\* 创建消费者线程,属性为默认值，入口函数是consumer，入口函数参数是NULL
\*/

result = pthread\_create(&tid2,NULL,consumer,NULL);

check\_result(\"consumer thread created \",result);

return 0;

}

消息队列
========

消息队列是另一种常用的线程间通讯方式，它能够接收来自线程或中断服务例程中不固定长度的消息，并把消息缓存在自己的内存空间中。其他线程也能够从消息队列中读取相应的消息，而当消息队列是空的时候，可以挂起读取线程。当有新的消息到达时，挂起的线程将被唤醒以接收并处理消息。

消息队列主要操作包括：通过函数mq\_open()创建或者打开，调用mq\_send()发送一条消息到消息队列，调用mq\_receive()从消息队列获取一条消息，当消息队列不在使用时，可以调用mq\_unlink()删除消息队列。

POSIX消息队列主要用于进程间通信，RT-Thread操作系统的POSIX消息队列主要是基于RT-Thread内核消息队列的一个封装，主要还是用于系统内线程间的通讯。使用方式和RT-Thread内核的消息队列差不多。

消息队列控制块
--------------

每个消息队列对应一个消息队列控制块，创建消息队列前需要先定义一个消息队列控制块。消息队列控制块定义在mqueue.h头文件里。

struct mqdes

{

rt\_uint16\_t refcount; /\* 引用计数 \*/

rt\_uint16\_t unlinked; /\*
消息队列的分离状态，值为1表示消息队列已经分离 \*/

rt\_mq\_t mq; /\* RT-Thread 消息队列控制块 \*/

struct mqdes\* next; /\* 指向下一个消息队列控制块 \*/

};

typedef struct mqdes\* mqd\_t; /\* 消息队列控制块指针类型重定义 \*/

创建或打开消息队列
------------------

  头文件     \#include\<mqueue.h\>
  ---------- -----------------------------------------------------------------------------
  函数原型   mqd\_t mq\_open(const char \*[]{#OLE_LINK11 .anchor}name, int oflag, \...);
  参数       描述
  name       消息队列名称
  oflag      消息队列打开方式
  返回值     成功则返回消息队列句柄，否则返回NULL

此函数会根据消息队列的名字name创建一个新的消息队列或者打开一个已经存在的消息队列。Oflag的可选值有0、O\_CREAT或O\_CREAT\|O\_EXCL。如果Oflag设置为O\_CREAT则会创建一个新的消息队列。如果Oflag设置O\_CREAT\|O\_EXCL，如果消息队列已经存在则会返回NULL，如果不存在则会创建一个新的消息队列。如果Oflag设置为0，消息队列不存在则会返回NULL。

分离消息队列
------------

  头文件     \#include\<mqueue.h\>
  ---------- --------------------------------------
  函数原型   int mq\_unlink(const char \*name);
  参数       描述
  name       消息队列的名称
  返回值     成功,返回0，若消息队列不存在则返回-1

此函数会根据消息队列名称name查找消息队列，若找到，则将消息队列置为分离状态，之后若持有计数为0，则删除消息队列，并释放消息队列占有的资源。[]{#_Toc499916172
.anchor}

关闭消息队列
------------

  头文件     \#include\<mqueue.h\>
  ---------- ------------------------------
  函数原型   int mq\_close(mqd\_t mqdes);
  参数       描述
  mqdes      消息队列句柄
  返回值     成功返回0，否则返回-1

当一个线程终止时，会对其占用的消息队列执行此关闭操作。不论线程是自愿终止还是非自愿终止都会执行这个关闭操作，相当于是消息队列的持有计数减1，若减1后持有计数为0，且消息队列处于分离状态，则会删除mqdes消息队列并释放其占有的资源。

阻塞方式发送消息
----------------

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<mqueue.h\>             |
+===================================+===================================+
| 函数原型                          | int [[]{#OLE_LINK19               |
|                                   | .anchor}]{#OLE_LINK18             |
|                                   | .anchor}mq\_send(mqd\_t mqdes,    |
|                                   |                                   |
|                                   | const char \*msg\_ptr,            |
|                                   |                                   |
|                                   | size\_t msg\_len,                 |
|                                   |                                   |
|                                   | unsigned msg\_prio);              |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| []{#_Hlk499912111 .anchor}mqdes   | 消息队列句柄,不能为NULL           |
+-----------------------------------+-----------------------------------+
| msg\_ptr                          | 指向要发送的消息的指针，不能为NULL |
+-----------------------------------+-----------------------------------+
| msg\_len                          | 发送的消息的长度                  |
+-----------------------------------+-----------------------------------+
| msg\_prio                         | Rt-thread未实现参数               |
+-----------------------------------+-----------------------------------+
| 返回值                            | 成功返回0，否则返回-1             |
+-----------------------------------+-----------------------------------+

此函数用来向mqdes消息队列发送一条消息，是rt\_mq\_send()函数的封装。此函数把msg\_ptr指向的消息添加到mqdes消息队列中，发送的消息长度msg\_len必须小于或者等于创建消息队列时设置的最大消息长度。

如果消息队列已经满，即消息队列中的消息数量等于最大消息数，发送消息的的线程或者中断程序会收到一个错误码（-RT\_EFULL）。

指定阻塞时间发送消息
--------------------

+-----------+----------------------------------------+
| 头文件    | \#include\<mqueue.h\>                  |
+===========+========================================+
| 函数原型  | int mq\_timedsend(mqd\_t mqdes,        |
|           |                                        |
|           | const char \*msg\_ptr,                 |
|           |                                        |
|           | size\_t msg\_len,                      |
|           |                                        |
|           | unsigned msg\_prio,                    |
|           |                                        |
|           | const struct timespec \*abs\_timeout); |
+-----------+----------------------------------------+
| 参数      | 描述                                   |
+-----------+----------------------------------------+
| mqdes     | 消息队列句柄,不能为NULL                |
+-----------+----------------------------------------+
| msg\_ptr  | 指向要发送的消息的指针，不能为NULL     |
+-----------+----------------------------------------+
| msg\_len  | 发送的消息的长度                       |
+-----------+----------------------------------------+
| msg\_prio | Rt-thread未实现参数                    |
+-----------+----------------------------------------+
| 返回值    | 成功返回0，否则返回-1                  |
+-----------+----------------------------------------+

目前RT-Thread不支持指定阻塞时间发送消息，但是函数接口已经实现，相当于调用mq\_send()。

阻塞方式接受消息
----------------

+-----------+------------------------------------+
| 头文件    | \#include\<mqueue.h\>              |
+===========+====================================+
| 函数原型  | ssize\_t mq\_receive(mqd\_t mqdes, |
|           |                                    |
|           | char \*msg\_ptr,                   |
|           |                                    |
|           | size\_t msg\_len,                  |
|           |                                    |
|           | unsigned \*msg\_prio);             |
+-----------+------------------------------------+
| 参数      | 描述                               |
+-----------+------------------------------------+
| mqdes     | 消息队列句柄,不能为NULL            |
+-----------+------------------------------------+
| msg\_ptr  | 存放接受消息的地址，不能为NULL     |
+-----------+------------------------------------+
| msg\_len  | 存放接受消息的内存大小             |
+-----------+------------------------------------+
| msg\_prio | Rt-thread未实现参数                |
+-----------+------------------------------------+
| 返回值    | 成功则返回消息长度，否则返回-1     |
+-----------+------------------------------------+

此函数会把mqdes消息队列里面最老的消息移除消息队列，并把消息放到msg\_ptr指向的内存里。

如果消息队列为空，调用mq\_receive()函数的线程将会阻塞，直到消息队列中消息可用。

指定阻塞时间接受消息
--------------------

+-----------------------------------+-----------------------------------+
| []{#_Hlk499911605 .anchor}头文件  | \#include\<mqueue.h\>             |
+===================================+===================================+
| 函数原型                          | ssize\_t mq\_timedreceive(mqd\_t  |
|                                   | mqdes,                            |
|                                   |                                   |
|                                   | char \*msg\_ptr,                  |
|                                   |                                   |
|                                   | size\_t msg\_len,                 |
|                                   |                                   |
|                                   | unsigned \*msg\_prio,             |
|                                   |                                   |
|                                   | const struct timespec             |
|                                   | \*abs\_timeout);                  |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| mqdes                             | 消息队列句柄,不能为NULL           |
+-----------------------------------+-----------------------------------+
| msg\_ptr                          | 存放接受消息的地址，不能为NULL    |
+-----------------------------------+-----------------------------------+
| msg\_len                          | 存放接受消息的内存大小            |
+-----------------------------------+-----------------------------------+
| msg\_prio                         | Rt-thread未实现参数               |
+-----------------------------------+-----------------------------------+
| abs\_timeout                      | 指定的等待时间，单位是操作系统时钟节拍（OS |
|                                   |                                   |
|                                   | Tick）                            |
+-----------------------------------+-----------------------------------+
| 返回值                            | 成功则返回消息长度，否则返回-1    |
+-----------------------------------+-----------------------------------+

此函数和mq\_receive
()函数的区别在于，若消息队列为空，线程将阻塞abs\_timeout时长，函数直接返回-1，线程将被唤醒由阻塞态进入就绪态。

消息队列示例代码
----------------

这个程序会创建3个线程，线程2从消息队列接受消息，线程2和线程3往消息队列发送消息。

\#include \<mqueue.h\>

\#include \<stdio.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

static pthread\_t tid3;

/\* 消息队列句柄 \*/

static mqd\_t mqueue;

/\* 函数返回值检查函数 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\*线程1入口函数\*/

static void\* thread1\_entry(void\* parameter)

{

char buf\[128\];

int result;

while (1)

{

/\* 从消息队列中接收消息 \*/

result = mq\_receive(mqueue, &buf\[0\], sizeof(buf), 0);

if (result != -1)

{

/\* 输出内容 \*/

printf(\"thread1 recv \[%s\]\\n\", buf);

}

/\* 休眠1秒\*/

// sleep(1);

}

}

/\*线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int i, result;

char buf\[\] = \"message2 No.x\";

while (1)

{

for (i = 0; i \< 10; i++)

{

buf\[sizeof(buf) - 2\] = \'0\' + i;

printf(\"thread2 send \[%s\]\\n\", buf);

/\* 发送消息到消息队列中 \*/

result = mq\_send(mqueue, &buf\[0\], sizeof(buf), 0);

if ( result == -1)

{

/\* 消息队列满， 延迟1s时间 \*/

printf(\"thread2:message queue is full, delay 1s\\n\");

sleep(1);

}

}

/\* 休眠2秒\*/

sleep(2);

}

}

/\* 线程3入口函数 \*/

static void\* thread3\_entry(void\* parameter)

{

int i, result;

char buf\[\] = \"message3 No.x\";

while (1)

{

for (i = 0; i \< 10; i++)

{

buf\[sizeof(buf) - 2\] = \'0\' + i;

printf(\"thread3 send \[%s\]\\n\", buf);

/\* 发送消息到消息队列中 \*/

result = mq\_send(mqueue, &buf\[0\], sizeof(buf), 0);

if ( result == -1)

{

/\* 消息队列满， 延迟1s时间 \*/

printf(\"thread3:message queue is full, delay 1s\\n\");

sleep(1);

}

}

/\* 休眠2秒\*/

sleep(2);

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

struct mq\_attr mqstat;

int oflag = O\_CREAT\|O\_RDWR;

\#define MSG\_SIZE 128

\#define MAX\_MSG 128

memset(&mqstat, 0, sizeof(mqstat));

mqstat.mq\_maxmsg = MAX\_MSG;

mqstat.mq\_msgsize = MSG\_SIZE;

mqstat.mq\_flags = 0;

mqueue = mq\_open(\"mqueue1\",O\_CREAT,0777,&mqstat);

/\*创建线程1,线程入口是thread1\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\*创建线程2,线程入口是thread2\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

/\*创建线程3,线程入口是thread3\_entry,
属性参数设为NULL选择默认值，入口参数为NULL\*/

result = pthread\_create(&tid3,NULL,thread3\_entry,NULL);

check\_result(\"thread3 created\",result);

return 0;

}

线程高级编程
============

[]{#_线程属性
.anchor}本章节会对一些很少使用的属性对象及相关函数做详细介绍。

线程属性
--------

线程有属性，定义在pthread\_attr\_t结构体里。[[]{#OLE_LINK22
.anchor}]{#OLE_LINK21
.anchor}RT-Thread实现的线程属性包括线程栈大小、线程优先级、线程分离状态、线程调度策略。pthread\_create()使用属性对象前必须先对属性对象进行初始化。设置线程属性之类的API函数应在创建线程之前就调用。线程属性的变更不会影响到已创建的线程。

线程属性结构pthread\_attr\_t定义在pthread.h头文件里。线程属性结构如下：

/\* pthread\_attr\_t 类型重定义\*/

typedef struct pthread\_attr pthread\_attr\_t;

/\*线程属性结构体\*/

struct [[]{#OLE_LINK2 .anchor}]{#OLE_LINK1 .anchor}pthread\_attr

{

void\* stack\_base; /\* 线程栈的地址 \*/

rt\_uint32\_t stack\_size; /\* 线程栈大小 \*/

rt\_uint8\_t priority; /\* 线程优先级 \*/

rt\_uint8\_t detachstate; /\*线程的分离状态 \*/

rt\_uint8\_t policy; /\* 线程调度策略 \*/

rt\_uint8\_t inheritsched; /\* 线程的继承性 \*/

};

### 线程属性初始化及去初始化

+----------+------------------------------------------------------+
| 头文件   | \#include\<pthread.h\>                               |
+==========+======================================================+
| 函数原型 | int pthread\_attr\_init(pthread\_attr\_t \*attr);    |
|          |                                                      |
|          | int pthread\_attr\_destroy(pthread\_attr\_t \*attr); |
+----------+------------------------------------------------------+
| 功能     | 初始化线程属性 \\ 对线程属性去初始化                 |
+----------+------------------------------------------------------+
| 参数     | 描述                                                 |
+----------+------------------------------------------------------+
| attr     | 指向线程属性的指针                                   |
+----------+------------------------------------------------------+
| 返回值   | 2个函数只返回0值，总是成功                           |
+----------+------------------------------------------------------+

使用pthread\_attr\_init()函数会使用默认值初始化线程属性结构体attr，等同于调用线程初始化函数时将此参数设置为NULL，使用前需要定义一个pthread\_attr\_t属性对象，此函数必须在pthread\_create()函数之前调用。

pthread\_attr\_destroy()函数对attr指向的属性去初始化，之后可以再次调用pthread\_attr\_init()函数对此属性对象重新初始化。

### 线程的分离状态

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int [[]{#OLE_LINK36               |
|                                   | .anchor}]{#OLE_LINK35             |
|                                   | .anchor}pthread\_attr\_setdetachs |
|                                   | tate(pthread\_attr\_t             |
|                                   | \*attr, int state);               |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getdetachstate(pth |
|                                   | read\_attr\_t                     |
|                                   | const \*attr, int \*state);       |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置线程的分离状态 /              |
|                                   | 获取线程的分离状态，默认情况下线程是非分离状态 |
+-----------------------------------+-----------------------------------+
| attr                              | 指向线程属性的指针                |
+-----------------------------------+-----------------------------------+
| state                             | 线程分离状态                      |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数只返回0值，总是成功        |
+-----------------------------------+-----------------------------------+

线程分离状态属性值state可以是PTHREAD\_CREATE\_JOINABL（非分离）和PTHREAD\_CREATE\_DETACHED（分离）。

线程的分离状态决定一个线程以什么样的方式来回收自己运行结束后占用的资源。线程的分离状态有2种：joinable或者detached。当线程创建后，应该调用pthread\_join()或者pthread\_detach()回收线程结束运行后占用的资源。如果线程的分离状态为joinable其他线程可以调用pthread\_join()函数等待该线程结束并获取线程返回值，然后回收线程占用的资源。分离状态为detached的线程不能被其他的线程所join，自己运行结束后，马上释放系统资源。

### 线程的调度策略

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setschedpolicy(pth |
|                                   | read\_attr\_t                     |
|                                   | \*attr, int policy);              |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getschedpolicy(pth |
|                                   | read\_attr\_t                     |
|                                   | const \*attr, int \*policy);      |
+-----------------------------------+-----------------------------------+
| 功能                              | 只实现了函数接口，默认不同优先级基于优先级调度，同一优先级时间片轮 |
|                                   | 询调度                            |
+-----------------------------------+-----------------------------------+

### 线程的调度参数

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setschedparam(pthr |
|                                   | ead\_attr\_t                      |
|                                   | \*attr,struct sched\_param const  |
|                                   | \*param)；                        |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getschedparam(pthr |
|                                   | ead\_attr\_t                      |
|                                   | const \*attr,struct sched\_param  |
|                                   | \*param)；                        |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置线程的优先级 /                |
|                                   | 获取线程的优先级，                |
|                                   |                                   |
|                                   | 获取系统支持的最小/最大优先级参数 |
+-----------------------------------+-----------------------------------+
| attr                              | 指向线程属性的指针                |
+-----------------------------------+-----------------------------------+
| param                             | 指向调度参数的指针                |
+-----------------------------------+-----------------------------------+
| 返回值                            | [[]{#OLE_LINK89                   |
|                                   | .anchor}]{#OLE_LINK88             |
|                                   | .anchor}2个函数只返回0值，总是成功 |
+-----------------------------------+-----------------------------------+

pthread\_attr\_setschedparam()函数设置线程的优先级。使用param对线程属性优先级赋值。

参数struct sched\_param定义在sched.h里，结构如下：

struct sched\_param

{

int sched\_priority; /\* 线程优先级 \*/

};

结构体sched\_param的成员sched\_priority控制线程的优先级值。

### 线程的堆栈大小

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setstacksize(pthre |
|                                   | ad\_attr\_t                       |
|                                   | \*attr, size\_t stack\_size);     |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getstacksize(pthre |
|                                   | ad\_attr\_t                       |
|                                   | const \*attr, size\_t             |
|                                   | \*stack\_size);                   |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置 / 获取 线程的堆栈大小        |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数只返回0值，总是成功        |
+-----------------------------------+-----------------------------------+

pthread\_attr\_setstacksize()函数可以设置堆栈大小，单位是字节。在大多数系统中需要做栈空间地址对齐（例如ARM体系结构中需要向4字节地址对齐）。

### 线程堆栈

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setstack(pthread\_ |
|                                   | attr\_t                           |
|                                   | \*attr,void \*stack\_base,size\_t |
|                                   | stack\_size);                     |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getstack(pthread\_ |
|                                   | attr\_t                           |
|                                   | const \*attr,void                 |
|                                   | \*\*stack\_base,                  |
|                                   |                                   |
|                                   | size\_t \*stack\_size);           |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置 / 获取                       |
|                                   | 线程的堆栈地址和堆栈大小          |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数只返回0值，总是成功        |
+-----------------------------------+-----------------------------------+

### 线程属性相关桩函数

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setscope(pthread\_ |
|                                   | attr\_t                           |
|                                   | \*attr, int scope)；              |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置线程的作用域                  |
+-----------------------------------+-----------------------------------+
| 返回值                            | scope 为                          |
|                                   | PTHREAD\_SCOPE\_SYSTEM则返回0，   |
|                                   |                                   |
|                                   | scope 为                          |
|                                   | PTHREAD\_SCOPE\_PROCESS则返回EOPNOTS |
|                                   | UPP，                             |
|                                   |                                   |
|                                   | scope为其他值则返回EINVAL         |
+-----------------------------------+-----------------------------------+

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------------------
  函数原型   int pthread\_attr\_getscope(pthread\_attr\_t const \*attr)
  功能       获取线程的作用域
  返回值     PTHREAD\_SCOPE\_SYSTEM

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setstackaddr(pthre |
|                                   | ad\_attr\_t                       |
|                                   | \*attr, void \*stack\_addr);      |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getstackaddr(pthre |
|                                   | ad\_attr\_t                       |
|                                   | const \*attr, void                |
|                                   | \*\*stack\_addr);                 |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置线程的堆栈地址 /              |
|                                   | 获取线程的堆栈地址                |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数都返回EOPNOTSUPP           |
+-----------------------------------+-----------------------------------+

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_attr\_setguardsize(pthre |
|                                   | ad\_attr\_t                       |
|                                   | \*attr, size\_t guard\_size);     |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_attr\_getguardsize(pthre |
|                                   | ad\_attr\_t                       |
|                                   | const \*attr, size\_t             |
|                                   | \*guard\_size);                   |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置线程的警戒缓冲区 /            |
|                                   | 获取线程的警戒缓冲区              |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数都返回EOPNOTSUPP           |
+-----------------------------------+-----------------------------------+

### 线程属性示例代码

这个程序会初始化2个线程，它们拥有共同的入口函数，但是它们的入口参数不相同。最先创建的线程会使用提供的attr线程属性，另外一个线程使用系统默认的属性。线程的优先级是很重要的一个参数，因此这个程序会修改第一个创建的线程的优先级为8，而系统默认的优先级为24。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

\#include \<sched.h\>

/\* 线程控制块 \*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 线程入口函数\*/

static void\* thread\_entry(void\* parameter)

{

int count = 0;

int no = (int) parameter; /\* 获得线程的入口参数 \*/

while (1)

{

/\* 打印输出线程计数值 \*/

printf(\"thread%d count: %d\\n\", no, count ++);

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

pthread\_attr\_t attr; /\* 线程属性 \*/

struct sched\_param prio; /\* 线程优先级 \*/

prio.sched\_priority = 8; /\* 优先级设置为8 \*/

pthread\_attr\_init(&attr); /\* 先使用默认值初始化属性 \*/

pthread\_attr\_setschedparam(&attr,&prio); /\* 修改属性对应的优先级 \*/

/\* 创建线程1,属性为attr，入口函数是thread\_entry，入口函数参数是1 \*/

result = pthread\_create(&tid1,&attr,thread\_entry,(void\*)1);

check\_result(\"thread1 created\",result);

/\* 创建线程2,属性为默认值，入口函数是thread\_entry，入口函数参数是2 \*/

result = pthread\_create(&tid2,NULL,thread\_entry,(void\*)2);

check\_result(\"thread2 created\",result);

return 0;

}

打开PuTTY使用list\_thread命令可以查看所有线程的状态：

![](media/image5.png){width="6.0in" height="3.5in"}

线程取消
--------

取消是一种让一个线程可以结束其它线程运行的机制。一个线程可以对另一个线程发送一个取消请求。依据设置的不同，目标线程可能会置之不理，可能会立即结束也可能会将它推迟到下一个取消点才结束。

### 发送取消请求

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------
  函数原型   int pthread\_cancel(pthread\_t thread);
  参数       描述
  thread     线程句柄
  返回值     只返回0值，总是成功

此函数发送取消请求给thread线程。Thread线程是否会对取消请求做出回应以及什么时候做出回应依赖于线程取消的状态及类型。

### 设置取消状态

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int pthread\_setcancelstate(int   |
|                                   | state, int \*oldstate);           |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| thread                            | 指向线程控制块的指针              |
+-----------------------------------+-----------------------------------+
| state                             | 有两种值：PTHREAD\_CANCEL\_ENABLE：取消使能 |
|                                   |                                   |
|                                   |                                   |
|                                   | PTHREAD\_CANCEL\_DISABLE：取消不使能（线程 |
|                                   | 创建时的默认值）                  |
+-----------------------------------+-----------------------------------+
| oldstate                          | []{#OLE_LINK79                    |
|                                   | .anchor}保存原来的取消状态        |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，state非PTHREAD\_CANCEL\_ENABL |
|                                   | E或者PTHREAD\_CANCEL\_DISABLE     |
+-----------------------------------+-----------------------------------+

此函数设置取消状态，由线程自己调用。取消使能的线程将会对取消请求做出反应，而取消没有使能的线程不会对取消请求做出反应。

### 设置取消类型

+-----------------------------------+-----------------------------------+
| []{#_Hlk498688066 .anchor}头文件  | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int pthread\_setcanceltype(int    |
|                                   | type, int \*[[]{#OLE_LINK15       |
|                                   | .anchor}]{#OLE_LINK14             |
|                                   | .anchor}oldtype);                 |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| type                              | 有2种值：PTHREAD\_CANCEL\_DEFFERED：线程 |
|                                   | 收到取消请求后继续运行至下一个取消点再结束。（线程创建时的默认值） |
|                                   |                                   |
|                                   |                                   |
|                                   | PTHREAD\_CANCEL\_ASYNCHRONOUS：线程立 |
|                                   | 即结束。                          |
+-----------------------------------+-----------------------------------+
| oldtype                           | 保存原来的取消类型                |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，type非PTHREAD\_CANCEL\_DEFFER |
|                                   | ED或者PTHREAD\_CANCEL\_ASYNCHRONOUS |
+-----------------------------------+-----------------------------------+

此函数设置取消类型，由线程自己调用。

### 设置取消点

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------
  函数原型   void pthread\_testcancel(void);

此函数在线程调用的地方创建一个取消点。主要由不包含取消点的线程调用，可以回应取消请求。

如果在取消状态处于禁用状态下调用pthread\_testcancel()，则该函数不起作用。

#### 取消点

取消点也就是线程接受取消请求后会结束运行的地方，根据POSIX标准，pthread\_join()、pthread\_testcancel()、pthread\_cond\_wait()、
pthread\_cond\_timedwait()、sem\_wait()等会引起阻塞的系统调用都是取消点。

RT-Thread包含的所有取消点如下：

-   mq\_receive()

-   mq\_send()

-   mq\_timedreceive()

-   mq\_timedsend()

-   msgrcv()

-   msgsnd()

-   msync()

-   pthread\_cond\_timedwait()

-   pthread\_cond\_wait()

-   pthread\_join()

-   pthread\_testcancel()

-   sem\_timedwait()

-   sem\_wait()

-   pthread\_rwlock\_rdlock()

-   pthread\_rwlock\_timedrdlock()

-   pthread\_rwlock\_timedwrlock()

<!-- -->

-   pthread\_rwlock\_wrlock()

### 线程取消示例代码

此程序会创建2个线程，线程2开始运行后马上休眠8秒，线程1设置了自己的取消状态和类型，之后在一个无限循环里打印运行计数信息。线程2唤醒后向线程1发送取消请求，线程1收到取消请求后马上结束运行。

\#include \<pthread.h\>

\#include \<unistd.h\>

\#include \<stdio.h\>

/\* 线程控制块\*/

static pthread\_t tid1;

static pthread\_t tid2;

/\* 函数返回值检查 \*/

static void check\_result(char\* str,int result)

{

if (0 == result)

{

printf(\"%s successfully!\\n\",str);

}

else

{

printf(\"%s failed! error code is %d\\n\",str,result);

}

}

/\* 线程1入口函数 \*/

static void\* thread1\_entry(void\* parameter)

{

int count = 0;

/\* 设置线程1的取消状态使能，取消类型为线程收到取消点后马上结束 \*/

pthread\_setcancelstate(PTHREAD\_CANCEL\_ENABLE, NULL);

pthread\_setcanceltype(PTHREAD\_CANCEL\_ASYNCHRONOUS, NULL);

while(1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread1 run count: %d\\n\",count ++);

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 线程2入口函数\*/

static void\* thread2\_entry(void\* parameter)

{

int count = 0;

sleep(8);

/\* 向线程1发送取消请求 \*/

pthread\_cancel(tid1);

/\* 阻塞等待线程1运行结束 \*/

pthread\_join(tid1,NULL);

printf(\"thread1 exited!\\n\");

/\* 线程2打印信息开始输出 \*/

while(1)

{

/\* 打印线程计数值输出 \*/

printf(\"thread2 run count: %d\\n\",count ++);

sleep(2); /\* 休眠2秒 \*/

}

}

/\* 用户应用入口 \*/

int rt\_application\_init()

{

int result;

/\* 创建线程1,属性为默认值，分离状态为默认值joinable,

入口函数是thread1\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid1,NULL,thread1\_entry,NULL);

check\_result(\"thread1 created\",result);

/\* 创建线程2,属性为默认值，分离状态为默认值joinable,

入口函数是thread2\_entry，入口函数参数为NULL \*/

result = pthread\_create(&tid2,NULL,thread2\_entry,NULL);

check\_result(\"thread2 created\",result);

return 0;

}

打开PuTTY可查看线程打印的信息：

![](media/image9.png){width="6.0in" height="3.5in"}

一次性初始化
------------

  头文件          \#include\<pthread.h\>
  --------------- ----------------------------------------------------------------------------------------------------------------------------------
  函数原型        int [[]{#OLE_LINK7 .anchor}]{#OLE_LINK6 .anchor}pthread\_once(pthread\_once\_t \* once\_control, void (\*init\_routine) (void));
  参数            描述
  once\_control   控制变量
  init\_routine   执行函数
  返回值          只返回0值，总是成功

有时候我们需要对一些变量只进行一次初始化。如果我们进行多次初始化程序就会出现错误。在传统的顺序编程中，一次性初始化经常通过使用布尔变量来管理。控制变量被静态初始化为0，而任何依赖于初始化的代码都能测试该变量。如果变量值仍然为0，则它能实行初始化，然后将变量置为1。以后检查的代码将跳过初始化。

线程结束后清理
--------------

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | void [[]{#OLE_LINK81              |
|                                   | .anchor}]{#OLE_LINK80             |
|                                   | .anchor}pthread\_cleanup\_pop(int |
|                                   | execute);                         |
|                                   |                                   |
|                                   | void pthread\_cleanup\_push(void  |
|                                   | (\*routine)(void\*), void \*arg); |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| execute                           | 0或1，决定是否执行cleanup函数     |
+-----------------------------------+-----------------------------------+
| \*routine                         | 指向清理函数的指针                |
+-----------------------------------+-----------------------------------+
| arg                               | 传递给清理函数的参数              |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数都没有返回值               |
+-----------------------------------+-----------------------------------+

pthread\_cleanup\_push()把指定的清理函数routine放到线程的清理函数链表里，pthread\_cleanup\_pop()从清理函数链表头部取出第一项函数，若execute为非0值，则执行此函数。

其他线程相关函数
----------------

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int pthread\_equal                |
|                                   | ([[]{#OLE_LINK91                  |
|                                   | .anchor}]{#OLE_LINK90             |
|                                   | .anchor}pthread\_t t1, pthread\_t |
|                                   | t2);                              |
|                                   |                                   |
|                                   | pthread\_t pthread\_self (void);  |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| pthread\_t                        | 线程句柄                          |
+-----------------------------------+-----------------------------------+
| 功能                              | pthread\_equal()判断2个线程是否相等 |
|                                   |                                   |
|                                   |                                   |
|                                   | pthread\_self()获取调用线程的线程句柄 |
+-----------------------------------+-----------------------------------+
| 返回值                            | pthread\_equal()返回0或1，相等则为1，不等则为0 |
|                                   |                                   |
|                                   |                                   |
|                                   | pthread\_self()返回调用线程的句柄 |
+-----------------------------------+-----------------------------------+

+----------+---------------------------------------------------------------+
| 头文件   | \#include \<sched.h\>                                         |
+==========+===============================================================+
| 函数原型 | int sched\_get\_priority\_min(int policy);                    |
|          |                                                               |
|          | int sched\_get\_priority\_max(int policy);                    |
+----------+---------------------------------------------------------------+
| 参数     | 描述                                                          |
+----------+---------------------------------------------------------------+
| policy   | 2个值可选：SCHED\_FIFO，SCHED\_RR                             |
+----------+---------------------------------------------------------------+
| 返回值   | sched\_get\_priority\_min()返回值为0，RT-Thread里为最大优先级 |
|          |                                                               |
|          | sched\_get\_priority\_max()返回值最小优先级                   |
+----------+---------------------------------------------------------------+

其他线程相关桩函数
------------------

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------------------------------------------------------------------------------------
  函数原型   int [[]{#OLE_LINK92 .anchor}]{#OLE_LINK78 .anchor}pthread\_atfork(void (\*prepare)(void), void (\*parent)(void), void (\*child)(void));
  返回值     EOPNOTSUPP

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------
  函数原型   int pthread\_kill(pthread\_t thread, int sig);
  返回值     PTHREAD\_SCOPE\_SYSTEM

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int pthread\_spin\_init           |
|                                   | (pthread\_spinlock\_t             |
|                                   | \*[[]{#OLE_LINK94                 |
|                                   | .anchor}]{#OLE_LINK93             |
|                                   | .anchor}lock, int pshared);       |
|                                   |                                   |
|                                   | int pthread\_spin\_destroy        |
|                                   | (pthread\_spinlock\_t \*lock);    |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数都是lock无效返回EINVAL，否则返回0 |
+-----------------------------------+-----------------------------------+

  头文件     \#include\<pthread.h\>
  ---------- ---------------------------------------------------------
  函数原型   int pthread\_spin\_lock (pthread\_spinlock\_t \* lock);
  返回值     lock无效返回EINVAL，否则返回0

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------------------
  函数原型   int pthread\_spin\_trylock (pthread\_spinlock\_t \* lock);
  返回值     lock无效返回EINVAL，否则返回0或EBUSY

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------
  函数原型   int pthread\_spin\_unlock (pthread\_spinlock\_t \* lock);
  返回值     lock无效返回EINVAL，否则返回0或EPERM

互斥锁属性
----------

RT-Thread实现的互斥锁属性包括互斥锁类型和互斥锁作用域。

### 互斥锁属性初始化及去初始化

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_mutexattr\_init(pthread\ |
|                                   | _mutexattr\_t                     |
|                                   | \*[]{#OLE_LINK10 .anchor}attr);   |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_mutexattr\_destroy(pthre |
|                                   | ad\_mutexattr\_t                  |
|                                   | \*attr);                          |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| attr                              | 指向互斥锁属性对象的指针          |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，attr为NULL              |
+-----------------------------------+-----------------------------------+

pthread\_mutexattr\_init()函数将使用默认值初始化attr指向的属性对象，等同于调用pthread\_mutex\_init()函数时将属性参数设置为NULL。

pthread\_mutexattr\_destroy()函数将会对attr指向的属性对象去初始化，之后可以调用pthread\_mutexattr\_init()函数重新初始化。

### 互斥锁作用域

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_mutexattr\_setpshared(pt |
|                                   | hread\_mutexattr\_t               |
|                                   | \*attr, int pshared);             |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_mutexattr\_getpshared(pt |
|                                   | hread\_mutexattr\_t               |
|                                   | \*attr, int \*pshared);           |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置 / 获取 互斥锁变量的作用域    |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| type                              | 互斥锁类型                        |
+-----------------------------------+-----------------------------------+
| pshared                           | 有2个可选值:                      |
|                                   |                                   |
|                                   | PTHREAD\_PROCESS\_PRIVATE：默认值，用于仅 |
|                                   | 同步该进程中的线程。              |
|                                   |                                   |
|                                   | PTHREAD\_PROCESS\_SHARED：用于同步该进程和 |
|                                   | 其他进程中的线程。                |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，attr或者pshared无效     |
+-----------------------------------+-----------------------------------+

### 互斥锁类型

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_mutexattr\_settype(pthre |
|                                   | ad\_mutexattr\_t                  |
|                                   | \*[]{#OLE_LINK23 .anchor}attr,    |
|                                   | int type);                        |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_mutexattr\_gettype(const |
|                                   | pthread\_mutexattr\_t \*attr, int |
|                                   | \*type);                          |
+-----------------------------------+-----------------------------------+
| 功能                              | 设置/获取互斥锁的类型             |
+-----------------------------------+-----------------------------------+
| 参数                              | 描述                              |
+-----------------------------------+-----------------------------------+
| attr                              | 指向互斥锁属性对象的指针          |
+-----------------------------------+-----------------------------------+
| type                              | 互斥锁类型                        |
+-----------------------------------+-----------------------------------+
| 返回值                            | 描述                              |
+-----------------------------------+-----------------------------------+
| 0                                 | 成功                              |
+-----------------------------------+-----------------------------------+
| EINVAL                            | 无效参数，attr无效或者type无效    |
+-----------------------------------+-----------------------------------+

互斥锁的类型决定了一个线程在获取一个互斥锁时的表现方式，RT-Thread实现了3种互斥锁类型：

-   PTHREAD\_MUTEX\_NORMAL：普通锁，当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后按先进先出方式获得锁。如果一个线程在不首先解除互斥锁的情况下尝试重新获得该互斥锁，不会产生死锁，而是返回错误码，和检错锁一样。

-   PTHREAD\_MUTEX\_RECURSIVE：嵌套锁，允许一个线程对同一个锁成功获得多次，需要相同次数的解锁释放该互斥锁。

-   PTHREAD\_MUTEX\_ERRORCHECK：检错锁，如果一个线程在不首先解除互斥锁的情况下尝试重新获得该互斥锁，则返回错误。这样就保证当不允许多次加锁时不会出现死锁。

条件变量属性
------------

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------
  函数原型   int pthread\_condattr\_init(pthread\_condattr\_t \*attr);
  功能       使用默认值PTHREAD\_PROCESS\_PRIVATE初始化条件变量属性attr
  参数       描述
  attr       指向条件变量属性对象的指针
  返回值     描述
  0          成功
  EINVAL     无效参数，attr为NULL

  头文件     \#include\<pthread.h\>
  ---------- -------------------------------------------------------------------------------------
  函数原型   int pthread\_mutexattr\_getpshared(pthread\_mutexattr\_t \*attr, int \*pshared);
  返回值     参数无效返回EINVAL，否则返回0，pshared指向的内存保存的值为PTHREAD\_PROCESS\_PRIVATE

### 条件变量属性相关桩函数

  头文件     \#include\<pthread.h\>
  ---------- --------------------------------------------------------------
  函数原型   int pthread\_condattr\_destroy(pthread\_condattr\_t \*attr);
  返回值     attr无效返回EINVAL，否则返回0或EPERM

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_condattr\_getclock(const |
|                                   | pthread\_condattr\_t \*attr,      |
|                                   |                                   |
|                                   | clockid\_t \*clock\_id);          |
|                                   |                                   |
|                                   | int                               |
|                                   | pthread\_condattr\_setclock(pthre |
|                                   | ad\_condattr\_t                   |
|                                   | \*attr,                           |
|                                   |                                   |
|                                   | clockid\_t clock\_id);            |
+-----------------------------------+-----------------------------------+
| 返回值                            | 2个函数只返回0值                  |
+-----------------------------------+-----------------------------------+

  头文件     \#include\<pthread.h\>
  ---------- --------------------------------------------------------------------------------
  函数原型   int pthread\_mutexattr\_setpshared(pthread\_mutexattr\_t \*attr, int pshared);
  返回值     参数无效返回EINVAL或者ENOSYS，否则返回0值

读写锁属性
----------

  头文件     \#include\<pthread.h\>
  ---------- ----------------------------------------------------------------
  函数原型   int pthread\_rwlockattr\_init (pthread\_rwlockattr\_t \*attr);
  功能       使用默认值PTHREAD\_PROCESS\_PRIVATE初始化读写锁属性attr
  参数       描述
  attr       指向读写锁属性的指针
  返回值     描述
  0          成功
  EINVAL     无效参数，attr为NULL

  头文件     \#include\<pthread.h\>
  ---------- -------------------------------------------------------------------------------------------
  函数原型   int pthread\_rwlockattr\_getpshared (const pthread\_rwlockattr\_t \*attr, int \*pshared);
  返回值     参数无效返回EINVAL，否则返回0，pshared指向的内存保存的值为PTHREAD\_PROCESS\_PRIVATE

### 读写锁属性相关桩函数

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------------------------------
  函数原型   int pthread\_rwlockattr\_setpshared (pthread\_rwlockattr\_t \*attr, int pshared);
  返回值     参数无效返回EINVAL，否则返回0值

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int pthread\_rwlockattr\_destroy  |
|                                   | (pthread\_rwlockattr\_t \*attr);  |
+-----------------------------------+-----------------------------------+
| 返回值                            | 参数无效返回EINVAL，否则返回0值   |
+-----------------------------------+-----------------------------------+

屏障属性
--------

  头文件     \#include\<pthread.h\>
  ---------- -----------------------------------------------------------------
  函数原型   int pthread\_barrierattr\_init(pthread\_barrierattr\_t \*attr);
  功能       使用默认值PTHREAD\_PROCESS\_PRIVATE初始化屏障属性attr
  参数       描述
  attr       指向屏障属性的指针
  返回值     描述
  0          成功
  EINVAL     无效参数，attr为NULL

  头文件     \#include\<pthread.h\>
  ---------- ------------------------------------------------------------------------------------
  函数原型   int pthread\_barrierattr\_setpshared(pthread\_barrierattr\_t \*attr, int pshared);
  功能       获取屏障的作用域
  参数       描述
  attr       指向屏障属性的指针
  pshared    屏障作用域，只能使用PTHREAD\_PROCESS\_PRIVATE
  返回值     参数无效返回EINVAL，否则返回0

  头文件     \#include\<pthread.h\>
  ---------- --------------------------------------------------------------------------------------------
  函数原型   int pthread\_barrierattr\_getpshared(const pthread\_barrierattr\_t \*attr, int \*pshared);
  功能       获取屏障的作用域
  参数       描述
  attr       指向屏障属性的指针
  pshared    指向保存屏障作用域数据的指针
  返回值     参数无效返回EINVAL，否则返回0

### 屏障属性相关桩函数

+-----------------------------------+-----------------------------------+
| 头文件                            | \#include\<pthread.h\>            |
+===================================+===================================+
| 函数原型                          | int                               |
|                                   | pthread\_barrierattr\_destroy(pth |
|                                   | read\_barrierattr\_t              |
|                                   | \*attr);                          |
+-----------------------------------+-----------------------------------+
| 返回值                            | 参数无效返回EINVAL，否则返回0值   |
+-----------------------------------+-----------------------------------+

消息队列属性
------------

消息队列属性控制块如下：

struct mq\_attr

{

long mq\_flags; /\* 消息队列的标志，用来表示是否阻塞 \*/

long mq\_maxmsg; /\* 消息队列最大消息数 \*/

long mq\_msgsize; /\* 消息队列每个消息的最大字节数 \*/

long mq\_curmsgs; /\* 消息队列当前消息数 \*/

};

+----------+----------------------------------------------------+
| 头文件   | \#include\<mqueue.h\>                              |
+==========+====================================================+
| 函数原型 | Int mq\_setattr(mqd\_t mqdes,                      |
|          |                                                    |
|          | const struct mq\_attr \*mqstat,                    |
|          |                                                    |
|          | struct mq\_attr \*[]{#OLE_LINK43 .anchor}omqstat); |
+----------+----------------------------------------------------+
| 功能     | RT-Thread具体未实现，是一个桩函数                  |
+----------+----------------------------------------------------+
| 返回值   | 只返回-1                                           |
+----------+----------------------------------------------------+

  头文件     \#include\<mqueue.h\>
  ---------- ----------------------------------------------------------
  函数原型   int mq\_getattr(mqd\_t mqdes, struct mq\_attr \*mqstat);
  功能       获取消息队列属性，
  参数       描述
  mqdes      指向消息队列控制块的指针
  mqstat     指向保存获取数据的指针
  返回值     成功则返回0，参数无效则返回-1

附录A 如何查找RT-Thread的文档 {#附录a-如何查找rt-thread的文档 .ListParagraph}
=============================

A.1 如何获取RT-Thread文档 {#a.1-如何获取rt-thread文档 .ListParagraph}
-------------------------

用户可以通过RT-Thread在线文档中心及时地访问到所有最新的RT-Thread官方文档和动态，详情请见:
[[http://www.rt-thread.org/document/site/]{.underline}](http://www.rt-thread.org/document/site/)

A.2 RT-Thread文档分类简介 {#a.2-rt-thread文档分类简介 .ListParagraph}
-------------------------

欢迎访问RT-Thread文档中心，RT-Thread文档按照用途分为如下几类：

+-----------------+-----------------+-----------------+-----------------+
| 用户检索提示    | 文档分类        | 用途            | 核心内容        |
+=================+=================+=================+=================+
| **遇到了具体问题？** | 应用笔记   | 面向RT-Thread某一类具 | 主要包括入门指南，进阶指南，高 |
|                 |                 | 体应用问题的综合阐述 | 级指南，移植说明，开发板说明， |
|                 | Application     |                 | 学习笔记等内容  |
| **对专题感兴趣？** | Note         |                 | ...             |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
| **想学习技巧？** |                |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
|                 | 常见问题        | 使用RT-Thread过程中可 | 常见问题  |
|                 |                 | 能遇到的常见问题说明 |            |
|                 | FAQ             |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| **想了解产品？** | 用户手册       | RT-Thread的技术参考手 | 主要包括编程手册，API手册， |
|                 |                 | 册，具体描述RT-Thread | 组件手册，设备和驱动手册，移植 |
|                 | User Manual     | 内核及其组件的具体实现和使用方 | 手册等内容 |
| **学习如何使用产品？** |          | 式              | ...             |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
| **想查找范例？** |                |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
|                 | 示例            | 使用具体的例子对于RT-Thr | 主要包括内核，组件设备和驱动的 |
|                 |                 | ead用户手册的进行补充说明 | 实例说明 |
|                 | Example Sheet   |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
|                 | 发布说明        | 每个RT-Thread发布版本 | 具体产品版本的快速上手指南 |
|                 |                 | 的功能介绍，移植简介和快速上手 |  |
|                 | Release Note    | 指南            |                 |
+-----------------+-----------------+-----------------+-----------------+

  {#section .ListParagraph}
=

免责声明 {#免责声明 .ListParagraph}
========

上海睿赛德电子科技有限公司随附提供的文档资料旨在提供给您（本公司的客户）使用，仅限于且只能在本公司销售或提供服务的产品上使用。

该文档资料为本公司和/或其供应商所有，并受适用的版权法保护，版权所有。如有违反，将面临相关适用法律的刑事制裁，并承担违背此许可的条款和条件的民事责任。

本公司保留在不通知读者的情况下，有修改文档或软件相关内容的权利，对于使用中所出现的任何效果，本公司不承担任何责任。

该软件或文档资料"按现状"提供，不提供保证，无论是明示的、暗示的还是法定的保证。这些保证包括(但不限于)对出于某一特定目的应用此软件的适销性和适用性默示的保证。在任何情况下，公司不会对任何原因造成的特别的、偶然的或间接的损害负责。

+-----------------------------------+-----------------------------------+
| []{#_Toc499916189                 | **关注RT-Thread公众号**           |
| .anchor}商务及技术支持            |                                   |
+===================================+===================================+
| 上海睿赛德电子科技有限公司        | ![](media/image10.jpeg){width="1. |
|                                   | 5694444444444444in"               |
| 地址：上海浦东新区张江高科碧波路500号310室 | height="1.573611111111111in"} |
|                                   |                                   |
|                                   |                                   |
| 邮编：201203                      |                                   |
|                                   |                                   |
| 电话：021-58995663                |                                   |
|                                   |                                   |
| 网址：[[http://www.rt-thread.com]{.u |                                |
| nderline}](http://www.rt-thread.c |                                   |
| om)                               |                                   |
|                                   |                                   |
| 商务邮箱：[[business@rt-thread.com]{.u |                              |
| nderline}](mailto:business@rt-thr |                                   |
| ead.com)                          |                                   |
|                                   |                                   |
| 技术支持：[[support@rt-thread.com]{.un |                              |
| derline}](mailto:support@rt-threa |                                   |
| d.com)                            |                                   |
+-----------------------------------+-----------------------------------+

目录  {#目录 .ListParagraph}
=====
