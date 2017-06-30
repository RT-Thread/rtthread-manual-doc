# finsh shell #

## 简介 ##

finsh是RT-Thread的命令行外壳（shell），提供一套供用户在命令行的操作接口，主要用于调试或查看系统信息。finsh支持两种模式：

1. C语言解释器模式，为行文方便称之为c-style；
2. 传统命令行模式，此模式又称为msh(module shell)。

C语言表达式解释模式下，finsh能够解析执行大部分C语言的表达式，并使用类似C语言的函数调用方式访问系统中的函数及全局变量，此外它也能够通过命令行方式创建变量。

在msh模式下，finsh运行方式类似于dos/bash等传统shell。

在本章的最后一节宏选项中介绍如何配置finsh，读者可以根据自己的喜好配置finsh。

## 工作模式 ##

用户由设备端口输入命令行，finsh 通过对设备输入的读取，解析输入内容，然后自动扫描内部段（内部函数表），寻找对应函数名，执行函数后输出回应。

![finsh的数据流结构](../../figures/finsh.png)

## 什么是shell? ##

在计算机发展的早期，图形系统出现之前，没有鼠标，甚至没有键盘。那时候人们如何与计算机交互呢？最早期的计算机使用打孔的纸条向计算机输入命令，编写程序。后来计算机不断发展，显示器、键盘成为计算机的标准配置，但此时的操作系统还不支持图形界面，计算机先驱们开发了一种软件，它接受用户输入的命令，解释之后，传递给操作系统，并将操作系统执行的结果返回给用户。这个程序像一层外壳包裹在操作系统的外面，所以它被称为shell。

![系统结构图](../../figures/finsh_what_is_shell.png)

在图形界面系统出现之前，shell这个命令行程序曾经统治计算机的交互接口数十年之久，并由大名鼎鼎的Unix系统发扬光大，诞生了多种shell软件，如bsh、bash、csh、ksh、zsh。这些shell功能都非常强大，不仅可以供用户输入命令，它们还都支持shell编程语言，可以完成复杂的操作。这些shell目前都可以在*nix系统上使用。即使后来windows统治PC，对于一般人来说，shell的光彩逐渐暗淡，它并未因此退出操作系统。在windows上，cmd 可以认为就是一种shell，同时在windows的后续版本发展出更强大的powershell。

## 初识finsh ##

在大部分嵌入式系统中，一般开发调试都使用硬件调试器和printf日志打印，在有些情况下，这两种方式并不是那么好用。比如对于RT-Thread这个多线程系统，我们想知道某个时刻系统中的线程运行状态、手动控制系统状态。如果有一个shell，就可以输入命令，直接执行相应的函数获得需要的信息，或者控制程序的行为，这无疑会十分方便。

### finsh(C-Style) ###

在嵌入式领域，C语言是最常用的开发语言，如果shell程序的命令是C语言的风格，那无疑是非常易用且有趣。

嵌入式设备通常采用交叉编译，一般需要将开发板与PC机连接起来通讯，常见连接方式包括，串口、USB、以太网、wifi等。一个灵活的shell也应该可以在多种连接方式上工作。

finsh正是基于这些考虑而诞生的，finsh可以发音为`[ˈfɪnʃ]`。finsh运行于开发板，它可以使用串口/以太网/USB等与PC机进行通信。其运行时的***finsh工作原理图*** 所示。

![finsh工作原理](../../figures/finsh-logic.png)

下图是finsh的实际运行运行效果图。开发板运行RT-Thread，并使能了finsh组件，通过串口与PC机连接，PC上运行Secure CRT。

![finsh运行界面](../../figures/finsh_run1.png)

按下回车，然后输入list_thread()将会打印系统当前所有线程，及其状态。关于这个命令的详细解释请参考本章最后一节。

