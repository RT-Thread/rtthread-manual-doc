# 简介 #

本手册是RT-Thread开源实时嵌入式操作系统的手册。

## RT-Thread 的软件结构 ##

![RT-Thread软件结构](figures/System_Arch.png)

RT-Thread实时操作系统是一个分层的操作系统，它包括了：

* 底层移植、驱动层，这层与硬件密切相关，由Drivers和CPU移植相构成。
* 硬实时内核，这层是RT-Thread的核心，包括了内核系统中对象的实现，例如多线程及其调度，信号量，邮箱，消息队列，内存管理，定时器等实现。
* 组件层，这些是基于RT-Thread核心基础上的外围组件，例如文件系统，命令行shell接口，lwIP轻型TCP/IP协议栈，GUI图形用户界面等。

