# 应用模块 #

在传统桌面操作系统中，用户空间和内核空间是分开的，应用程序运行在用户空间，内核以及内核模块则运行于内核空间，其中内核模块可以动态加载与删除以扩展内核功能，而在小型嵌入式设备领域，通常并不区分内核态与用户态，并且整个系统通常编译成一个单独的固件下载到单片机芯片的Flash中。自RT-Thread 0.4.0版开始引入了一种称为RT-Thread application module（应用模块）的技术，它提供了一种动态加载或卸载应用程序的功能，应用程序可以独立编译，并存储外部存储介质上，如SD卡、SPI Flash，甚至可以通过网络传输。但RT-Thread依然没有区分用户空间与内核空间，应用模块兼顾应用程序和内核模块的属性，是两者的结合体，所以称作应用模块。为书写简单，下文将RT-Thread application module简称为应用模块或模块。

[TODO] 更新使用应用模块为使用[rtthread-apps](https://github.com/RT-Thread/rtthread-apps)的方式 ；加入RTM_EXPORT宏描述。

## 功能和限制 ##

应用模块为RT-Thread提供一种类似桌面系统安装卸载应用程序的功能，功能十分灵活。从实现上讲，这是一种将内核和应用分开的机制，通过这种机制，内核和应用可以分开编译，并在运行时通过内核中的模块加载器将编译好的应用加载到内核中运行。

当前RT-Thread支持应用模块的架构包括ARM7、ARM9、Cortex-M3/M4/M7。RT-Thread内核固件部分可使用多种编译器工具链，如GCC, ARMCC、IAR等工具链；但应用模块部分编译只支持GCC工具链。因此编译RT-Thread模块需下载GCC工具，例如CodeSourcery的arm-none-eabi工具链。

应用模块也有一定限制，它仅支持加载到RAM中运行（因为需要在内存中把符号进行绝对化固定下来），而不能直接在flash上运行，因此，耗费的RAM会更多一些（在Flash上直接XIP执行的Cortex-M处理器不见得合适）。

## 使用应用模块 ##

要想在板子测试使用应用模块，需要编译一个支持应用模块的RT-Thread主程序以及独立编译的应用模块程序。下面将分为两部分介绍。

### 编译主程序 ###

在rtconfig.h中打开如下宏（如果不存在则手动添加）

    #define RT_USING_MODULE

然后重新编译主工程。读者需要参考SCons构建系统那一章学习如何编译RT-Thread源代码。将编译好的主程序下载到芯片中运行。

注意：

1. 如果是手动创建RT-Thread的MDK工程，需要在链接选项中应加入`--keep __rtmsym_*`参数
2. 某些分支bsp目录下的startup.c可能未加入`rt_module_system_init()`函数

### 使用应用模块 ###

RT-Thread源码中提供了几个应用模块的基本例子，它们位于RT-Thread源码树的example/module目录下，目前该目录下共有如下目录和文件

- example/module/basicapp/  一个简单的应用模块工程
- example/module/tetris/    一个使用RTGUI的俄罗斯方块的应用模块工程
- example/module/SConstruct  应用模块工程的构建脚本
- example/module/rtconfig.py 应用模块工程的配置脚本，需要配置工具链路径和bsp，默认为mini2440分支
- example/module/rtconfig_lm3s.py lm3s8962分支的应用工程配置脚本模板， 使用时需更名为rtconfig.py
- example/module/README 使用说明

这里以stm32f10x bsp为例，编译一个stm32的应用模块程序，起名为test。假如工作目录为D:/work，复制以下文件或目录到work目录下，

- $RTT_ROOT/example/module/basicapp/ `-->` work/basicap/ `-->` work/test
- $RTT_ROOT/example/module/SConstruct `-->` work/SConstruct

并将work目录下的basicapp重命名为test。

由于stm32f10x属于arm-cortex M3架构，因此复制rtconfig_lm32s.py到work目录下，并重命名为rtconfig.py，如下所示

- $RTT_ROOT/example/module/rtconfig_lm3s.py `-->` work/rtconfig.py

打开work/rtconfig.py文件，并做如下修改

1. 修改BSP为stm32f10x
2. 首先确保ARM GCC已经安装，并修改EXEC_PATH为你的编译器路径

在笔者的机器上，修改后的work/rtconfig.py如下所示：

	# bsp name
	BSP = 'stm32f10x'
	
	# toolchains
	EXEC_PATH 	= r'C:\Program Files (x86)\CodeSourcery\Sourcery_CodeBench_Lite_for_ARM_EABI\bin'
	PREFIX = 'arm-none-eabi-'
	CC = PREFIX + 'gcc'
	CXX = PREFIX + 'g++'
	AS = PREFIX + 'gcc'
	AR = PREFIX + 'ar'
	LINK = PREFIX + 'gcc'
	TARGET_EXT = 'so'
	SIZE = PREFIX + 'size'
	OBJDUMP = PREFIX + 'objdump'
	OBJCPY = PREFIX + 'objcopy'
	
	DEVICE = ' -mcpu=cortex-m3'
	CFLAGS = DEVICE + ' -mthumb -mlong-calls -Dsourcerygxx -O0 -fPIC'
	AFLAGS = ' -c' + DEVICE + ' -x assembler-with-cpp'
	LFLAGS = DEVICE + ' -mthumb -Wl,-z,max-page-size=0x4 -shared -fPIC -e main -nostdlib'
	
	CPATH = ''
	LPATH = ''

在work目录下打开命令行，执行如下命令编译应用模块

	scons --app=test

如果没有错误，会在work/test下生成build/stm32f10x/目录，其中test.so为RT-Thread应用模块。

将test.so拷贝到SD或其他设备中。然后在开发板中运行第一步编译的支持了RT-Thread应用模块的主程序。加入test.so已经被置于SD卡，并且将SD卡挂载到RT-Thread根目录下，则在finsh Shell中运行
	
	finsh>>exec("/test.so")

可以看到如下效果，test.so正确运行。

    Hello RT-Thread 1 101
    Hello RT-Thread 2 102
    Hello RT-Thread 3 103
    Hello RT-Thread 4 104
    Hello RT-Thread 5 105
    Hello RT-Thread 6 106
    Hello RT-Thread 7 107
    Hello RT-Thread 8 108
    Hello RT-Thread 9 109
    Hello RT-Thread 10 110
    Hello RT-Thread 11 111
    Hello RT-Thread 12 112
    Hello RT-Thread 13 113
    ……

读者也可以尝试example/module目录下的其他应用模块示例，也可以自己编写应用模块程序。尽情享受应用模块带来的灵活吧~，限制你的只有想象力！

## 应用模块API ##

除了可以通过finsh手动加载应用模块外，也可以在主程序中使用RT-Thread提供的应用模块API来加载或卸载应用模块。

	rt_module_t rt_module_open(const char *path)

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          path  模块完整路径名；
-----------------------------------------------------------------------

**函数返回**

正确加载返回模块指针，否则返回NULL

这个函数从文件系统中加载应用模块到内存中运行，若正确加载返回该模块的指针。

	rt_module_t rt_module_find(const char *name)

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
          name  模块名；
-----------------------------------------------------------------------

**函数返回**

如果找到则返回模块指针，否则返回NULL

这个函数根据模块名查找系统已加载的模块，若找到返回该模块的指针。

	rt_err_t rt_module_destroy(rt_module_t module)

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
        module  模块指针；
-----------------------------------------------------------------------

**函数返回**

成功返回RT_EOK ；失败返回-RT_ERROR。

这个函数会销毁应用模块占用的RT-Thread内核对象，如信号量、互斥量、mempool等，如果它使用了的话。

	rt_err_t rt_module_unload(rt_module_t module)

**函数参数**


-----------------------------------------------------------------------
          参数  描述
--------------  -------------------------------------------------------
        module  模块指针；
-----------------------------------------------------------------------

**函数返回**

成功返回RT_EOK ；失败返回-RT_ERROR。

这个函数会销毁应用模块的线程以及子线程。