**WARNING**: 此模式下，finsh是一个C语言风格的Shell，与Linux/Unix以及Windows下的cmd 的风格不同。在finsh shell中使用命令（即C语言中的函数），必须类似C语言中的函数调用方式，即必须携带`()`符号。最后finsh命令的输出为此函数的返回值。对于一些不存在返回值的函数（void返回值），这个打印输出没有意义。

### finsh(msh) ###

实际上，一开始finsh仅支持C-Style模式。后来随着RT-Thread的不断发展，尤其支持app module之后（请参考本书应用模块一章了解相关内容），C-Style模式执行app module时操作起来不太方便，而传统的shell则更方便些。另外，C-Style模式下，finsh占用体积较大。出于这些考虑，在RT-Thread 1.2.0中为finsh增加了msh模式。

msh模式下，finsh与传统shell（dos/bash）执行方式一致，其命令执行格式如下

    command [arg1] [arg2] [...]

其中command既可以RT-Thread内置的命令，也可以是编译出的app module（类似于dos里的exe可执行文件）。finsh(msh)内置的命令风格采用bash的风格，内置命令将在本章后续章节详细介绍。

### finsh中的按键 ###

finsh支持TAB键自动补全，当没有输入任何字符时按下TAB键将会打印当前所有的符号，包括当前导出的所有命令和变量。若已经输入部分字符时按下TAB键，将会查找匹配的命令，并自动补全，并可以继续输入，多次补全。如果是msh状态，输入一个字符后，不仅仅会按系统导出函数命令方式自动补全，也会按照文件系统的当前目录下的文件名进行补全。

上下键可以回溯最近输入的历史命令，左右键可移动光标，退格键删除。

目前finsh的按键处理还比较薄弱。不支持CTRL+C等控制键中断命令，也不支持DELETE键删除。

## finsh特性 ##

### finsh(c-style)的数据类型 ###

finsh支持基本的C语言数据类型，包括：

+----------------------+-------------------------------------+
|   数据类型           |         描述                        |
+======================+=====================================+
|   void               |   空数据格式，只用于创建指针变量    |
+----------------------+-------------------------------------+
|  char, unsigned char |   （带符号）字符型变量              |
+----------------------+-------------------------------------+
|   int, unsigned int  |   （带符号）整数型变量              |
+----------------------+-------------------------------------+
| short, unsigned short|   （带符号）短整型变量              |
+----------------------+-------------------------------------+
|  long, unsigned long |   （带符号）长整型变量              |
+----------------------+-------------------------------------+
|   char*, short*,     |    指针型变量                       |
|   long*, void *      |                                     |
+----------------------+-------------------------------------+


在finsh的命令行上，输入上述数据类型的C表达式可以被识别。浮点类型以及复合数据类型unin与struct等暂不支持。此外，finsh也不支持`if, for, while, goto, switch`等分支、跳转语句。

## finsh(c-style)中增加命令/变量 ##

finsh支持两种方式向finsh中输出符号（函数或变量），下面将分别介绍这两种方式。

### 宏方式 ###

需要在rtconfig.h中定义宏FINSH_USING_SYMTAB。

```c
#include <finsh.h>
FINSH_FUNCTION_EXPORT(name, desc)
```

-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  函数指针，一般为函数名；

          desc  针对这个函数命令的描述信息;
-----------------------------------------------------------------------

desc一般为一段字符串，中间不可以有逗号，两边也没有引号。

```c
FINSH_VAR_EXPORT(name, type, desc)  
```

-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  变量名；

          type  变量类型；

          desc  变量描述;
-----------------------------------------------------------------------

type的包括以下类型：

```c
enum finsh_type{
    finsh_type_unknown = 0,            /**< unknown data type */
    finsh_type_void,                   /**< void */
    finsh_type_voidp,                  /**< void pointer */
    finsh_type_char,                   /**< char */
    finsh_type_uchar,                  /**< unsigned char */
    finsh_type_charp,                  /**< char pointer */
    finsh_type_short,                  /**< short */
    finsh_type_ushort,                 /**< unsigned short */
    finsh_type_shortp,                 /**< short pointer */
    finsh_type_int,                    /**< int */
    finsh_type_uint,                   /**< unsigned int */
    finsh_type_intp,                   /**< int pointer */
    finsh_type_long,                   /**< long */
    finsh_type_ulong,                  /**< unsigned long */
    finsh_type_longp,                  /**< long pointer */
};
```

