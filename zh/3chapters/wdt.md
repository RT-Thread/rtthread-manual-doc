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
![image](https://github.com/liu2guang/rtthread-manual-doc/tree/master/zh/3chapters/wdt.png)

## 4.编写wdt驱动程序

这里我们用stm32f103的独立看门狗iwdg移植来讲解：

移植平台：stm32f103RE
看门狗外设：iwdg

**4.1.声明平台相关数据结构体IWDG_HandleTypeDef hiwdg**
~~~
IWDG_HandleTypeDef hiwdg;
~~~

**4.2.声明结构体rt_watchdog_t**
~~~
rt_watchdog_t iwdg_device;
~~~

**4.3.编写WDT硬件接口函数 - 初始化看门狗**
~~~
static rt_err_t stm32_iwdg_init(rt_watchdog_t *wdt)
{
	HAL_StatusTypeDef status;

	RT_ASSERT(wdt != RT_NULL);
	
	hiwdg.Instance       = IWDG;
	hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
	hiwdg.Init.Reload    = 4000;	/*3~5s*/

	status = HAL_IWDG_Init(&hiwdg);
	if(status)
	{
		rt_kprintf("HAL_IWDG_Init error\n");
		return RT_EIO;
	}

	return RT_EOK;
}
~~~

**4.4.编写WDT硬件接口函数 - 启动看门狗**

~~~
static rt_err_t stm32_iwdg_open(rt_watchdog_t *wdt, rt_uint16_t oflag)
{
	HAL_StatusTypeDef status;
	
	RT_ASSERT(wdt != RT_NULL);

	status = HAL_IWDG_Start(&hiwdg);
	if(status)
	{
		rt_kprintf("HAL_IWDG_Start error\n");
		return RT_EIO;
	}	

	return RT_EOK;
}
~~~

**4.5.编写WDT硬件接口函数 - 关闭看门狗**

在stm32平台上独立看门狗启动了是无法关闭的，所以这里没有对独立看门狗有任何操作，其他平台可以直接添加关闭看门狗的操作代码。

~~~
static rt_err_t stm32_iwdg_close(rt_watchdog_t *wdt)
{
	RT_ASSERT(wdt != RT_NULL);

	rt_kprintf("[ERR] stm32 iwdg cannot close\n");

	return RT_EIO;
}
~~~

**4.6.编写WDT硬件接口函数 - 控制看门狗**

这里控制看门狗主要是对获取超时时间/设置超时时间/或者剩余复位时间/喂狗/启动看门狗/关闭看门狗，除了喂狗是必须实现的命令之外，其他命令可以选择实现，这里我们只实现了喂狗命令：

~~~
static rt_err_t stm32_iwdg_control(rt_watchdog_t *wdt, int cmd, void *args)
{
	HAL_StatusTypeDef status;
	
	RT_ASSERT(wdt != RT_NULL);

	switch(cmd)
	{
		case RT_DEVICE_CTRL_WDT_GET_TIMEOUT:
		{
			rt_kprintf("[ERR] RT_DEVICE_CTRL_WDT_GET_TIMEOUT do not implement\n");
		}
		break;

		case RT_DEVICE_CTRL_WDT_SET_TIMEOUT:
		{
			rt_kprintf("[ERR] RT_DEVICE_CTRL_WDT_SET_TIMEOUT do not implement\n");
		}
		break;

		case RT_DEVICE_CTRL_WDT_GET_TIMELEFT:
		{
			rt_kprintf("[ERR] RT_DEVICE_CTRL_WDT_GET_TIMELEFT do not implement\n");
		}
		break;

		case RT_DEVICE_CTRL_WDT_KEEPALIVE:
		{
			status = HAL_IWDG_Refresh(&hiwdg);
			if(status)
			{
				rt_kprintf("HAL_IWDG_Refresh error\n");
				return RT_EIO;
			}				
		}
		break;

		case RT_DEVICE_CTRL_WDT_START:
		{
			status = HAL_IWDG_Start(&hiwdg);
			if(status)
			{
				rt_kprintf("HAL_IWDG_Start error\n");
				return RT_EIO;
			}	
		}
		break;

		case RT_DEVICE_CTRL_WDT_STOP:
		{
			stm32_iwdg_close(wdt);
		}
		break;

		default:
		break;
	}

	return RT_EOK;
}
~~~

**4.7.向RTT注册WDT设备，初始化并打开WDT设备，并创建喂狗线程**

~~~
/* 喂狗线程 */
void feeddog_thread(void *parameter)
{
	while(1)
	{
		stm32_iwdg_control(&iwdg_device, RT_DEVICE_CTRL_WDT_KEEPALIVE, RT_NULL);
		
		rt_thread_delay(RT_TICK_PER_SECOND);
	}
}

/* 向RTT注册WDT设备，初始化并打开WDT设备，并创建喂狗线程 */
int stm32_hw_iwdg_init(void)
{
	rt_err_t err_code;
	rt_thread_t iwdg_thread = RT_NULL;
	
	iwdg_device.ops = &ops;

	iwdg_thread = rt_thread_create(
		"feeddog", feeddog_thread, RT_NULL, 256, 1, 10);
	
	err_code = rt_hw_watchdog_register(
		&iwdg_device, "iwdg", RT_DEVICE_FLAG_WRONLY, RT_NULL);

	if(iwdg_thread != RT_NULL)
	{
		rt_thread_startup(iwdg_thread);
	}

	stm32_iwdg_init(&iwdg_device);
	stm32_iwdg_open(&iwdg_device, RT_DEVICE_OFLAG_WRONLY);

	return err_code;
}
~~~



