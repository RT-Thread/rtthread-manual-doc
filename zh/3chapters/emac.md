# 以太网EMAC驱动

## 初始化工作

rt-thread中的以太网及lwIP网络协议栈初始化序列类似如下所示：

~~~{.c}
    eth_system_device_init()
    rt_hw_emac_eth_init();
    lwip_sys_init();
~~~

a. eth_system_device_init函数创建了Rx和Tx两个线程,同时创建了两个邮箱用于数据包的同步接收和发送，该函数里被创建线程的stack大小和线程优先级统一在rtconfig.h里进行设置。

b. rt_hw_emac_eth_init是需要重点介绍的函数,它实现了所有的EMAC和phy初始化。下面我们通过具体的代码来分析如何初始化emac和phy。

~~~{.c}
    int rt_hw_emac_eth_init(void)
    {
        struct rt_emac_eth * emac_eth_device;
        emac_eth0_ptr->EMACx=(Emac_EMAC_REG_GRP*)Emac_EMAC0_REG_GRP_PTR;
        emac_eth0_ptr->plugged    = 0;
        emac_eth0_ptr->tx_vector  = EMC0_TX_IRQn;
        emac_eth0_ptr->rx_vector  = EMC0_RX_IRQn;

        init_emac0_rx_ring_buffer();
        emac_eth_device = &emac_eth0_device;
        emac_eth_device->ETHx = emac_eth0_ptr;
        memcpy(emac_eth_device->dev_addr, emac_eth0_addr, sizeof(emac_eth0_addr));

        /* register interrupt */
        rt_hw_interrupt_install(emac_eth_device->ETHx->tx_vector,ETH_TX_IRQHandler,emac_eth_device,"e0txisr");
        rt_hw_interrupt_set_priority(emac_eth_device->;ETHx->tx_vector,IRQ_LEVEL_1);
        rt_hw_interrupt_install(emac_eth_device->ETHx->rx_vector,ETH_RX_IRQHandler,emac_eth_device,"e0rxisr");
        rt_hw_interrupt_set_priority(emac_eth_device-->ETHx->rx_vector,IRQ_LEVEL_1);

        /* RT ETH device interface init */
        emac_eth_device->parent.parent.init       = rt_emac_eth_init;
        emac_eth_device->parent.parent.open       = rt_emac_eth_open;
        emac_eth_device->parent.parent.close      = rt_emac_eth_close;
        emac_eth_device->parent.parent.read       = rt_emac_eth_read;
        emac_eth_device->parent.parent.write      = rt_emac_eth_write;
        emac_eth_device->parent.parent.control    = rt_emac_eth_control;
        emac_eth_device->parent.parent.user_data  = RT_NULL;
        emac_eth_device->parent.eth_rx            = rt_emac_eth_rx;
        emac_eth_device->parent.eth_tx            = rt_emac_eth_tx;

        /* Init tx buffer free semaphore */
        rt_sem_init(&emac_eth_device->tx_buf_free, "eth0_tx", TX_DESCRIPTOR_NUM, RT_IPC_FLAG_FIFO);

        /* Register eth device */
        eth_device_init(&(emac_eth_device->parent), "e0");
        eth_init(emac_eth_device->ETHx, emac_eth_device->dev_addr);

        /* Creat a timer for monitor the link status */
        rt_timer_init(&(emac_eth_device->link_timer), "link_timer", eth_chk_link, (void *) emac_eth_device, RT_TICK_PER_SECOND, RT_TIMER_FLAG_PERIODIC);

        rt_timer_start(&(emac_eth_device->link_timer));
        rt_hw_interrupt_umask(emac_eth_device->ETHx->tx_vector);
        rt_hw_interrupt_umask(emac_eth_device->;ETHx->rx_vector);
        rt_kprintf("ETH0 init Completed ....\n");
        return 0;
    }
~~~

在我们开始了解该函数之前，我们先了解下几个重要的数据结构。

    a. emac_eth0_device
    emac_eth0_device的原型为:

~~~{.c}
    struct rt_emac_eth
    {
        /* inherit from ethernet device */
        struct eth_device parent;
        ETH_STR *ETHx;
        uint8_t dev_addr[6];
        struct rt_semaphore tx_buf_free;
        struct rt_timer     link_timer;
    };
~~~

