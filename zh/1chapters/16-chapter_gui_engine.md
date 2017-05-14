# 图像用户界面引擎 

## 介绍 ##

GUI engine是一套基本的绘图引擎，由C代码编写而成，代码主要放于`rt-thread/components/gui/`目录中。主要的功能包括：

* （基于绘图设备DC的）绘图，包括点，线，圆，椭圆，多边形（填充）等；
* 各类图像格式加载（从文件系统中加载，需要DFS）及绘图；
* 各种字体的文本显示；
* GUI的C/S架构，及基础的事件框架机制

当需要使用GUI engine时，需要在rtconfig.h中定义：

    #define RT_USING_GUIENGINE

打开GUI engine。

在rtconfig.h中打开GUI engine后，可以使用命令行重新生成工程，或使用scons来进行编译。

## 引擎初始化 ##

在打开GUI engine后，需要在启动时对它进行初始化，把UI的服务打开起来，如果程序中已经使用了组件自动初始化，则不再需要额外进行单独的初始化，否则需要在初始化任务中执行：

    rtgui_system_server_init();

GUI engine初始化后，它会创建基础的UI服务，包括任务：`rtgui`。虽然GUI engine是基础的引擎，但它依然还带了多窗口系统，绘图主要以GUI应用的模式而存在（创建应用，创建窗口（制定窗口位置，以及大小），在窗口中进行绘图，关闭窗口，删除应用）。

## 绘图设备上下文 ##

为了划分不同的绘图区域（而不是直接和硬件打交道），在GUI engine中抽象出了DC（设备上下文，Device Context），而GUI engine中提供的绘图API基本上都是基于DC的API接口。DC根据不同的面向领域，主要包括这三类DC：

* Hardware DC，硬件设备上下文；
* Client DC，客户端设备上下文（每个窗口，控件都会携带一个Client DC）；
* Buffer DC，基于软件缓冲区的软件虚拟设备上下文；

## 绘图渲染 ##

[基本绘图]

[字体]

[图像]

## 事件传递机制 ##

## 控件和剪切域 ##

## 字体API ##

## 图像API ##