此外FINSH还提供了另外一个宏，它在输出函数为命令时，可以指定命令的名字。

```c
#include <finsh.h>
FINSH_FUNCTION_EXPORT_ALIAS(name, alias, desc)
```

-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  函数指针，一般为函数名；

          alias 输出到finsh中的命令名;

          desc  函数描述;
-----------------------------------------------------------------------

当函数名字过长时，可以利用上面这个宏导出一个简短的名字以方便输入。

* 说明: FINSH的函数名字长度有一定限制，它由finsh.h中的宏定义FINSH_NAME_MAX控制，默认是16字节。意味着finsh命令长度不会超过16字节。这里有个潜在的问题。当一个函数名长度超过FINSH_NAME_MAX时，使用FINSH_FUNCTION_EXPORT导出这个函数到命令表中后，在finsh符号表中看到完整的函数名，但是完整输入执行会出现null node错误。这是因为虽然显示了完整的函数名，但是实际上finsh中却保存了前16字节作为命令，过多的输入会导致无法正确找到命令，这时就可以使用FINSH_FUNCTION_EXPORT_ALIAS来对导出的命令进行重命名。

一个简单的输出函数和变量到finsh的例子：

```c
#include <finsh.h>

int var;

int hello_rtt(int a)
{
　　rt_kprintf("hello, world! I am %d\n", a);
　　return a;
}
FINSH_FUNCTION_EXPORT(hello_rtt, say hello to rtt)
FINSH_FUNCTION_EXPORT_ALIAS(hello_rtt, hr, say hello to rtt)

FINSH_VAR_EXPORT(var, finsh_type_int, just a var for test)
```

编译后运行，可以看到finsh中增加了两个命令，一个变量var。

### 函数方式 ###

```c
#include <finsh.h>
void finsh_syscall_append(const char* name, syscall_func func)
```

-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  函数在finsh中访问的名称，即命令名;

          func  函数地址，一般为函数名;
-----------------------------------------------------------------------

这个函数可以输出一个函数到finsh中，使之可以在finsh命令行中使用。

```c
#include <finsh.h>
void finsh_sysvar_append(const char* name, u_char type, void* addr)
```
-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  变量在finsh中访问的名称，即命令名;

          type  变量的类型;

          addr  变量地址;
-----------------------------------------------------------------------

这个函数用于输出一个变量到finsh中。

## msh中增加命令 ##

当使用msh模式时，finsh不支持C表达式，因此只能添加（函数）命令，并不能向c-style模式下动态创建变量。msh模式添加命令仅支持下面两种宏方式添加命令。

```c
#include <finsh.h>
MSH_CMD_EXPORT(command, desc);
FINSH_FUNCTION_EXPORT_ALIAS(name, alias, desc);
```

### 添加内置命令 ###

下面是一种向msh导出命令的方式：

```c
#include <finsh.h>

int mycmd(void)
{
    printf("hello!\n");
    return 0;
}
MSH_CMD_EXPORT(mycmd, my command test);
```

在上面的代码例子中，定义了自己的命令函数 `mycmd` ，这个函数会在命令行中输出一句

    hello!

当我们想在msh中能够调用这个命令时，就可以使用`MSH_CMD_EXPORT`的宏来定义这个函数导出到msh中。把以上的代码加入到系统中，并进行编译，当系统运行起来进入命令行后，我们可以在命令行下输入：

    msh /> mycmd
    hello!

来调用这个`mycmd`命令。我们可以看到，这个命令方式和finsh的方式并不一样，在它的后面不需要加入`()`括号（类似C代码中调用一个C函数那样），而是直接回车即可执行这个命令。

同样的，在msh下也可以使用参数，例如如下的示例代码：