该结构体里的parent参数原型在ethernetif.h里,用户在封装自己的数据类型时必须定义该参数,它在注册网络设备时需要用到。

ETH_STR为与EMAC相关的属性定义:

~~~{.c}
    typedef struct _eth_str
    {
        EMAC_REG_GRP *EMACx;

        struct eth_descriptor rx_desc[RX_DESCRIPTOR_NUM];
        struct eth_descriptor tx_desc[TX_DESCRIPTOR_NUM];

        volatile struct eth_descriptor  *cur_tx_desc_ptr, *cur_rx_desc_ptr, *fin_tx_desc_ptr;

        rt_uint8_t rx_buf[RX_DESCRIPTOR_NUM][PACKET_BUFFER_SIZE];
        rt_uint8_t tx_buf[TX_DESCRIPTOR_NUM][PACKET_BUFFER_SIZE];
        rt_int32_t plugged;
        rt_int32_t tx_vector,rx_vector;
    }ETH_STR;
~~~

`Emac_EMAC_REG_GRP *EMACx` 为EMAC寄存器组指针,紧接着是Rx和Tx描述符定义,网络链接状态定义,Tx和Rx中断向量号定义。该结构体的内容可根据用户的EMAC实际情况来定义，没有严格的要求和限制。有的EMAC要求描述符应该定义在非Cache内存区间。dev_addr存放MAC地址。

tx_buf_free和link_timer为笔者自定义的变量，并非要求用户必须定义。 因此,EMAC的数据结构的封装除了struct eth_device parent是必须定义的外，其他的变量可根据用户的实际情况来进行封装。

rt_hw_emac_eth_init函数初始化了EMAC的寄存器组,以太网的连接状态，Tx中断号,Rx中断号,然后初始化Tx和Rx中断参数。中断初始化代码如下:

~~~{.c}
    /* register interrupt */
    rt_hw_interrupt_install(emac_eth_device->ETHx->tx_vector,ETH_TX_IRQHandler, emac_eth_device,"e0txisr");
    rt_hw_interrupt_set_priority(emac_eth_device->ETHx->tx_vector,IRQ_LEVEL_1);

    rt_hw_interrupt_install(emac_eth_device->ETHx->rx_vector,ETH_RX_IRQHandler, emac_eth_device,"e0rxisr");
    rt_hw_interrupt_set_priority(emac_eth_device->ETHx->rx_vector,IRQ_LEVEL_1);
~~~

`emac_eth_device->ETHx->tx_vector`为Tx中断向量号。

ETH_TX_IRQHandler为Tx中断服务程序。

emac_eth_device为EMAC封装好的结构体,作为参数传输给ETH_TX_IRQHandler使用。

"e0txisr" 是中断服务程序的名称。

在设置完中断参数后,不建议立即使能中断,因为相关的EMAC操作函数和RT-Thread的以太网架构还没有初始化好。

在初始化EMAC的中断参数后,紧接着是设置EMAC结构体里parent的参数,请见如下代码。

~~~{.c}
    /* RT ETH device interface init */
    emac_eth_device->parent.parent.init       = rt_emac_eth_init;
    emac_eth_device->parent.parent.open       = rt_emac_eth_open;
    emac_eth_device->parent.parent.close      = rt_emac_eth_close;
    emac_eth_device->parent.parent.read       = rt_emac_eth_read;
    emac_eth_device->parent.parent.write      = rt_emac_eth_write;
    emac_eth_device->parent.parent.control    = rt_emac_eth_control;
    emac_eth_device->parent.parent.user_data  = RT_NULL;

    emac_eth_device->parent.eth_rx            = rt_emac_eth_rx;
    emac_eth_device->parent.eth_tx            = rt_emac_eth_tx;
~~~

我们发现以上代码类似于Linux的file_operations初始化。

## 网卡驱动接口

### 设备通用接口

rt_emac_eth_init函数里可将对EMAC和PHY的初始化代码置于此。rt_emac_eth_open、rt_emac_eth_close、rt_emac_eth_read、rt_emac_eth_write、rt_emac_eth_control可按类似于下面的代码格式编写。

