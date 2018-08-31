# 动态模块 #

在传统桌面操作系统中，用户空间和内核空间是分开的，应用程序运行在用户空间，内核以及内核模块则运行于内核空间，其中内核模块可以动态加载与删除以扩展内核功能。`dlmodule` 则是RT-Thread下，在内核空间对外提供的动态模块加载机制的软件组件。在RT-Thread v3.1.0以前的版本中，这也称之为应用模块（Application Module），在RT-Thread v3.1.0及之后，则回归传统，以动态模块命名。

`dlmodule`组件更多的是一个ELF格式加载器，把单独编译的一个elf文件的代码段，数据段加载到内存中，并对其中的符号进行解析，绑定到内核导出的API地址上。动态模块elf文件主要放置于RT-Thread下的文件系统上。

## 功能和限制 ##

动态模块为RT-Thread提供了动态加载程序模块的机制，因为也独立于内核编译，所以使用方式比较灵活。从实现上讲，这是一种将内核和动态模块分开的机制，通过这种机制，内核和动态模块可以分开编译，并在运行时通过内核中的模块加载器将编译好的动态模块加载到内核中运行。

在RT-Thread的动态模块中，目前支持两种格式：

* `.mo` 则是编译出来时以`.mo`做为后缀名的可执行动态模块；它可以被加载，并且系统中会自动创建一个主线程执行这个动态模块中的`main`函数；同时这个`main(int argc, char** argv)`函数也可以接受命令行上的参数。
* `.so` 则是编译出来时以`.so`做为后缀名的动态库；它可以被加载，并驻留在内存中，并提供一些函数集由其他程序（内核里的代码或动态模块）来使用。

当前RT-Thread支持应用模块的架构主要包括ARM类架构，未来会扩展到x86，MIPS，以及RISC-V等架构上。RT-Thread内核固件部分可使用多种编译器工具链，如GCC, ARMCC、IAR等工具链；但动态模块部分编译当前只支持GNU GCC工具链编译。因此编译RT-Thread模块需下载GCC工具，例如CodeSourcery的arm-none-eabi工具链。一般的，最好内核和动态模块使用一样的工具链进行编译（这样不会在libc上产生不一致的行为）。另外，动态模块一般只能加载到RAM中使用，并进行符号解析绑定到内核导出的API地址上，而不能基于Flash直接以XIP方式运行（因为Flash上也不能够再行修改其中的代码段）。

## 使用动态模块 ##

当要在系统中测试使用动态模块，需要编译一份支持动态模块的固件，以及需要运行的动态模块。下面将固件和动态模块的编译方式分为两部分进行介绍。

### 编译固件 ###

当要使用动态模块时，需要在固件的配置中打开对应的选项，如下配置：

```
   RT-Thread Components  --->
       POSIX layer and C standard library  --->
	       [*] Enable dynamic module with dlopen/dlsym/dlclose feature
```

也需要在bsp对应的rtconfig.py中设置动态模块编译时需要用到的配置参数：

```Python
    M_CFLAGS = CFLAGS + ' -mlong-calls -fPIC '
    M_CXXFLAGS = CXXFLAGS + ' -mlong-calls -fPIC'
    M_LFLAGS = DEVICE + CXXFLAGS + ' -Wl,--gc-sections,-z,max-page-size=0x4' +\
                                    ' -shared -fPIC -nostartfiles -nostdlib -static-libgcc'
    M_POST_ACTION = STRIP + ' -R .hash $TARGET\n' + SIZE + ' $TARGET \n'
    M_BIN_PATH = r'E:\qemu-dev310\fatdisk\root'
```

相关的解释如下：

* M_CFLAGS - 动态模块编译时用到的C代码编译参数，一般此处以PIC方式进行编译（即代码地址支持浮动方式执行）；
* M_CXXFLAGS - 动态模块编译时用到的C++代码编译参数，参数和上面的`M_CFLAGS`类似；
* M_LFLAGS - 动态模块进行链接时的参数。同样是PIC方式，并且是按照共享库方式链接（部分链接）；
* M_POST_ACTIOn - 动态模块编译完成后要进行的动作，这里会对elf文件进行strip下，以减少elf文件的大小；
* M_BIN_PATH - 当动态模块编译成功时，对应的动态模块文件是否需要复制到统一的地方；

基本上来说，ARM9、Cortex-A、Cortex-M系列的这些编译配置参数是一样的。

内核固件也会通过`RTM(function)`的方式导出一些函数API给动态模块使用，这些导出符号可以在msh下通过命令

    list_symbols

列出固件中所有导出的符号信息。`dlmodule`加载器也是把动态模块中需要解析的符号按照这里导出的符号表来进行解析，完成最终的绑定动作。

这段符号表会放在一个专门的，名字是RTMSymTab的section中，所以对应的固件链接脚本也需要保留这块区域，而不会被链接器优化移除掉。可以在链接脚本中添加对应的信息：

```text
        /* section information for modules */
        . = ALIGN(4);
        __rtmsymtab_start = .;
        KEEP(*(RTMSymTab))
        __rtmsymtab_end = .;
```

然后在bsp工程目录下执行`scons`正确无误地生成固件后，在bsp工程目录下执行：

    scons --target=ua -s

来生成编译动态模块时需要包括的内核头文件搜索路径及全局宏定义。

### 建立动态模块 ###

