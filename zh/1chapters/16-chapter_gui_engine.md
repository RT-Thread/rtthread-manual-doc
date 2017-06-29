# 图像用户界面引擎 

## 介绍 ##

GUI engine是一套基本的绘图引擎，由C代码编写而成，代码主要放于`rt-thread/components/gui/`目录中。主要的功能包括：

* 基于绘图设备DC的绘图操作，包括点，线，圆，椭圆，多边形（填充）等；
* 各类图像格式加载（从文件系统中加载，需要DFS文件系统）及绘图；
* 各种字体的文本显示；
* GUI的C/S架构及基础的窗口机制，事件框架机制等。

当需要使用GUI engine时，需要在rtconfig.h中定义：

```c
#define RT_USING_GUIENGINE
```

打开GUI engine。在rtconfig.h中打开GUI engine后，可以使用命令行重新生成工程，或使用scons来进行编译。

## 引擎初始化 ##

在打开GUI engine后，需要在启动时对它进行初始化，把UI的服务打开起来，如果程序中已经使用了组件自动初始化，则不再需要额外进行单独的初始化，否则需要在初始化任务中调用初始化函数：

```c
rtgui_system_server_init();
```

GUI engine初始化后，它会创建基础的GUI服务，包括任务：`rtgui`。虽然GUI engine是基础的引擎，但它依然还带了多窗口系统，绘图主要以GUI应用的模式而存在（创建应用，创建窗口（制定窗口位置，以及大小），在窗口中进行绘图，关闭窗口，删除应用）。

## 绘图设备上下文 ##

为了划分不同的绘图区域（而不是直接和硬件打交道），在GUI engine中抽象出了DC（设备上下文，Device Context），而GUI engine中提供的绘图API基本上都是基于DC的API接口。DC根据不同的面向领域，主要包括这三类DC：

* Hardware DC，硬件设备上下文；
* Client DC，客户端设备上下文（每个窗口，控件都会携带一个Client DC）；
* Buffer DC，基于软件缓冲区的软件虚拟设备上下文；

## 绘图渲染 ##

基于DC，UI引擎提供了基本的绘图操作，例如画点、线、矩形、圆、多边形等。这里统一的操作设备是DC，例如：

```c
void rtgui_dc_draw_line(struct rtgui_dc *dc, int x1, int y1, int x2, int y2);
```

这个函数是在dc上绘制一条从 `(x1, y1) - (x2, y2)` 的线。线的颜色则是dc的图形上下文（gc, graphic context）中定义的前景色。

除了基本的绘图操作以外，GUI引擎也提供了DC上的字体文本绘制，例如：

```c
void rtgui_dc_draw_text(struct rtgui_dc *dc, const char *text, struct rtgui_rect *rect);
```

这个函数就是在dc上，在由rect参数定义区域内显示text文本。文本颜色、字体依然是由图形上下文来定义的，并且图形上下文也定义了文本的对齐方式textalign，对齐方式取值是在RTGUI_ALIGN枚举类型中定义。

```c
enum RTGUI_ALIGN
{
    RTGUI_ALIGN_NOT               = 0x00,
    RTGUI_ALIGN_CENTER_HORIZONTAL = 0x01,
    RTGUI_ALIGN_LEFT              = RTGUI_ALIGN_NOT,
    RTGUI_ALIGN_TOP               = RTGUI_ALIGN_NOT,
    RTGUI_ALIGN_RIGHT             = 0x02,
    RTGUI_ALIGN_BOTTOM            = 0x04,
    RTGUI_ALIGN_CENTER_VERTICAL   = 0x08,
    RTGUI_ALIGN_CENTER            = RTGUI_ALIGN_CENTER_HORIZONTAL | RTGUI_ALIGN_CENTER_VERTICAL,
    RTGUI_ALIGN_EXPAND            = 0x10,
    RTGUI_ALIGN_STRETCH           = 0x20,
};
```

对于一个DC而言，可以通过以下方式获得DC的图形上下文：

```c
rtgui_gc_t *rtgui_dc_get_gc(struct rtgui_dc *dc);
```

也可以通过以下简单的方式来访问DC的前景、背景、字体、对齐方式等信息：

```c
#define RTGUI_DC_FC(dc)         (rtgui_dc_get_gc(RTGUI_DC(dc))->foreground)
#define RTGUI_DC_BC(dc)         (rtgui_dc_get_gc(RTGUI_DC(dc))->background)
#define RTGUI_DC_FONT(dc)       (rtgui_dc_get_gc(RTGUI_DC(dc))->font)
#define RTGUI_DC_TEXTALIGN(dc)  (rtgui_dc_get_gc(RTGUI_DC(dc))->textalign)
```

同样的，一幅图像也可以在DC上渲染出来，例如使用以下API：

```c
void rtgui_image_blit(struct rtgui_image *image, struct rtgui_dc *dc, struct rtgui_rect *rect);
```

这个API用于把一副image图像，绘制到dc的rect区域上，如果rect区域超出dc的范围，则会被自动裁剪。

## 基本的GUI引擎应用例子 ##

```c
```

## 事件传递机制 ##

## 控件和剪切域 ##

## 字体API ##

## 图像API ##