~~~{.c}
    static rt_err_t rt_stm32_eth_open(rt_device_t dev, rt_uint16_t oflag)
    {
        return RT_EOK;
    }
    static rt_err_t rt_stm32_eth_close(rt_device_t dev)
    {
        return RT_EOK;
    }

    static rt_size_t rt_stm32_eth_read(rt_device_t dev, rt_off_t pos, void* buffer, rt_size_t size)
    {
        rt_set_errno(-RT_ENOSYS);
        return 0;
    }

    static rt_size_t rt_stm32_eth_write (rt_device_t dev, rt_off_t pos, const void* buffer, rt_size_t size)
    {
        rt_set_errno(-RT_ENOSYS);
        return 0;
    }

    static rt_err_t rt_stm32_eth_control(rt_device_t dev, rt_uint8_t cmd, void *args)
    {
        switch(cmd)
        {
            case NIOCTL_GADDR:
          /* get mac address */
            if(args) memcpy(args, stm32_eth_device.dev_addr, 6);
                else return -RT_ERROR;
                break;

            default :
                break;
        }

        return RT_EOK;
    }
~~~

### 接收函数

接收函数rt_emac_eth_rx是从描述符里读取数据并存储于lwIP的pbuf缓冲区。

~~~{.c}
    /* reception packet. */
    struct pbuf *rt_emac_eth_rx(rt_device_t dev)
    {
        struct pbuf* p;
        rt_uint32_t framelength = 0;
        register rt_base_t level = 0;
        unsigned char *buffer_addr = 0;

        /* init p pointer */
        p = RT_NULL;

        /* disable interrupt */
        level = rt_hw_interrupt_disable();

        if (emac0_ring_buffer.rx_cnt > 0)
        {
                emac0_ring_buffer.rx_cnt--;
                buffer_addr = emac0_ring_buffer.buffer_rx_out_ptr->buffer;
                framelength = emac0_ring_buffer.buffer_rx_out_ptr->packet_length;
                emac0_ring_buffer.buffer_rx_out_ptr++;

                if(emac0_ring_buffer.buffer_rx_out_ptr == &emac0_ring_buffer.rx_buff[RX_RING_BUFFER_LENGTH])
                {
                    emac0_ring_buffer.buffer_rx_out_ptr = &emac0_ring_buffer.rx_buff[0];
                }
        }
        else
        {
             rt_hw_interrupt_enable(level);
            return p;
        }

        /* allocate buffer */
        p = pbuf_alloc(PBUF_RAW, framelength, PBUF_POOL);
        if (p != RT_NULL)
        {
            rt_uint8_t * from;

            from = (unsigned char *)(buffer_addr);

            pbuf_take(p, from, framelength);

    #ifdef ETH_RX_DUMP
            packet_dump("RX dump", p);
    #endif /* ETH_RX_DUMP */
        }
        rt_hw_interrupt_enable(level);
        return p;
    }
~~~

因各个EMAC的DMA和描述符的数据结构以及读取数据的方式不一样,这里不对如何读取数据作出介绍,重点介绍如何将数据存储于lwIP的pbuf缓冲区。即对于如下代码片段：

~~~{.c}
    struct pbuf* p;

    /* allocate buffer */
    p = pbuf_alloc(PBUF_RAW, framelength, PBUF_POOL);
    if (p != RT_NULL)
    {
        rt_uint8_t * from;

        from = (unsigned char *)(buffer_addr);

        pbuf_take(p, from, framelength);

    #ifdef ETH_RX_DUMP
        packet_dump("RX dump", p);
    #endif /* ETH_RX_DUMP */
    }
    rt_hw_interrupt_enable(level);
    return p;
~~~

pbuf_alloc(PBUF_RAW, framelength, PBUF_POOL);从lwIP的缓冲区中分配framelength大小的内存,framelength为实际接收到的数据包大小,请注意PBUF_RAW和PBUF_POOL的类型，设置不合适有可能造成接收和发送异常,因本人使用的是lwIP的memory pool内存分配方式，所以使用这两个参数。

pbuf_take将接收到的数据包copy到lwIP的缓冲区,由lwIP的IP层去做拆包处理。

### 发送函数

发送函数rt_emac_eth_tx将lwIP IP层的数据包存储至描述符区间，由EMAC的DMA读取发送出去。

