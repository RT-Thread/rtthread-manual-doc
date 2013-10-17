# finsh shell #

## 简介 ##

finsh是RT-Thread的命令行外壳（shell），提供一套供用户在命令行的操作接口，主要用于调试、查看系统信息。finsh被设计成一个不同与传统命令行的C语言表达式解释器：由于很多嵌入式系统都是采用C语言来编写，finsh正是采用了这种系统软件开发人员都会的语法形式，把C语言表达式变成了命令行的风格。它能够解析执行大部分C语言的表达式，也能够使用类似C语言的函数调用方式访问系统中的函数及全局变量，此外它也能够通过命令行方式创建变量。

## 工作模式 ##

外界由设备端口输入命令行，finsh 通过对设备输入的读取，解析输入内容，然后自动扫描内部两个段，寻找对应函数名，输出回应。

![finsh的数据流结构](figures/finsh.png)

Finsh shell的实现采用了完整的语法分析，文法分析，中间代码生成，虚拟机运行等编译器技术，其分析流程如下：

！[语法运行模式]（）

当finsh shell接受到例如8+0x200的命令输入时，它将根据预定的语法规则对它进行扫描，形成一颗语法树。然后finsh将进入编译的阶段，扫描语法树上所有节点，编译成相应的中间码（finsh虚拟机的机器指令）。finsh虚拟机被实现成一个基于栈的虚拟机，它将从编译出来的中间码中依次读取指令，例如ldd 0x08指令，它将把0x08载入到机器栈中；ldd 0x200指令，它将把0x200载入到机器栈中；addd指令，虚拟机会从栈中读取栈顶的两个值进行相加操作，然后把结果放到栈中。

此外虚拟机也包括：一个字符串池，用于放置运行过程中使用到的字符串（在指令中依据地址对它进行引用）；一个系统变量区，用于放置创建了的变量。

## RT-Thread内置命令 ##

针对RT-Thread RTOS，finsh内建了一些命令函数，可以在命令行中调用：

* 注：在finsh shell中使用命令（即Ｃ语言中的函数），必须类似Ｃ语言中的函数调用方式，即必须携带"()"符号。而最后finsh shell的输出为此函数的返回值，对于一些不存在返回值的函数，这个打印输出没有意义。要查看命令行信息必须定义对应相应的宏。

### list() ###

显示当前系统中存在的命令及变量，执行结果如下：

	--Function List:
	list_mem         -- list memory usage information
	hello            -- say hello world
	version          -- show RT-Thread version information
	list_thread      -- list thread
	list_sem         -- list semaphone in system
	list_event       -- list event in system
	list_mutex       -- list mutex in system
	list_mailbox     -- list mail box in system
	list_magqueue    -- list messgae queue in system
	list_mempool     -- list memory pool in system
	list_timer       -- list timer in system
	list_device      -- list device in system
	list             -- list all symbol in system
	--Variable List:
	dummy            -- dummy variable for finsh
		0, 0x00000000

### list_thread() ###

列表显示当前系统中线程状态，显示类似结果如下：

	thread   pri   status     sp       stack size   max used     left tick   error    
	------- ----- ------- ----------- ------------ ------------ ------------ ----
	tidle   0x1f   ready  0x00000058   0x00000100   0x00000058   0x0000000b  000
	shell   0x14   ready  0x00000080   0x00000800   0x000001b0   0x00000006  000


-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
       thread  线程的名称；

          pri  线程的优先级；

       status  线程当前的状态：

	   sp  线程当前的栈位置;

   stack size  线程的栈大小;

     max used  线程历史中使用的最大栈位置;

    left tick  线程剩余的运行节拍数;

	error  线程的错误号;
-----------------------------------------------------------------------

### list_sem() ###

列表显示系统中信号量状态，显示类似结果如下：

	semaphore  v   suspend thread    
	--------- --- ----------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
     semaphore  信号量的名称；

             v  信号量当前的值；

suspend thread  等待这个信号量的线程数目；
-----------------------------------------------------------------------

### list_event() ###

列表显示系统中事件状态，显示类似结果如下：

	event  set   suspend thread    
	----- ----- ----------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         event  事件的名称；

           set  事件当前的值；

suspend thread  等待这个事件的线程数目：
-----------------------------------------------------------------------
### list_mutex() ###

列表显示系统中互斥量状态，显示类似结果如下：

	mutex  owner  hold  suspend thread   
	-----  ----- -----  ---------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         mutxe  互斥量的名称；

         owner  当前持有互斥量的线程；

          hold  持有线程的持有次数；

suspend thread  等待这个互斥量的线程数目；
-----------------------------------------------------------------------
### list_mb() ###

列表显示系统中信箱状态，显示类似结果如下：

	mailbox  entry  size  suspend thread    
	-------  -----  ----  ---------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
       mailbox  信箱的名称；

         entry  信箱中包含的信件数目；

	  size  信箱能够容纳的最大信件数目；

suspend thread  等这个信箱上的线程数目；
-----------------------------------------------------------------------
### list_mq() ###

列表显示系统中消息队列状态，显示类似结果如下：

	msgqueue  entry  suspend thread    
	--------  -----  ----------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
      msgqueue  消息队列的名称；

         entry  消息队列当前包含的消息数目；

suspend thread  等待这个消息队列上的线程数目；
-----------------------------------------------------------------------
### list_memp() ###

列表显示系统中内存池状态，显示类似结果如下：

	mempool  block  total free  suspend thread    
	-------  -----  ----- ----  --------------

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
       mempool  内存池的名称；

         block  内存块容量；

         total  内存块总数量；

          free  空余内存块数量；

