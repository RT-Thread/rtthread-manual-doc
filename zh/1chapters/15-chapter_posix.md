# POSIX接口 #

## 简介 ##

[描述POSIX的历史情况，POSIX API情况]

[RT-Thread中对POSIX API的支持情况]

## 在RT-Thread中使用POSIX ##

在RT-Thread中使用POSIX API接口包括几个部分：libc（例如newlib），file system，pthread等。

需要在rtconfig.h中打开相关的选项：

~~~{.c}
    #define RT_USING_LIBC
    #define RT_USING_DFS
    #define RT_USING_DFS_DEVFS
    #define RT_USING_PTHREADS
~~~

## POSIX Thread介绍 ##

[API介绍]

### 栏杆: barrier ###

pthread_barrier 系列函数在<pthread.h>中定义，用于多线程的同步，它包含三个函数：

~~~{.c}
      --pthread_barrier_init()
      --pthread_barrier_wait()
      --pthread_barrier_destroy()
~~~

pthread_barrier_*实现一个类似栏杆的功能（barrier意为栏杆)。形象的说就是把先后到达的多个线程挡在同一栏杆前，直到所有线程到齐，然后撤下栏杆同时放行。其中:

* pthread_barrier_init函数负责指定要等待的线程个数；
* pthread_barrier_wait函数由每个线程主动调用，它告诉栏杆“我到起跑线前了”。pthread_barrier_wait函数执行末尾栏杆会检查是否所有人都到栏杆前了，如果是，栏杆就消失所有线程继续执行下一句代码；如果不是，则所有已到pthread_barrier_wait函数的线程停在该函数不动，剩下没执行到pthread_barrier_wait函数的线程继续执行；
* pthread_barrier_destroy函数释放init申请的资源。

使用场景举例：

这种“栏杆”机制最大的特点就是最后一个执行wait的动作最为重要，就像赛跑时的起跑枪一样，它来之前所有人都必须等着。所以实际使用中，pthread_barrier_*常常用来让所有线程等待“起跑枪”响起后再一起行动。比如我们可以用pthread_create（）生成100 个线程，每个子线程在被create出的瞬间就会自顾自的立刻进入回调函数运行。但我们可能不希望它们这样做，因为这时主进程还没准备好，和它们一起配合的其它线程还没准备好，我们希望它们在回调函数中申请完线程空间、初始化后停下来，一起等待主进程释放一个“开始”信号，然后所有线程再开始执行业务逻辑代码。

解决方法：

为了解决上述场景问题，我们可以在init时指定n+1个等待，其中n是线程数。而在每个线程执行函数的首部调用wait()。这样100个pthread_create()结束后所有线程都停下来等待最后一个wait()函数被调用。这个wait()由主进程在它觉得合适的时候调用就好。最后这个wait()就是鸣响的起跑枪。

函数原型：

~~~{.c}
#include <pthread.h>

int pthread_barrier_init(pthread_barrier_t *restrict barrier, 
    const pthread_barrierattr_t *restrict attr, unsigned count);
int pthread_barrier_wait(pthread_barrier_t *barrier);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
~~~

参数解释：

* pthread_barrier_t，是一个计数锁，对该锁的操作都包含在三个函数内部，我们不用关心也无法直接操作。只需要实例化一个对象丢给它就好。
* pthread_barrierattr_t，锁的属性设置，设为NULL让函数使用默认属性即可。
* count，你要指定的等待个数。
