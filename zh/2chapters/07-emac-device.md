# EMAC以太网设备

主要实现设备接口：

```C
static rt_err_t your_emac_tx( rt_device_t dev, struct pbuf* p);
```

```C
static struct pbuf *your_emac_rx(rt_device_t dev);
```

```C
static rt_err_t your_emac_control(rt_device_t dev, rt_uint8_t cmd, void *args)
{
    struct net_device *ndev = (struct net_device *)dev;
    switch(cmd)
    {
    case NIOCTL_GADDR:
        /* get mac address */
        if(args) rt_memcpy(args, ndev->dev_addr, 6);
        else return -RT_ERROR;
        break;

    default :
        break;
    }

    return RT_EOK;
}
```

主要用于获得底层网卡MAC地址（协议栈也会通过这个接口获得网卡的MAC地址）。