```c
#include <finsh.h>

int mycmdarg(int argc, char** argv)
{
    printf("argv[0]: %s\n", argv[0]);

    if (argc > 1)
        printf("argv[1]: %s\n", argv[1]);

    return 0;
}
MSH_CMD_EXPORT(mycmdarg, my command with args);
```

当我们在命令行下运行这个命令时，特别是以不同的参数运行时，会发现：

    msh /> mycmdarg
    argv[0]: mycmdarg

    msh /> mycmdarg 0
    argv[0]: mycmdarg
    argv[1]: 0

    msh /> mycmdarg str 1 2 3
    argv[0]: mycmdarg
    argv[1]: str

即，

* argc - 反映的是总计有多少个命令行参数（也包含命令行自身）；
* argv - 反映的是命令行参数组，且都是以字符形式存储；

从上的例子可以看出，一个msh导出函数很类似于一个main函数：

```c
int main(int argc, char** argv);
```

最初的msh设计确实就是按照主函数方式进行的，所以其命令行参数传递风格也和main函数完全一致。当对命令行参数进行完整的校验时，就可以确保参数的合法性，并对非法的参数提供出相应的错误信息出来。

使用宏导出命令的形式，

```c
MSH_CMD_EXPORT(cmd, cmd description);
```

可以导出到msh下。实际上，使用finsh的函数导出宏也可以导出成msh的命令，两者的差别是函数命令在实际存放时，msh的命令名字上会多出`__cmd_`的前缀。例如以下的finsh导出宏定义也同样的可以在msh中导出对应的命令：

```c
FINSH_FUNCTION_EXPORT_ALIAS(cmd_ls, __cmd_ls, List information about the FILEs.);
```

这里面就是把cmd_ls函数重命名成__cmd_ls导出到shell中。当执行这个命令时，它会被特殊对待，只能当成msh命令使用。实际上，纯粹的finsh shell在显示命令时，对`__`开头的函数名并不显示，会被当成一类特殊的命令对待（例如提供给msh的函数命令）。

## RT-Thread内置命令 ##

在RT-Thread中默认内置了一些finsh命令，在finsh中按下TAB键可以打印则会当前系统支持所有符号，也可以输入list()回车，二者效果相同。

### finsh(c-style) ### 

注意：在finsh(c-style)中使用命令（即Ｃ语言中的函数），必须类似Ｃ语言中的函数调用方式，即必须携带"()"符号。finsh shell的输出为此函数的返回值，对于那些不存在返回值的函数，这个打印输出没有意义。要查看命令行信息必须定义对应相应的宏。

    finsh>>list()

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

    finsh>>list_thread()

    thread   pri   status     sp       stack size   max used     left tick   error    
    ------- ----- ------- ----------- ------------ ------------ ------------ ----
    tidle   0x1f   ready  0x00000058   0x00000100   0x00000058   0x0000000b  000
    shell   0x14   ready  0x00000080   0x00000800   0x000001b0   0x00000006  000

显示当前系统中线程状态：

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

    finsh>>list_sem()

    semaphore  v   suspend thread    
    --------- --- ----------------

显示系统中信号量状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
     semaphore  信号量的名称；

             v  信号量当前的值；

suspend thread  等待这个信号量的线程数目；
-----------------------------------------------------------------------

    finsh>>list_event()

    event  set   suspend thread    
    ----- ----- ----------------

显示系统中事件状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         event  事件的名称；

           set  事件当前的值；

suspend thread  等待这个事件的线程数目：
-----------------------------------------------------------------------

    finsh>>list_mutex()
 
    mutex      owner   hold   suspend thread   
    -------  -------- ------  ---------------
    fslock   (NULL)   0000    0
    lock     (NULL)   0000    0

显示系统中互斥量状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         mutxe  互斥量的名称；

         owner  当前持有互斥量的线程；

          hold  持有者在这个互斥量上嵌套持有的次数；

