# WDT驱动

## 1.简介
下文介绍的WDT驱动主要是依赖于IO设备框架和watchdog驱动框架。

## 2.在工程中添加watchdog框架

**2.1.添加看门狗宏**

首先,在编写WDT驱动之前我们先再rtconfig.h配置文件中添加以下宏来开启工程对watchdog驱动框架支持：
~~~
#define RT_USING_WDT
~~~
在添加完成以上宏之后，需要使用scons重新生成工程，这里需要根据用者根据直接的平台选择对应的生成工程根据,切换到对应工程根目录下打开cmd命令行执行以下命令：

**2.2.生成工程**

~~~
1. scons --target=mdk4		/* for mdk4 project */
2. scons --target=mdk5		/* for mdk5 project */
3. scons --target=iar		/* for iar project */
~~~
执行完生成工程命令之后，scons就帮我们将watchdog驱动框架相关的代码添加到了工程中，接下来就直接编写代码。

## 3.介绍watchdog框架
在编写WDT驱动之前，先来讲解一下WDT框架，主要**从数据结构**、**代码框架**和**函数调用关系**三个方面来讲解。

**3.1.数据结构**

在wdt驱动中有两个比较重要的结构体:

**3.1.1 struct rt_watchdog_device**

struct **rt_watchdog_device**结构体是从**rt_device**结构体继承而来。rt_device主要是底层IO设备框架需要的数据结构，这里我们不探究rt_device结构体。具体结构体定义如下：
~~~
struct rt_watchdog_device
{
    struct rt_device parent;		/* 继承rt_device */
    struct rt_watchdog_ops *ops;		/* 看门狗底层操作函数 */
};
typedef struct rt_watchdog_device rt_watchdog_t;
~~~

**3.1.2 struct rt_watchdog_ops**

我们可以看见在rt_watchdog_device数据结构中除了rt_device结构体，还有**struct rt_watchdog_ops**结构体。该结构体主要是用来关联底层wdt驱动操作看门狗外设的接口，具体结构体定义如下：
~~~
struct rt_watchdog_ops
{
    rt_err_t (*init)(rt_watchdog_t *wdt);				/* 初始化看门狗 */
    rt_err_t (*control)(rt_watchdog_t *wdt, int cmd, void *arg);	/* 控制看门狗 */
};
~~~
根据结构体可以知道，编写WDT驱动需要实现初始化看门狗和控制看门狗两个接口。对应控制看门狗接口，需要对以下几个命令进行实现：
~~~
1. RT_DEVICE_CTRL_WDT_GET_TIMEOUT		/* 获取超时时间 */
2. RT_DEVICE_CTRL_WDT_SET_TIMEOUT		/* 设置超时时间 */
3. RT_DEVICE_CTRL_WDT_GET_TIMELEFT		/* 获取超时剩余时间 */
4. RT_DEVICE_CTRL_WDT_KEEPALIVE		/* 喂狗*/
5. RT_DEVICE_CTRL_WDT_START		/* 驱动看门狗 */
6. RT_DEVICE_CTRL_WDT_STOP		/* 停止看门狗 */
~~~

对于以上看门狗控制命令**RT_DEVICE_CTRL_WDT_KEEPALIVE**必须实现，其他命令可以选择实现。

**3.2.代码框架与函数调用关系**

wdt整体框架与函数调用关系如下图：
![image](https://github.com/enkiller/rtthread-manual-doc/blob/master/zh/3_chapters/wdt.png)


