# SCons构建系统 #

SCons是一套由Python语言编写的开源构建系统，类似于GNU Make。它采用不同于通常Makefile文件的SConstruct和SConscript文件。这些文件也是Python脚本，能够使用标准的Python语法来编写。所以在SConstruct、SConscript文件中可以调用Python标准库进行各类复杂的处理，而不局限于Makefile设定的规则。

在[SCons的网站](http://www.scons.org/doc/production/HTML/scons-user/index.html)上可以找到详细的SCons用户手册，本章节讲述SCons的基本用法，以及如何在RT-Thread中用好SCons工具。

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

目录下，可以把 C:\\Python27\\Scripts 目录加入到PATH环境变量中。在Windows的我的电脑中，右键把系统属性设置窗口点出来，如下图所示：

![我的电脑系统属性设置](figures/scons_SettingENV1.png)

点击其中的高级设置

![修改PATH环境变量](figures/scons_SettingENV2.png)

选择PATh项，然后点击编辑按钮，然后把C:\\Python27\\Scripts目录添加到PATH最后的位置。添加完成后，可以按Win键+R，然后输入cmd回车打开Windows命令行窗口，在其中输入：

    scons

如果能够见到下面的输出，说明Python和SCons安装正确。

![SCons命令行输出](figures/scons_CMD.png)

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

它的效果与 ``` Program('program', ['hello.c', 'file1.c']) ``` 是一致的，但具有更
清晰的可读性。

## SConstruct与SConscript ##

对于复杂、大型的系统，显然不仅仅是一个目录下的几个文件就可以搞定的，很可能是由数个文件夹一级级组合而成。

在 SCons中，可以编写SConscript脚本文件来编译这些相对独立目录中的文件，同时也可以使用SCons中的Export和Import函数在SConstruct与SConscript文件之间共享数据（也就是Python中的一个对象数据）。

假设

## RT-Thread构建 ##

RT-Thread的历史中，最早依然是使用Makefile来进行构建，但当RT-Thread需要支持多个CPU时发现Makefile在配置上会非常麻烦，稍有不甚极容易出错。而再加上各类组件后，使用哪些组件，不使用哪些组件，等等配置后，使用Makefile变成更加复杂。当要支持不同的交叉编译环境时，Linux、Windows等不同Host PC主机时，就更加难上加难。从0.3.x开始，RT-Thread开发团队逐渐引入了SCons构建系统，引入SCons唯一的目标是：使大家从复杂的Makefile配置、IDE配置中脱离出来，把精力集中在代码的本身开发上。

### 编写BSP的SConstruct ###

### 编写组件的SConscript ###

### RT-Thread building脚本 ###

在RT-Thread tools目录下存放有RT-Thread自己定义的一些辅助building的脚本，例如用于自动生成RT-Thread针对一些IDE集成开发环境的工程文件。其中最主要的是building.py脚本

### 进阶 ###