在github上有一份独立仓库： [rtthread-apps](https://github.com/RT-Thread/rtthread-apps) ，这份仓库中放置了一些和动态模块，动态库相关的示例。

其目录结构如下：

| 目录名 | 说明 |
| ----- | ---- |
| cxx | 演示了如何在动态模块中使用C++进行编程 |
| hello | 最简单的`hello world`示例 |
| lib | 动态库的示例 |
| md5 | 为一个文件产生 md5 码 |
| tools | 动态模块编译时需要使用到的Python/SConscript脚本 |
| ymodem | 通过串口以 YModem协议下载一个文件到文件系统上 |

可以把这份git clone到本地，然后在命令行下以scons工具进行编译，如果是Windows平台，推荐使用RT-Thread/ENV工具。

进入控制台命令行后，进入到这个rtthread-apps repo所在的目录（同样的，请保证这个目录所在全路径不包含空格，中文字符等字符），并设置好两个变量：

* RTT_ROOT - 指向到RT-Thread代码的根目录；
* BSP_ROOT - 指向到BSP的工程目录；

Windows下可以使用(假设使用的BSP是qemu-vexpress-a9)：

    set RTT_ROOT=d:\your_rtthread
	set RTT_ROOT=d:\your_rtthread\bsp\qemu-vexpress-a9

来设置对应的环境变量。然后使用如下命令来编译动态模块，例如hello的例子：

    scons --app=hello

编译成功后，它会在rtthread-apps/hello目录下生成hello.mo文件。

也可以编译动态库，例如lib的例子：

    scons --lib=lib

编译成功后，它会在rtthread-apps/lib目录下生成lib.so文件。

我们可以把这些mo、so文件放到RT-Thread文件系统下。在msh下，可以简单的以`hello`命令方式执行`hello.mo`动态模块：

    msh />ls
	Directory /:
	hello.mo            1368
	lib.so              1376
	msh />hello
	msh />Hello, world

调用hello后，会执行hello.mo里的main函数，执行完毕后退出对应的动态模块。其中`hello/main.c`的代码如下：

```c
#include <stdio.h>

int main(int argc, char *argv[])
{
    printf("Hello, world\n");

    return 0;
}
```

## 动态模块相关API ##

除了可以通过msh直接加载并执行动态模块外，也可以在主程序中使用RT-Thread提供的动态模块API来加载或卸载应用模块。

`struct rt_dlmodule *dlmodule_load(const char* pgname);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| pgname | 动态模块的路径 |

**函数返回**

正确加载返回模块指针，否则返回NULL

这个函数从文件系统中加载应用模块到内存中，若正确加载返回该模块的指针。这个函数并不会创建一个线程去执行这个动态模块，仅仅把模块加载到内存中，并解析其中的符号地址。

`struct rt_dlmodule *dlmodule_exec(const char* pgname, const char* cmd, int cmd_size);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| pgname | 动态模块的路径 |
| cmd | 包括动态模块命令自身的命令行字符串 |
| cmd_size | 命令行字符串大小 |

**函数返回**

加载并运行动态模块成功则返回动态模块指针，否则返回NULL

这个函数根据`pgname`路径加载动态模块，并启动一个线程来执行这个动态模块的`main`函数，同时`cmd`会作为命令行参数传递给动态模块的`main`函数入口。

`void dlmodule_exit(int ret_code);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| ret_code | 模块的返回参数 |

**函数返回**

无

这个函数由模块运行时调用，它可以设置模块退出的返回值`ret_code`，然后从模块退出。

`struct rt_dlmodule *dlmodule_find(const char *name);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| name | 模块名称 |

**函数返回**

如果系统中有对应的动态模块，则返回这个动态模块的指针；否则返回NULL。

这个函数以`name`查找系统中是否已经有加载的动态模块。

`struct rt_dlmodule *dlmodule_self(void);`

**函数参数**

无

**函数返回**

返回调用上下文环境下动态模块本身；如果不处于动态模块运行上下文环境内，则返回NULL。

这个函数返回调用上下文环境下动态模块的指针。

`rt_uint32_t dlmodule_symbol_find(const char *sym_str);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| name | 模块名称 |

**函数返回**

如果系统中有对应的动态模块，则返回这个动态模块的指针；否则返回NULL。

这个函数以`name`查找系统中是否已经有加载的动态模块。

## 标准的libdl API ##

在RT-Thread dlmodule中也支持POSIX标准的libdl API，类似于把一个动态库加载到内存中（并解析其中的一些符号信息），由这份动态库提供对应的函数操作集。支持的libdl API包括如下。

libdl API需要包含的头文件

```c
#include <dlfcn.h>
```

`void * dlopen (const char * pathname, int mode);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| pathname | 动态库路径名称 |
| mode | 打开动态库时的模式，在RT-Thread中并未使用 |

**函数返回**

打开成功，返回动态库的句柄指针（实质是`struct dlmodule`结构体指针）；否则返回NULL

这个函数类似`dlmodule_load`的功能，会从文件系统上加载动态库，并返回动态库的句柄指针。

`void*dlsym(void *handle, const char *symbol);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| handle | 动态库句柄，应该是`dlopen`的返回值 |
| symbol | 要返回的符号地址 |

**函数返回**

打开成功，返回对应符号的地址，否则返回NULL。

这个函数在动态库`handle`中查找是否存在`symbol`的符号，如果存在返回它的地址。

`int dlclose (void *handle);`

**函数参数**

| 参数 | 描述 |
| ---- | ---- |
| handle | 动态库句柄 |

**函数返回**

关闭成功，返回0；否则返回负数。

这个函数会关闭`handle`指向的动态库，从内存中卸载掉。需要注意的是，当动态库关闭后，原来通过`dlsym`返回的符号地址将不再可用。如果依然尝试去访问，可能会引起fault错误。