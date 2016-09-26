# PWM设备 #

基于 Iot Camera 开源项目

RT-Thread的PWM操作
先不看驱动，先把PWM驱动起来再说!

## 编写简单的PWM测试代码，让PWM口一直输出 ##
* 直接上代码，然后分析
  打开firmware-201609xx\applications\main.c添加操作pwm操作需要的的头文件
```C
/* pwm init test
 *
 *
 *
 * */
#include <rtdevice.h>
#include <drivers/pwm.h>
#include <finsh.h>
#include "iomux.h"//端口复用
###############################################################

int pwmSet()
{
	rt_device_t pwm_dev;
	struct pwm_device pwm;
	int ret;
	pwm_dev = rt_device_find("pwm");
	rt_kprintf("pwm_dev->device_id = %d \r\n" , pwm_dev->device_id);
	ret = rt_device_open(pwm_dev, RT_DEVICE_OFLAG_RDWR);
	rt_kprintf("rt_device_open end\r\n");

	/* 打开设备时就使能了具体看驱动,所以屏蔽
	ret = rt_device_control(pwm_dev, ENABLE_PWM, &pwm);
	rt_kprintf("ENABLE_PWM end\r\n");
	*/

	//端口复用,参考lua_pwm.c//感谢网友提醒
	//定义在#include "iomux.h"//端口复用里面
	fh_iomux_pin_switch(PMU_PAD_GPIO_12, 1);//48
	fh_iomux_pin_switch(PMU_PAD_GPIO_13, 1);//49
	fh_iomux_pin_switch(PMU_PAD_GPIO_14, 1);//50

	//ret = rt_device_control(pwm_dev, GET_PWM_DUTY_CYCLE, &pwm);
	pwm.id=0;//pwm0
	pwm.working= 1;
	//38kHz =26.3
	pwm.counter_ns = 13000;//for duty
	pwm.period_ns = 26000;
	ret = rt_device_control(pwm_dev, SET_PWM_DUTY_CYCLE, &pwm);
	rt_kprintf("SET_PWM_DUTY_CYCLE end\r\n");

#if 1
	ret = rt_device_control(pwm_dev, GET_PWM_DUTY_CYCLE, &pwm);
	rt_kprintf("GET_PWM_DUTY_CYCLE id= %d pwm.counter_ns = %d pwm.period_ns = %d\r\n" , pwm.id,pwm.counter_ns , pwm.period_ns);
#endif

#if 1
	pwm.id=1;//pwm1
	pwm.working= 1;
	//38kHz =26.3
	pwm.counter_ns = 13000;//for duty
	pwm.period_ns = 26000;
	ret = rt_device_control(pwm_dev, SET_PWM_DUTY_CYCLE, &pwm);
	rt_kprintf("SET_PWM_DUTY_CYCLE pwm1 end\r\n");

#endif

#if 1
	pwm.id=2;//pwm2
	pwm.working= 1;
	//38kHz =26.3
	pwm.counter_ns = 13000;//for duty
	pwm.period_ns = 26000;
	ret = rt_device_control(pwm_dev, SET_PWM_DUTY_CYCLE, &pwm);
	rt_kprintf("SET_PWM_DUTY_CYCLE pwm1 end\r\n");

#endif


}
MSH_CMD_EXPORT(pwmSet, pwmSet);



```

 * 测试PWM在msh终端，编译后重新启动rtt
  iot_camera 系统启动默认进入msh命令行模式，在命令行直接输入 pwmSet即可启动PWM。

 * 测试pwm在rtthread 初始化完成后自动启动,在return 0;前面添加一行刚刚写的函数即可，编译后重新启动rtt
```C
	pwmSet();
	return 0;
```
## 测试PWM的代码函数int pwmSet()分析说明 ##
 * 用函数rt_device_find 查找pwm设备,在msh 命令行里面也可以看到
```C
     ls /dev/
     或
     list_device
     都可以看到
```

     

 * 打开 pwm设备,其实只用到了控制pwm操作，没有用读写操作
```C
rt_device_open(pwm_dev, RT_DEVICE_OFLAG_RDWR);
```
 * 开启端口复用，切换到PWM功能,重点注意,富翰的芯片，端口有复用功能，上电初始化后，端口默认功能是不一样的，
 就拿pwm来说，共3个端口,其中pwm0 默认是pwm0功能
pwm1 和pwm2默认的就是gpio的功能，必须切换到pwm功能，这样端口才有输出！

```C
//端口复用,参考lua_pwm.c//感谢网友提醒
	//定义在#include "iomux.h"//端口复用里面
	fh_iomux_pin_switch(PMU_PAD_GPIO_12, 1);//48
	fh_iomux_pin_switch(PMU_PAD_GPIO_13, 1);//49
	fh_iomux_pin_switch(PMU_PAD_GPIO_14, 1);//50
```	
 * 控制3路PWM，ID分别为 0 1 2
```C
	pwm.id=0;//pwm0   设置要控制的pwm为PWM0  ，根据需要,可以设置为 0 1 2
	pwm.working= 1;  //设置为正在工作中
	//38kHz =26.3  //我是产生一个红外载波频率
	pwm.counter_ns = 13000;//for duty  //设置高电平时间13000ns=13ms
	pwm.period_ns = 26000;  //总脉宽26MS
	ret = rt_device_control(pwm_dev, SET_PWM_DUTY_CYCLE, &pwm); //把pwm结构体数据通过 rt_device_control 函数传入设备驱动并开始启动pwm
	rt_kprintf("SET_PWM_DUTY_CYCLE end\r\n");
```
 * pwm的其他控制参数,在firmware-201609xx\drivers\pwm.h里面可以找到，共4个
```C
#define ENABLE_PWM                  (0x10)
#define DISABLE_PWM                 (0x11)

#define SET_PWM_DUTY_CYCLE          (0x12)
#define GET_PWM_DUTY_CYCLE          (0x13)
```	 
 
 * 其中的具体参数细节，参考驱动文件firmware-201609xx\drivers\pwm.c


## PWM设备驱动部分分析 ##
驱动文件：
firmware-201609xx\drivers\pwm.c
firmware-201609xx\drivers\pwm.h

## 初始化，注册一个pwm设备 ##
void rt_hw_pwm_init(void)
该函数会被系统初始化时调用
## 打开和关闭pwm设备的驱动实现 ##
static rt_err_t fh_pwm_open(rt_device_t dev, rt_uint16_t oflag)
该函数里面会使能PWM，因此在应用层打开pwm设备后，可以省掉对pwm使能ENABLE_PWM
关闭pwm设备的驱动 同样包含了禁用pwm操作函数PWM_Enable(pwm_obj, RT_FALSE);

## 对pwm设备操作函数的实现 ##
主要就是应用层的 rt_device_control函数在驱动层的具体实现,
调用函数时，起始就是调用驱动里面的
static rt_err_t fh_pwm_ioctl(rt_device_t dev, rt_uint8_t cmd, void *arg)
函数
即上面提到的4条控制密令

## 对pwm设备操作富翰soc底层的寄存器操作库函数 ##
寄存器底层驱动：firmware-201609xx\libraries\driverlib\fh_pwm.c
它为firmware-201609xx\drivers\pwm.c 系统设备驱动提供具体的硬件寄存器操作
从而 将 pwm的应用层、rtt的pwm驱动层、富翰芯片的pwm硬件联通起来

**以上应用针对IOT_CAMERA进行编写**

--Edit by duxingkei