suspend thread  挂起线程数目； 
-----------------------------------------------------------------------
### list_timer() ###

列表显示系统中定时器状态，显示类似结果如下：

	timer    periodic    timeout       flag    
	------  ----------  ----------  -----------
	tidle   0x00000000  0x00000000  deactivated
	tshell  0x00000000  0x00000000  deactivated
	current tick:0x0000d7e
-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         timer  定时器的名称；

      periodic  定时器是否是周期性的；

       timeout  定时器超时时的节拍数；

          flag  定时器的状态，activated表示活动的，deactivated表示不活动的；
-----------------------------------------------------------------------
### list_device() ###

列表显示系统中设备状态，显示类似结果如下：

	device  type    
	------  ----

-----------------------------------------------------------------------
          字段   描述
-------------- -------------------------------------------------------
       device  设备的名称；

         type  设备的类型；
-----------------------------------------------------------------------

type输出下列数据类型：

~~~{.c}
char * const device_type_str[]{
	"Character Device",
	"Block Device",
	"Network Interface",
	"MTD Device",
	"CAN Device",
	"RTC",
	"Sound Device",
	"Graphic Device",
	"I2C Bus",
	"USB Slave Device",
	"USB Host Bus",
	"SPI Bus",
	"SPI Device",
	"SDIO Bus",
	"PM Pseudo Device",
	"Unknown",
};
~~~

## 应用程序接口 ##

finsh的应用程序接口提供了上层注册函数或变量的接口，使用时应包含如下头文件：

	#include <finsh.h>

### 添加函数 ###

	void finsh_syscall_append(const char* name, syscall_func func);

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  函数在finsh shell中访问的名称；

          func  函数的地址；
-----------------------------------------------------------------------

### 添加变量 ###

	void finsh_sysvar_append(const char* name,  u_char type, void* var_addr);

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  变量在finsh shell中访问的名称；

          type  数据类型（finsh_type给出）；

      var_addr  变量的地址;
-----------------------------------------------------------------------

当前finsh支持下列数据类型：

~~~{.c}
enum finsh_type{
	finsh_type_unknown = 0,	               /**< unknown data type */
	finsh_type_void,	               /**< void */
	finsh_type_voidp,	               /**< void pointer */
	finsh_type_char,	               /**< char */
	finsh_type_uchar,	               /**< unsigned char */
	finsh_type_charp,	               /**< char pointer */
	finsh_type_short,	               /**< short */
	finsh_type_ushort,	               /**< unsigned short */
	finsh_type_shortp,	               /**< short pointer */
	finsh_type_int,	                       /**< int */
	finsh_type_uint,	               /**< unsigned int */
	finsh_type_intp,	               /**< int pointer */
	finsh_type_long,	               /**< long */
	finsh_type_ulong,	               /**< unsigned long */
	finsh_type_longp,                      /**< long pointer */
};
~~~

### 宏方式输出函数、变量 ###

当使能了宏FINSH_USING_SYMTAB时，也能够使用宏输出方式向finsh shell增加命令。当需要输出函数或变量到finsh shell时，可以通过引用宏：FINSH_FUNSTION_EXPORT和FINSH_VAR_EXPORT的方式。例如：

~~~{.c}
long hello(void)
{
	rt_kprintf("Hello RT-Thread!\n");

	return 0;
}
FINSH_FUNCTION_EXPORT(hello, say hello word)

static int dummy = 0;
FINSH_VAR_EXPORT(dummy, finsh_type_int, dummy variablle for finsh)
~~~

hello函数、counter变量将自动输出到设立了中，即可在shell中调用、访问hello函数、counter变量。而定义了宏FINSH_USING_DESCRIPTION将可以在list()列出函数、变量列表时显示相应的帮助描述。

## 移植 ##

由于finsh完全采用ANSI C编写，具备极好的移植性，同时在内存占用上也非常小，如果不使用上述提到的API函数，整个finsh将不会动态申请内存。

* finsh shell线程：

每次的命令执行都是在finsh shell线程的上下文中完成的。定义了宏RT_USING_FINSH，在启动函数中就配置了finsh_system_init()和finsh_set_device(const char*device_name)函数，finsh shell线程在函数finsh_system_init()中创建，它将一直等待rx_sem信号量的释放。

* finsh的输出：

finsh的输出依赖于系统的输出，在RT-Thread中依赖的是rt_printf输出。在启动函数rt_hw_board_init()中，rt_console_set_device(const char*name)函数设置了finsh的打印输出设备。

* finsh的输入：

finsh shell线程在获得了rx_sem信号量后，调用rt_decice_read()函数从设备(选用串口设备)中获得一个字符然后处理。所以finsh的移植需要rt_decice_read()函数的实现。而rx_sem信号量的释放通过调用rx_indicate()函数以完成对finsh shell线程的输入通知。通常的过程是，当串口接收中断发生是（即串口有输入），接受中断服务例程调用rx_indicate()函数通知finsh shelli线程有输入：而后finsh shell线程获取串口输入最后做相应的命令处理。如下图所示：

![shell线程读取数据](figures/sem_shell.png)

## 宏选项 ##

finsh下有许多宏定义，开启不同部分实现的功能不一样。

### #define RT_USING_FINSH ###

要开启finsh的支持，将其作为shell，在RT-Thread的配置中必须定义,此宏在rtconfig.h中。

### #define FINSH_USING_SYMTAB 和#define FINSH_USING_DESCRIPTION  ###

可以使用内置符号表。

### #define FINSH_USING_HISTORY ###

可以查看历史指令。