~~~{.c}
    /* transmit packet. */
    rt_err_t rt_emac_eth_tx( rt_device_t dev, struct pbuf* p)
    {
        register rt_base_t level = 0;
        rt_uint16_t length = 0;

        /* Disable interrupt */
        level = rt_hw_interrupt_disable();
        if (emac0_ring_buffer.tx_cnt &lt;TX_RING_BUFFER_LENGTH)
        {
            emac0_ring_buffer.tx_cnt++;
            length = p->tot_len;
            if (length >1518)
            length = 1518;

            pbuf_copy_partial(p, ( void *)emac0_ring_buffer.buffer_tx_in_ptr->buffer, length, 0);
            emac0_ring_buffer.buffer_tx_in_ptr->packet_length = length;
            emac0_ring_buffer.buffer_tx_in_ptr++;

            if(emac0_ring_buffer.buffer_tx_in_ptr== &emac0_ring_buffer.tx_buff[TX_RING_BUFFER_LENGTH])
            {
                emac0_ring_buffer.buffer_tx_in_ptr   = &emac0_ring_buffer.tx_buff[0];
            }

            rt_hw_interrupt_enable(level);
            rt_sem_release(&emac0_ring_buffer.tx_buf_available_sem);

            /* Return SUCCESS */
            return RT_EOK;
        }

        rt_hw_interrupt_enable(level);
        return -RT_ERROR;
    }
~~~

该函数读取lwIP协议IP层指定长度的数据包存储至EMAC的发送描述符区间供DMA读取发送至以太网络。`pbuf_copy_partial(p, ( void *)emac0_ring_buffer.buffer_tx_in_ptr-&gt;buffer, length, 0)`，p作为形参包括要发送数据包的长度;`( void *)emac0_ring_buffer.buffer_tx_in_ptr->buffer`为发送描述符缓冲区;length为要发送数据包的长度,从p中读取;最后一个参数为offset即偏移为0，即从头开始取数据。因不同EMAC的DMA数据结构及描述符的定义方式不一样,数据发送在描述符及DMA的操作这一块也就不一样了,这里不对其作进一步的阐述。

### 链路状态监控

* rt_sem_init初始化信号量用于Tx中断服务程序与rt_emac_eth_tx之间的同步。
* eth_device_init注册网络设备,e0为设备名,请注意上面强调的parent在这里用到, eth_device_init函数位于ethernetif.c文件中。
* eth_init函数初始化EMAC和PHY,上面提到过可将该函数实现的全部功能置于rt_emac_eth_init函数中。
* rt_timer_init创建了一个定时器任务用于检测网络的链路状态,比如断线或重新接通网络。下面我们将简要讲明该函数的作用。

~~~{.c}
     /* Creat a timer for monitor the link status */
    rt_timer_init(&(emac_eth_device-&gt;link_timer), "link_timer",
    eth_chk_link,
    (void *)emac_eth_device,
    RT_TICK_PER_SECOND,
    RT_TIMER_FLAG_PERIODIC);

    rt_timer_start(&(emac_eth_device->link_timer));
~~~

我们重点看eth_chk_link任务。

