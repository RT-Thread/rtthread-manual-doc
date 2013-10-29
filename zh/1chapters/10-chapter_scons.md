# SCons构建系统 #

SCons是一套由Python语言编写的开源构建系统，类似于GNU Make。它采用不同于通常Makefile文件的SConstruct和SConscript文件。这些文件也是Python脚本，能够使用标准的Python语法来编写。所以在SConstruct、SConscript文件中可以调用Python标准库进行各类复杂的处理，而不局限于Makefile设定的规则。

在[SCons的网站](http://www.scons.org/doc/production/HTML/scons-user/index.html)上可以找到详细的SCons用户手册，本章节讲述SCons的基本用法，以及如何在RT-Thread中用好SCons工具。

## 什么是构建工具(software construction tool) ##

构建工具是一种软件，它可以根据一定的规则或指令，将源代码编译成可执行的二进制程序。这是构建工具最基本也是最重要的功能。实际上，构建工具的功能不至于此，通常这些规则有一定的语法，并组织成文件。这些文件用于来控制构建工具的行为，在完成软件构建之外，也可以做其他事情。

目前最流行的构建工具是Make。很多知名开源软件，如Linux内核就采用Make构建。Make通过读取Makefile文件来检测文件的组织结构和依赖关系，并完成Makefile中所指定的命令。

由于历史原因，Makefile的语法比较混乱,不利于初学者学习。此外，在Windows平台上使用Make也不方便，需要安装Cygwin环境。为了克服Make的种种缺点，人们开发了其他构建工具，如CMak和SCons等。

## RT-Thread构建 ##

RT-Thread早期使用Make/Makefile构建。从0.3.x开始，RT-Thread开发团队逐渐引入了SCons构建系统，引入SCons唯一的目是：使大家从复杂的Makefile配置、IDE配置中脱离出来，把精力集中在RT-Thread功能开发上。

有些读者可能会有些疑惑，这里介绍的构建工具有IDE有什么不同。

通常IDE有自己的管理源码的方式，一些IDE使用XML来组织文件，并解决依赖关系。大部分IDE会根据用户所添加的源码生成类似Makefile与SConscript的脚本文件，在底层调用类似Make与SCons的工具来构建源码。IDE通过可以图形化的操作来完成构建。

## 安装SCons环境 ##

在使用SCons系统前需要在PC主机中安装它，因为它是Python语言编写的，所以在之前需要安装Python语言环境。需要注意的是，由于目前SCons还不支持Python 3.x，所以需要安装Python 2.x环境。

### Linux、BSD环境 ###

在Linux、BSD环境中Python应该是已经默认安装了，一般也是2.x版本系列的Python环境。这时只需要安装SCons即可，例如在Ubuntu中可以使用如下命令：

    sudo apt-get install scons

### Windows环境 ###

请到[Python网站](http://www.python.org/getit/)下载Python 2.x系列安装包，当前推荐使用Python 2.7.x系列的Python版本。

请到[SCons网站](http://www.scons.org/)下载SCons安装包，从RT-Thread使用经验来看，SCons的各个版本（1.0.0 - 2.3.x）都可以在RT-Thread上正常使用

在Windows下安装完成Python 和 SCons后，需要把scons命令添加到系统的PATH环境变量中，假设Python默认安装在

    C:\Python27

目录下，可以把```C:\\Python27\\Scripts```目录加入到PATH环境变量中。在Windows的我的电脑中，右键把系统属性设置窗口点出来，如下图所示：

![我的电脑系统属性设置](figures/scons_SettingENV1.png)

点击其中的高级设置

![修改PATH环境变量](figures/scons_SettingENV2.png)

选择PATh项，然后点击编辑按钮，然后把C:\\Python27\\Scripts目录添加到PATH最后的位置。添加完成后，可以按Win键+R，然后输入cmd回车打开Windows命令行窗口，在其中输入：

    scons

如果能够见到下面的输出，说明Python和SCons安装正确。

![SCons命令行输出](figures/scons_CMD.png)

## SCons基本使用 ##

初次使用SCons编译某个bsp之前，需要先为bsp指定编译器。这需要修改该bsp目录下的rtconfig.py文件。

### 配置编译器 ###

rtconfig.py是一个RT-Thread标准的编译器配置文件，主要用于完成以下工作：

- 指定编译器（可以支持多个编译器）
- 指定编译器参数，如编译选项、链接选线等

首先确保你的系统上已经安装了编译器。RT-Thread构建系统支持多种编译器。目前支持的编译器包括arm gcc，MDK，IAR，VisualStudio，Visual DSP。主流的ARM Cortex M0、M3、M4平台，基本上ARM GCC、MDK、IAR都是支持的。有一些bsp可能仅支持一种，读者可以阅读该bsp目录下的rtconfig.py查看当前支持的编译器。

这里以bsp/stm32f10x 为例，其rtconfig.py如下所示

~~~{.py}
ARCH='arm'
CPU='cortex-m3'
CROSS_TOOL='keil'
 
if  CROSS_TOOL == 'gcc':
    PLATFORM    = 'gcc'
    EXEC_PATH   = r'D:\SourceryGCC\bin'
elif CROSS_TOOL == 'keil':
    PLATFORM    = 'armcc'
    EXEC_PATH   = r'C:\Keil'
elif CROSS_TOOL == 'iar':
    PLATFORM    = 'iar'
    IAR_PATH    = r'E:/Program Files/IAR Systems/Embedded Workbench 6.0'
 .....
~~~

一般来说，我们只需要修改CROSS_TOOL和下面的EXEC_PATH两个选项。

- CROSS_TOOL 

编译器名字，可选的值为```'keil'```, ```'gcc'```,```'iar'```。大致浏览rtconfig.py查看当前bsp所支持的编译器。

- EXEC_PATH

编译器的安装路径。

如果您的机器上安装了MDK，那么将CROSS_TOOL修改为```'keil'```，并修改```EXEC_PATH = r'C:/Keil'```为您的MDK的安装路径。

这里有两点需要注意：

1. 安装编译器时（如MDK，ARM GCC，IAR等），不要安装到带有中文或者空格的路径中。否则，某些解析路径时会出现错误。有些程序默认会安装到```C:\Program Files```目录下，中间带有空格。建议安装时选择其他路径，养成良好的开发习惯。
2. 修改EXEC_PATH时，需要注意路径的格式。在windows平台上，默认的路径分割符号是反斜杠```\```,而这个符号在C语言以及Python中都是用于转移字符的。所以修改路径时，可以将```\```改为```/```，或者在前面加r（python特有的语法）。

假如某编译器安装位置为```D:\Dir1\Dir2```下。下面几种是正确的写法:

    EXEC_PATH = r'D:\Dir1\Dir2'  注意，字符串前带有r，则可正常使用“\”
    EXEC_PATH = 'D:/Dir1/Dir2'   注意，改用“/”，前面没有r
    EXEC_PATH = 'D:\\Dir1\\Dir2' 注意，这里使用“\”的转义性来转义“\”自己。

下面是错误的写法

    EXEC_PATH = 'D:\Dir1\Dir2'

编译器配置完成之后，我们就可以使用SCons来编译RT-Thread的bsp了。

在当前目录打开命令行窗口，执行scons.就会启动编译。

小技巧：
在WIN7上，在当前目录按下SHIFT同时点击鼠标右键，弹出的菜单中，会有“在此处打开命令窗口”的菜单项。点击可以快速打开CMD窗口。

### SCons基本命令 ###

本节介绍RT-Thread中常用SCons命令。SCons不仅完成基本的编译，还可以生成MDK/IAR/VS工程。

#### scons ####

编译目标。如果执行过scons后修改一些文件，再次执行scons则SCons会增量编译，仅编译修改过的文件并链接。

#### scons -jN ####

多线程编译目标，在多核计算机上可以加快编译速度。一般来说，一颗cpu核心可以支持2个线程线程。双核机器上使用-j4即可。

    scons -j4

#### scons -c ####

清除编译目标。这个命令会清除执行scons时生成的临时文件和目标文件。

#### scons --target=XXX -s ####

    scons --target=mdk4 -s

可以在当前目录生成一个新的名为project.uvproj文件。双击它打开，就可以使用MDK来编译、调试。不习惯SCons的同学可以使用这种方式。

当修改了rtconfig.h打开或者关闭某些组件时，也需要使用这个命令重新生成工程。 

注意：

不要试图打开template.uvproj的文件，这个文件仅是一个模板文件，用于辅助SCons生成project.uvproj。

如果打开project.uvproj失败，则删除project.uvopt，然后重新打开project.uvproj。

    scons --target=iar -s

自动生成IAR工程

    scons --target=vs2012 -s
    Scons --target=vs2005 -s

在bsp/simulator下，可以使用这个命令下生成vs2012的工程或vs2005的工程。

#### scons --verbose ####

默认情况下，scons编译的输出不会显示编译参数，如下所示：

    F:\Project\git\rt-thread\bsp\stm32f10x>scons
    scons: Reading SConscript files ...
    scons: done reading SConscript files.
    scons: Building targets ...
    scons: building associated VariantDir targets: build
    CC build\applications\application.o
    CC build\applications\startup.o
    CC build\components\drivers\serial\serial.o
    ...

使用scons --verbose的效果

    armcc -o build\src\mempool.o -c --device DARMSTM --apcs=interwork -ID:/Keil/ARM/
    RV31/INC -g -O0 -DUSE_STDPERIPH_DRIVER -DSTM32F10X_HD -Iapplications -IF:\Projec
    t\git\rt-thread\applications -I. -IF:\Project\git\rt-thread -Idrivers -IF:\Proje
    ct\git\rt-thread\drivers -ILibraries\STM32F10x_StdPeriph_Driver\inc -IF:\Project
    \git\rt-thread\Libraries\STM32F10x_StdPeriph_Driver\inc -ILibraries\STM32_USB-FS
    -Device_Driver\inc -IF:\Project\git\rt-thread\Libraries\STM32_USB-FS-Device_Driv
    er\inc -ILibraries\CMSIS\CM3\DeviceSupport\ST\STM32F10x -IF:\Project\git\rt-thre
    ad\Libraries\CMSIS\CM3\DeviceSupport\ST\STM32F10x -IF:\Project\git\rt-thread\com
    ponents\CMSIS\Include -Iusb -IF:\Project\git\rt-thread\usb -I. -IF:\Project\git\
    rt-thread -IF:\Project\git\rt-thread\include -IF:\Project\git\rt-thread\libcpu\a
    rm\cortex-m3 -IF:\Project\git\rt-thread\libcpu\arm\common -IF:\Project\git\rt-th
    read\components\drivers\include -IF:\Project\git\rt-thread\components\drivers\in
    clude -IF:\Project\git\rt-thread\components\finsh -IF:\Project\git\rt-thread\com
    ponents\init F:\Project\git\rt-thread\src\mempool.c
    ...

## SCons进阶 ##

### 修改编译器选项 ###

### 增加文件 ###

### 添加库 ###

### 生成库 ###

### 配置其他参数 ###


## SCons高级 ##

### 编写BSP的SConstruct ###

### 编写组件的SConscript ###

### 增加一个SCons命令 ###

### RT-Thread building脚本 ###

在RT-Thread tools目录下存放有RT-Thread自己定义的一些辅助building的脚本，例如用于自动生成RT-Thread针对一些IDE集成开发环境的工程文件。其中最主要的是building.py脚本。

## 简单的SContruct ##

例如针对一个hello world的简单程序，假设它的源文件是：

~~~{.c}
/* file: hello.c */
#include <stdio.h>

int main(int argc, char** argv)
{
    printf("Hello, world!\n");
}
~~~

只需要在这个文件目录下添加一个如下内容的SConstruct文件：

    Program('hello.c')

然后在这个目录下执行命令：

    % scons
    scons: Reading SConscript files ...
    scons: done reading SConscript files.
    scons: Building targets ...
    cc -o hello.o -c hello.c
    cc -o hello hello.o
    scons: done building targets.

将会在当前目录下生成hello的应用程序。所以相比于Makefile，一个简单的hello.c到hello的转换，只需要一句话。如果hello是由两个文件编译而成，也只需要把SConstruct文件修改成：

    Program(['hello.c', 'file1.c'])

同时也可以指定编译出的目标文件名称：

    Program('program', ['hello.c', 'file1.c'])

有的时候也可以偷偷懒，例如把当面目录下的所有C文件都作为源文件来编译：

    Program('program', Glob('*.c'))

Glob函数就是用于使用当前目录下的所有C文件。除了Glob函数以外，也有Split函数。Split函数写的脚本具备更好的可读性以及更精确的可定制性：

    src = Split('''
        hello.c
        file1.c
        ''')
    Program('program', src)

它的效果与 ``` Program('program', ['hello.c', 'file1.c']) ``` 是一致的，但具有更清晰的可读性。

## SConstruct与SConscript ##

对于复杂、大型的系统，显然不仅仅是一个目录下的几个文件就可以搞定的，很可能是由数个文件夹一级级组合而成。

在 SCons中，可以编写SConscript脚本文件来编译这些相对独立目录中的文件，同时也可以使用SCons中的Export和Import函数在SConstruct与SConscript文件之间共享数据（也就是Python中的一个对象数据）。
