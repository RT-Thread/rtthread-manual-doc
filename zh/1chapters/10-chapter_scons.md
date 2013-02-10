# scons构建系统 #

scons是一个由Python语言编写的构建系统，类似于GNU Make。它采用不同于通常Makefile文
件的SConstruct和SConscript文件。这些文件也是Python脚本，能够使用标准的Python语法
来编写。所以在SConstruct、SConscript文件中可以调用Python标准库进行各类复杂的处理，
而不局限于Makefile设定的规则。

scons本身的用户手册

http://www.scons.org/doc/production/HTML/scons-user/index.html

## 简单的SContruct ##

例如针对一个hello world的简单程序，假设它的源文件是：

    /**
     * file: hello.c
     */
    #include <stdio.h>
    
    int main(int argc, char** argv)
    {
        printf("Hello, world!\n");
    }

只需要在这个文件目录下添加一个SConstruct文件，它的内容是：

    Program('hello.c')

然后在这个目录下执行命令：

    % scons
    scons: Reading SConscript files ...
    scons: done reading SConscript files.
    scons: Building targets ...
    cc -o hello.o -c hello.c
    cc -o hello hello.o
    scons: done building targets.

将会在当前目录下生成hello的应用程序。所以相比于Makefile，一个简单的hello.c --> 
hello的转换，只需要一句话。

如果hello是由两个文件编译而成，也只需要把SConstruct文件修改成：

    Program(['hello.c', 'file1.c'])

同时也可以指定编译出的目标文件名称：

    Program('program', ['hello.c', 'file1.c'])

有的时候也可以偷偷懒，例如把当面目录下的所有C文件都作为源文件来编译：

    Program('program', Glob('*.c'))

Glob函数就是用于使用当前目录下的所有C文件。除了Glob函数以外，也有Split函数。Split
函数写的脚本具备更好的可读性以及更精确的可定制性：

    src = Split('''
        hello.c
        file1.c
        ''')
    Program('program', src)

它的效果与 ``` Program('program', ['hello.c', 'file1.c']) ``` 是一致的，但具有更
清晰的可读性。

## SConstruct与SConscript ##

对于复杂、大型的系统，显然不仅仅是一个目录下的几个文件就可以搞定的，很可能是由数
个文件夹一级级组合而成。

在 scons中，可以编写SConscript脚本文件来编译这些相对独立目录中的文件，同时也可以
使用scons中的Export和Import函数在SConstruct与SConscript文件之间共享数据（也就是
Python中的一个对象数据）。

假设

## RT-Thread构建 ##

### 编写BSP的SConstruct ###

### 编写组件的SConscript ###

### RT-Thread building脚本 ###

在RT-Thread tools目录下存放有RT-Thread自己定义的一些辅助building的脚本，例如用于
自动生成RT-Thread针对一些IDE集成开发环境的工程文件。其中最主要的是building.py脚本

### 进阶 ###
