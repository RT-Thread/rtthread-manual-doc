# gpio驱动编写

gpio驱动框架函数调用关系如图所示：
![image](https://github.com/enkiller/rtthread-manual-doc/blob/master/zh/3_chapters/gpio.png)
根据调用关系图我们可以知道，无论从通用I/O设备管理接口调用还是从开放API调用，都会间接调用底层三个接口函数，所以我们只需要三个底层的接口即可。具体驱动实现，分为五步完成。
## 1 pin_read接口实现

pin_read接口原型为：
``` 
int (*pin_read)(struct rt_device *device, rt_base_t pin);
``` 
参数说明

| 参数   | 说明         | 其他           |
|--------|--------------|----------------|
| device | 设备描述句柄 |                |
| pin    | 引脚标号     | 不同引脚的标识 |

返回值：

相应引脚的电平状态。

该接口需要实现的功能就是读取某个引脚的电平状态。在实现这个接口的时候，除了名字可以不一样外，函数返回值类型、参数类型、参数个数都与接口一致。举个例子。

``` 
int xx_gpio_read(struct rt_device * dev, rt_base_t pin)
{
    rt_base_t sta; //保存引脚电平临时变量
    
     /* 将pin值转换为对应引脚 */
    
     /* 读取对应引脚电平到sta中 */
    
    return sta;   //返回引脚电平状态
}
``` 

其中struct rt_device \*
可以写成rt_device_t。上层驱动框架通过传递不同的pin值来标识不同的引脚，读者需要将pin值与引脚一一对应起来。我们可以把操作一个引脚所用的信息打包成结构体，然后用数组进行管理。这里简单演示如何使用结构体和数组来达到这一目的，读者也可以使用其他的方法。

``` 
struct _gpio_info
{
    gpio_info info; /* 打包GPIO引脚信息 */
}
/* 使用数组初始化需要操作的GPIO引脚信息 */
static const struct _gpio_info gpios[] = 
{
    {GPIO0},
    {GPIO1},
    {GPIO2},
    …
}
```
当上层驱动框架传递标号 0
时，直接取出gpios[0]中的信息，就对应操作GPIO0这个引脚了。

读引脚电平的功能的实现由读者自行完成。

## 2 pin_write接口实现

pin_write接口原型为：
``` 
void (*pin_write)(struct rt_device *device, rt_base_t pin, rt_base_t value);
``` 
参数说明

| 参数   | 说明         | 其他         |
|--------|--------------|--------------|
| device | 设备描述句柄 |              |
| pin    | 引脚标号     | 不同引脚标识 |
| value  | 引脚状态     | 引脚电平值   |

返回值：

空。

该接口需要实现的功能是根据value值配置引脚状态，也就是设置引脚的高低电平。在实现这个接口的时候，除了名字可以不一样外，函数返回值类型、参数类型、参数个数都与接口一致。举个例子。

``` 
void xx_gpio_write(struct rt_device *dev, rt_base_t pin, rt_base_t value )
{
    /* 将pin值转换为对应引脚 */
    
    /* 配置对应引脚电平状态 */
}
``` 
pin值与引脚对应方法请看pin_read接口实现。
写某一电平状态功能，由读者自行完成。

## 3 pin_mode接口实现

pin_mode接口原型为:
``` 
void (*pin_mode)(struct rt_device *device, rt_base_t pin, rt_base_t mode);
``` 
参数说明:

| 参数   | 说明         | 其他           |
|--------|--------------|----------------|
| device | 设备描述句柄 |                |
| Pin    | 引脚标号     | 不同引脚标识   |
| mode   | 引脚模式     | 输入、输出模式 |

返回值：

空。

当前pin驱动框架支持的模式如下:

\#define PIN_MODE_OUTPUT 0x00 /\* 输出模式 \*/

\#define PIN_MODE_INPUT 0x01 /\* 输入模式 \*/

\#define PIN_MODE_INPUT_PULLUP 0x02 /\* 上拉输入 \*/

该接口需要实现的功能是配置引脚模式，如输入模式，输出模式等。在实现这个接口的时候，除了名字可以不一样外，函数返回值类型、参数类型、参数个数都与接口一致。举个例子。
``` 
void xx_gpio_mode(struct rt_device *dev, rt_base_t pin, rt_base_t mode)
{
    /* 将pin值转换为对应引脚 */
    /* 对应引脚模式配置 */
    switch(mode)
    {
        PIN_MODE_INPUT:
		/* 配置成输入模式 */
		break;
	    PIN_MODE_OUTPUT:
		/* 配置成输出模式 */
		break;
    }
}
``` 
pin值与引脚对应方法请看pin_read接口实现。
配置引脚输出输出模式，由读者自行完成。

## 4 初始化接口

定义一个pin接口类型的结构体，这个结构体可以是全局变量，也可以rt_malloc出来的内存块。并把我们实现的三个接口赋值给这个结构体，完成接口的初始化。举个例子。
``` 
const static struct rt_pin_ops _xx_pin_ops =
{
    xx_gpio_mode,
    xx_gpio_write,
    xx_gpio_read,
};

``` 
## 5注册驱动

最后一步就是注册该引脚驱动,调用rt_device_pin_register()函数即可。

rt_device_pin_register原型如下:
``` 
int rt_device_pin_register(const char *name, const struct rt_pin_ops *ops, void *user_data)
``` 
函数参数说明如下:

| 参数      | 说明             | 其他               |
|-----------|------------------|--------------------|
| name      | 驱动名字         | 不能和其他驱动重名 |
| ops       | 底层引脚操作接口 |                    |
| user_data | 用户私有数据     |                    |

返回值：

0。

注册驱动名为:pin，操作接口指针为:_xx_pin_ops，私有数据为: 空。驱动注册举例。
``` 
rt_device_pin_register("pin", &_stm32_pin_ops, RT_NULL);
``` 