~~~{.c}
    void eth_chk_link(void *param)
    {
        uint8_t  phy_addr = CONFIG_ETH0_PHY_ADDR;
        uint32_t reg      = 0;
        struct rt_emac_eth *emac_eth_device = (struct rt_emac_eth *)0 ;
        ETH_STR *ETHx = (ETH_STR *)0;

        emac_eth_device = (struct rt_emac_eth *)param;
        ETHx = (ETH_STR *)(emac_eth_device->ETHx);


        if(ETHx->EMACx == Emac_EMAC0_REG_GRP_PTR)
        {
            phy_addr = CONFIG_ETH0_PHY_ADDR;
        }
        else
        {
            phy_addr = CONFIG_ETH1_PHY_ADDR;
        }

        reg = mdio_read(ETHx,phy_addr, MII_BMSR);

        if (reg & BMSR_LSTATUS)
        {
            if (!ETHx->plugged)
            {
                ETHx->plugged = TRUE;

            reg = mdio_read(ETHx,phy_addr, MII_LPA);

            if (reg &ADVERTISE_100FULL)
            {
                ETH_DBG("100 Full\n");
                ETHx->EMACx->emac_mcmdr |= MCMDR_100M_FULL_MSK;
            }
            else if (reg & ADVERTISE_100HALF)
            {
                ETH_DBG("100 Half\n");
                ETHx->EMACx->emac_mcmdr &= (~MCMDR_FDUP_MSK);
                ETHx->EMACx->emac_mcmdr |= MCMDR_100M_HALF_MSK;
            }
            else if (reg & ADVERTISE_10FULL)
            {
                ETH_DBG("10 Full\n");
                ETHx->EMACx->emac_mcmdr &= ~MCMDR_OPMOD_MSK;
                ETHx->EMACx->emac_mcmdr |= MCMDR_FDUP_MSK;
            }
            else
            {
                ETH_DBG("10 Half\n");
                ETHx->EMACx->emac_mcmdr &= (~MCMDR_100M_FULL_MSK);
            }
            ETHx->EMACx->emac_mcmdr  |= (MCMDR_RXON_MSK|MCMDR_TXON_MSK);

            /* Maybe some income packets in buffer */
            ETH_TRIGGER_TX(ETHx->EMACx);

            /* Send link up. */
            eth_device_linkchange(&(emac_eth_device->;parent), RT_TRUE);
        }
    }
    else
    {
        if (ETHx->plugged)
        {
            ETH_DBG("Link Down\n");
            ETHx->plugged = FALSE;
            rt_hw_interrupt_mask(ETHx-&gt;tx_vector);
            rt_hw_interrupt_mask(ETHx-&gt;rx_vector);

            /* Disable Tx and Rx */
            ETHx-&gt;EMACx-&gt;emac_mcmdr  &= (~(MCMDR_RXON_MSK|MCMDR_TXON_MSK));

            /* Send link down. */
            eth_device_linkchange(&(emac_eth_device->parent), RT_FALSE);
        }
    }
}
~~~

根据以上代码我们可以看到都是围绕PHY的状态寄存器来判断网络的工作状态,但无论状态是否变化都会调用eth_device_linkchange函数,第2个参数RT_TRUE表示链路正常,RT_FALSE表示链路不正常。该函数可以由用户单独作为一个线程,优先级可以设得比较低,调度时间可以设得比较长以防止链路正常时引起任务的频繁调度切换对整个系统性能产生影响。

在初始化最后,我们终于可以使能中断了。

~~~{.c}
    rt_hw_interrupt_umask(emac_eth_device-&gt;ETHx-&gt;tx_vector);
    rt_hw_interrupt_umask(emac_eth_device-&gt;ETHx-&gt;rx_vector);
~~~

RT-Thread为系统本身和驱动程序提供了一种初始化机制,用户只需要在函数的}末尾加上INIT_COMPONENT_EXPORT(函数名);RT-Thread即可自动调用这些函数执行,为了方便调试这里我们在以下三个函数的末尾去掉这个宏,由用户手动调用这三个函数。

~~~ {.c}
    eth_system_device_init();
    rt_hw_nuc970_eth_init();
    lwip_sys_init();
~~~

以上内容讲述了如何在RT-Thread的以太网框架上实现EMAC的初始化,在这里没有介绍如何初始化PHY和EMAC,也没有详细讲解如何初始化描述符和DMA,因不同的EMAC和PHY它们的初始化方式和流程不一样。

## EMAC ISR注意事项

EMAC在收到一个packet后会产生一个Rx 中断,在发送完一个packet后会产生一个Tx中断。有的MCU or MPU将这两个IRQ分成两个中断向量来处理,有的MCU or MPU用一个中断向量来处理;但无论是一个IRQ还是两个IRQ向量我们都分成两个部分来讲解RT-Thread框架下的EMAC ISR应该做哪些工作。

a. Rx ISR

Rx ISR根据EMAC的架构清除接收中断标志位及相关的状态,这里要注意接收错误中断标志位的处理,以免造成调试上的麻烦。然后根据EMAC的DMA与描述符的数据处理流程在接收到一个有效的包后就调用eth_device_ready(&amp;(emac_eth-&gt;parent))函数,发送一个邮箱通知rt_nuc970_eth_rx函数对描述符里的数据作出处理。

b. Tx ISR

rt_nuc970_eth_tx函数在取出lwIP IP层的数据包后存储至发送描述符中,然后启动EMAC的发送功能,EMAC发送完一个数据包后会产生一个中断。在Tx ISR中应该作清除EMAC状态和发送中断标志位的工作，如EMAC允许查询其他描述符是否还有数据未发送,Tx ISR将查询其他描述符是否有未发送的数据,将其发送出去。</code></pre>
