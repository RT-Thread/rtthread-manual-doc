# IIC设备

依赖于RT_USING_I2C选项。


## IIC设备访问接口

** 注：一般访问IIC设备并不通过rt_device_open/close/read/write的方式进行，而更多的是通过IIC自己的API接口进行。 **

* IIC消息传输
```C
rt_size_t rt_i2c_transfer(struct rt_i2c_bus_device *bus,
                          struct rt_i2c_msg         msgs[],
                          rt_uint32_t               num);
```

* IIC消息结构定义

```C
struct rt_i2c_msg
{
    rt_uint16_t addr;
    rt_uint16_t flags;
    rt_uint16_t len;
    rt_uint8_t  *buf;
};
```

** 注：Linux设备文件接口兼容考虑 **