suspend thread  等待这个互斥量的线程数目；
-----------------------------------------------------------------------

    finsh>>list_mb()

    mailbox  entry  size  suspend thread    
    -------  -----  ----  ---------------

显示系统中信箱状态:

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
       mailbox  信箱的名称；

         entry  信箱中包含的信件数目；

          size  信箱能够容纳的最大信件数目；

suspend thread  等这个信箱上的线程数目；
-----------------------------------------------------------------------

    finsh>>list_mq()

    msgqueue  entry  suspend thread    
    --------  -----  ----------------

显示系统中消息队列状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
      msgqueue  消息队列的名称；

         entry  消息队列当前包含的消息数目；

suspend thread  等待这个消息队列上的线程数目；
-----------------------------------------------------------------------

    finsh>>list_memp() 

    mempool  block  total free  suspend thread    
    -------  -----  ----- ----  --------------

显示系统中内存池状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
       mempool  内存池的名称；

         block  内存池中内存块大小；

         total  内存块总数量；

          free  空余内存块数量；

suspend thread  挂起线程数目； 
-----------------------------------------------------------------------

    finsh>>list_timer()

    timer    periodic    timeout       flag    
    ------  ----------  ----------  -----------
    tidle   0x00000000  0x00000000  deactivated
    tshell  0x00000000  0x00000000  deactivated
    current tick:0x0000d7e

显示系统中定时器状态：

-----------------------------------------------------------------------
          字段  描述
--------------  -------------------------------------------------------
         timer  定时器的名称；

      periodic  定时器是否是周期性的；

       timeout  定时器超时时的节拍数；

          flag  定时器的状态，activated表示活动的，deactivated表示不活动的；

  current tick  当前系统的节拍;
-----------------------------------------------------------------------

    finsh>> list_device()

    device       type    
    -------  ----------------
    uart3    Character Device 
    uart2    Character Device 
    uart1    Character Device

显示系统中设备状态：

-----------------------------------------------------------------------
          字段   描述
-------------- -------------------------------------------------------
       device  设备的名称；

         type  设备的类型；
-----------------------------------------------------------------------

type输出下列数据类型：

```c
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
```

RT-Thread的各个组件会向finsh输出一些命令。 如当打开DFS组件时，还会增加如下命令，各个命令详细介绍参见文件系统一章。

    mkfs             -- make a file system
    df               -- get disk free
    ls               -- list directory contents
    rm               -- remove files or directories
    cat              -- print file
    copy             -- copy source file to destination file
    mkdir            -- create a directory

### finsh(msh) 内置命令 ###

msh模式下，内置命令风格与bash类似，按下tab键后可以列出当前支持的所有命令。

    RT-Thread shell commands:
    list_timer       - list timer in system
    list_device      - list device in system
    version          - show RT-Thread version information
    list_thread      - list thread
    list_sem         - list semaphore in system
    list_event       - list event in system
    list_mutex       - list mutex in system
    list_mailbox     - list mail box in system
    list_msgqueue    - list message queue in system
    ls               - List information about the FILEs.
    cp               - Copy SOURCE to DEST.
    mv               - Rename SOURCE to DEST.
    cat              - Concatenate FILE(s)
    rm               - Remove (unlink) the FILE(s).
    cd               - Change the shell working directory.
    pwd              - Print the name of the current working directory.
    mkdir            - Create the DIRECTORY.
    ps               - List threads in the system.
    time             - Execute command with time.
    free             - Show the memory usage in the system.
    exit             - return to RT-Thread shell mode.
    help             - RT-Thread shell help.

执行方式与传统shell相同，因此不详细赘述，以cat为例简单介绍。如果打开DFS，并正确挂载了文件系统，则可以执行ls查看列出的当前目录。

    finsh>ls
    Directory /:
    ..                  <DIR>
    a.txt               1119

当前目录下存在名为a.txt的文件，则可执行如下命令打印a.txt的内容。

    finsh>> cat a.txt

## 移植 ##

