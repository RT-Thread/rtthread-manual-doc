# IIC设备 #

基于 Iot Camera 开源项目

RT-Thread的I2C驱动框架共分三层：
1. 硬件I2C驱动层 
2. I2C驱动抽象层 
3. 系统接口层

## IIC设备操作结构体 ##

** 注：系统接口层通过rt_device_open/close/read/write的方式进行，驱动抽象层通过rt_i2c_master/device_send/recv的方式进行操作。**

**富瀚提供的函数有些用的是Fh_i2c.c的底层,函数结构不一样，我们用的时候只考虑接口层就行了 **

* IIC消息传输函数结构体
```C
rt_size_t rt_i2c_transfer(struct rt_i2c_bus_device *bus,
                          struct rt_i2c_msg         msgs[],
                          rt_uint32_t               num);
```
* IIC主机消息发送函数结构体
```C
rt_size_t rt_i2c_master_send(struct rt_i2c_bus_device *bus,
                             rt_uint16_t               addr,
                             rt_uint16_t               flags,
                             const rt_uint8_t         *buf,
                             rt_uint32_t               count);
```
* IIC主机消息接收函数结构体
```C
rt_size_t rt_i2c_master_recv(struct rt_i2c_bus_device *bus,
                             rt_uint16_t               addr,
                             rt_uint16_t               flags,
                             rt_uint8_t               *buf,
                             rt_uint32_t               count);
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
		addr:  -  I2C设备地址(需查看不同设备的数据手册)
		flags: -  控制读写标志
		len:   -  传输的数据长度 
		buf:   -  数据起始地址
** 以上是rtt系统中的相关定义，在fh提供的函数里面则是将众多参数放到同一个结构体中**
```C
struct i2c_driver
{
    int cmd_err;
    int msg_err;
    rt_uint32_t status;

    struct rt_i2c_msg *msgs;
    int msgs_num;
    int msg_write_idx;
    rt_uint32_t tx_buf_len;
    rt_uint8_t *tx_buf;
    int msg_read_idx;
    rt_uint32_t rx_buf_len;
    rt_uint8_t *rx_buf;

    struct rt_i2c_bus_device *i2c_bus_dev;
    struct rt_completion transfer_completion;
    rt_mutex_t lock;
    void*  priv;
};
```

## IIC设备访问接口 ##

### 1. 初始化 ###
* IIC硬件初始化

   向系统注册设备
```C
void rt_hw_i2c_init(void)
```
* IIC传输初始化

  设置i2c总线上从设备的地址
```C
static void fh_i2c_xfer_init(struct rt_i2c_bus_device *dev, 
							 struct rt_i2c_msg msgs[], 
					  		 rt_uint32_t num)

```
### 2. IIC消息传输函数 ###

用于数据在总线上的传输，数据的读取和写入都要用到这个函数。

```C
static rt_size_t fh_i2c_xfer(struct rt_i2c_bus_device *dev,
 							 struct rt_i2c_msg msgs[],
							 rt_uint32_t num)
```

### 3. 总线消息读写操作 ###
* IIC消息读函数 i2c_fh_read()

读到的数据赋值给消息结构体i2c_drv中，暂没有传输的函数外。
如果是DBG模式，这些数据会被打印出来
```C
static void i2c_fh_read(struct rt_i2c_bus_device *dev)
```
* IIC消息写函数，i2c_fh_xfer_msg() 和fh_i2c_read()对应
```C
static void i2c_fh_xfer_msg(struct rt_i2c_bus_device *dev)
```
### 4. 寄存器读写操作 ###
* IIC寄存器读函数 rt_i2c_read_reg()

函数读到的数据通过rt_i2c_transfer（）被发送到总线上
```C
static rt_err_t fh_i2c_read_reg(struct rt_i2c_bus_device *fh81_i2c,
								rt_uint16_t reg, 
								rt_uint8_t *data)
```
* IIC寄存器写函数 rt_i2c_write_reg()
```C
static rt_err_t fh_i2c_write_reg(struct rt_i2c_bus_device *fh81_i2c,
								 rt_uint16_t reg,
								 rt_uint8_t data) 
```
** 寄存器的读写调用了i2c_core.c中的rt_i2c_transfer（）  **

### 5. i2c中断 ###
* IIC中断函数 fh_i2c_interrupt()
```C
static void fh_i2c_interrupt(int this_irq, void *dev_id)
```
### i2c探针 ###
* IIC设备探测函数 fh_i2c_probe()
```C 
int fh_i2c_probe(void *priv_data)
```
** 注：探测总线上的IIC设备，并进行初始化。fh的i2c包括i2c_0、i2c_1**

### i2c设备测试 ###
* 传感器测试函数 i2c_test_sensor()

```C
void i2c_test_sensor() {
	struct rt_i2c_bus_device *fh81_i2c;
	struct rt_i2c_msg msg[2];
	rt_uint8_t data[1] = { 0x00 };

	fh81_i2c = rt_i2c_bus_device_find("i2c1");

	fh_i2c_write_reg(fh81_i2c, 0x04, 0x02);

	fh_i2c_read_reg(fh81_i2c, 0x02, data);

	rt_kprintf("data read from 0x3038 is 0x%x\r\n", data[0]);
	PRINT_I2C_DBG("%s end\n", __func__);
}
```
** 以上应用针对IOT_CAMERA进行编写 **