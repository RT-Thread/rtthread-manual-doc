# finsh shell #

## 简介 ##

finsh是RT-Thread的命令行外壳（shell），提供一套供用户在命令行的操作接口，主要用于调试、查看系统信息。finsh被设计成一个不同与传统命令行的C语言表达式解释器：由于很多嵌入式系统都是采用C语言来编写，finsh正是采用了这种系统软件开发人员都会的语法形式，把C语言表达式变成了命令行的风格。它能够解析执行大部分C语言的表达式，也能够使用类似C语言的函数调用方式访问系统中的函数及全局变量，此外它也能够通过命令行方式创建变量。

![finsh的数据流结构](figures/finsh.png)

## RT-Thread内置命令 ##

针对RT-Thread RTOS，finsh提供了一系列基本函数命令：

### list() ###

### list_thread() ###

列表显示当前系统中线程状态，显示类似结果如下：

  thread   pri   status     sp     stack size  max used  left tick  error    
  ------- ----- ------- ---------- ---------- ---------- ---------- ----
  tidle   0x1f   ready  0x00000058 0x00000100 0x00000058 0x0000000b 000
  shell   0x14   ready  0x00000080 0x00000800 0x000001b0 0x00000006 000

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

### list_thread() ###
### list_sem() ###
### list_event() ###
### list_mutex() ###
### list_mailbox() ###
### list_msgqueue() ###
### list_memheap() ###
### list_timer() ###
### list_device() ###
### list_module() ###
### list_mod_detail(*name) ###


## 应用程序接口 ##

finsh的应用程序接口提供了上层注册函数或变量的接口，使用时应包含如下头文件：

#include<finsh.h>

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
每次的命令执行都是在finsh shell线程的上下文中完成的，finsh shell线程在函数finsh_system_init()中创建，它将一直等待uart_sem信号量的释放。
*finsh的输出：
finsh的输出依赖于系统的输出，在RT-Thread中依赖的是rt_printf输出。
*finsh的输入：
finsh shell线程在获得了uart_sem信号量后，调用rt_serial_getc()函数从串口中获得一个字符然后处理。所以finsh的移植需要rt_serial_getc()函数的实现。而uart_sem信号量的释放通过调用finsh_notify()函数以完成对finsh shell线程的输入通知。
通常的过程是，当串口接收中断发生是（即串口有输入），接受中断服务例程调用finsh_notify()函数通知finsh shelli线程有输入：而后finsh shell线程获取串口输入最后做相应的命令处理。

## 宏选项 ##

finsh下有许多宏定义，开启不同部分实现的功能不一样。

### #define RT_USING_FINSH ###

要开启finsh的支持，在RT-Thread的配置中必须定义，

### #define FINSH_USING_SYMTAB ###
### #define FINSH_USING_DESCRIPTION ###
### #define FINSH_USING_HISTORY ###