finsh完全采用ANSI C编写，具备极好的移植性；内存占用少，如果不使用前面章节中介绍的函数方式动态地向finsh添加符号，finsh将不会动态申请内存。finsh源码位于 `components/finsh` 目录下。移植finsh需要注意以下几个方面：

* finsh shell线程：

每次的命令执行都是在finsh shell线程的上下文中完成的。当定义RT_USING_FINSH宏时，就可以在初始化线程中调用finsh_system_init()初始化finsh shell线程。RT-Thread 1.2.0之后的版本中可以不使用 `finsh_set_device(const char* device_name)` 函数去显式指定使用的设备，而是会自动调用 `rt_console_get_device()` 函数去使用console设备（RT-Thread 1.1.x及以下版本中必须使用 `finsh_set_device(const char* device_name)` 指定finsh shell使用的设备）。finsh shell线程在函数 `finsh_system_init()` 函数中被创建，它将一直等待rx_sem信号量。

* finsh的输出：

finsh的输出依赖于系统的输出，在RT-Thread中依赖`rt_kprintf`输出。在启动函数 `rt_hw_board_init()` 中， `rt_console_set_device(const char* name)` 函数设置了finsh的打印输出设备。

* finsh的输入：

finsh shell线程在获得了rx_sem信号量后，调用 `rt_device_read()` 函数从设备(选用串口设备)中获得一个字符然后处理。所以finsh的移植需要 `rt_device_read()` 函数的实现。而rx_sem信号量的释放通过调用 `rx_indicate()` 函数以完成对finsh shell线程的输入通知。通常的过程是，当串口接收中断发生是（即串口有输入），接受中断服务例程调用 `rx_indicate()` 函数通知finsh shelli线程有输入：而后finsh shell线程获取串口输入最后做相应的命令处理。

## 宏选项 ##

finsh有一些宏定义可以简单配置。

```c
    #define RT_USING_FINSH
```

此宏定义在rtconfig.h中，用于在RT-Thread中打开finsh，并将其作为shell。

```c
    #define FINSH_USING_SYMTAB
    #define FINSH_USING_DESCRIPTION
```

此宏定义在rtconfig.h中。打开`FINSH_USING_SYMTAB`可以在finsh中使用符号表，打开`FINSH_USING_DESCRIPTION`后需要给每个finsh的符号添加一段字符串描述。这两个宏一般都需要打开。

```c
    #define FINSH_USING_HISTORY
```

此宏定义在rtconfig.h中，打开后可以在finsh中使用方向键（上下）回溯历史指令。

```c
    #define FINSH_USING_MSH
```

此宏定义在rtconfig.h中，打开后finsh将支持传统shell模式。

```c
    #define FINSH_USING_MSH_ONLY
```

此宏定义在rtconfig.h中，打开后finsh仅支持msh模式。

如果打开了`FINSH_USING_MSH`而没有打开`FINSH_USING_MSH_ONLY`，finsh同时支持两种c-style模式与msh模式，但是默认进入c-style模式，执行 `msh()`即可切换到msh模式，在msh模式下执行 `exit`后即退回到c-style模式。

```c
    #define DFS_USING_WORKDIR
```

此宏定义在rtconfig.h中，它实际上是DFS组件的宏，但由于它与finsh有一定关系，因此在这里也介绍一下。打开此宏后finsh可以支持工作目录。当使用msh时，建议打开此宏。

```c
    #define FINSH_USING_AUTH
```

此宏定义在rtconfig.h中，打开则开启权限验证功能。系统在启动后，只有权限验证（目前仅支持密码验证）通过，才会开启finsh功能，提升系统输入的安全性。

```c
    #define FINSH_DEFAULT_PASSWORD  "rtthread"
```

此宏定义在rtconfig.h中，设置finsh在密码验证模式下的默认密码。密码长度大于等于`FINSH_PASSWORD_MIN`（默认6），小于等于`FINSH_PASSWORD_MAX`（默认`RT_NAME_MAX`）。
