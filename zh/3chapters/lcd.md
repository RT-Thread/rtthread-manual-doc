# LCD屏驱动

## 简介
LCD属于RT-Thread的一个设备，LCD设备具备一般RT-Thread设备的所有属性：init/open/close/read/write/control。其设备结构体rt_lcd_device由struct rt_device派生而来。此外，LCD还具备特有的属性：绘图操作。要在RT-Thread中正常使用LCD，需要按照RT-Thread的设备框架编写LCD设备驱动程序，同时也是使用UIENGINE的基础。
## lcd设备接口
RT-Thread的LCD设备驱动由以下4个内容组成。

    I>>  1个结构体
        rt_lcd_device
    Ⅱ>> 2个硬件接口函数
        lcd_init();
        lcd_control();
    Ⅲ>> 5个基础绘图函数，用于初始化rt_device_graphic_ops(这一步只有pixel像素屏用到，frambuffer帧缓冲屏驱动程序忽略此步)
        void rt_hw_lcd_set_pixel(const char *pixel, int x, int y);
        void rt_hw_lcd_get_pixel(char *pixel, int x, int y);
        void rt_hw_lcd_draw_hline(const char *pixel, int x1, int x2, int y);
        void rt_hw_lcd_draw_vline(const char *pixel, int x, int y1, int y2);
        void rt_hw_lcd_draw_raw_hline(const char *pixel, int x, int y, rt_size_t size);
    Ⅳ>> 向RTT注册LCD设备，初始化并打开LCD设备
        调用函数int rt_hw_lcd_init(void)，向RTT注册、初始化并打开LCD设备。

### pixel像素屏驱动
**LCD设备结构体rt_lcd_device**

rt_lcd_device结构体是rt_device结构体扩展而来，编写LCD驱动的需要填充此结构体，其原型如下：
~~~
    struct rt_lcd_device
    {
        struct rt_device    parent;

        rt_uint8_t   pixel_format;       /*像素格式*/
        rt_uint8_t   bits_per_pixel;     /*每个像素占用位数*/
        rt_uint16_t  width;              /*LCD宽度，横向像素个数*/
        rt_uint16_t  height;             /*LCD高度，纵向像素个数*/
        rt_uint8_t   direction;          /*LCD显示方向*/
        rt_uint16_t  id;

        void *framebuffer;               /*帧缓存*/
    };
    typedef struct rt_lcd_device rt_lcd_t;
~~~

**硬件接口函数**

    这两个接口函数是所有RTT管理的设备都具备的公共函数，RTT的设备还具备其它的公共函数，但对于LCD设备，我们只需要填充这两个函数。

    函数功能：①.初始化和lcd连接的处理器的接口；②.初始化液晶硬件。
    函数定义：static rt_err_t lcd_init(rt_device_t dev)。
    参数说明：dev，lcd设备。

    函数功能：设置lcd的基本参数。
    函数定义：static rt_err_t lcd_control(rt_device_t dev, rt_uint8_t cmd, void *args)。
    参数说明：dev，lcd设备；
             cmd，操作命令；
             args，操作参数。

**基本绘图函数**

    5个基本函数是UIENGINE的所有绘制操作的基础，是使用LCD必须提供的函数，其描述如下：

    函数功能：画点
    函数定义：void rt_hw_lcd_set_pixel(const char *pixel, int x, int y)。
    参数说明：pixel，像素点颜色。RT-THREAD中对像素颜色有专门的处理函数，只需要引用即可；
             x，液晶的x坐标；
             y，液晶的y坐标。

    函数功能：获取点颜色
    函数定义：void rt_hw_lcd_get_pixel(char *pixel, int x, int y);
    参数说明：pixel，像素点颜色；
             x，液晶的x坐标；
             y，液晶的y坐标。

    函数功能：画水平线
    函数定义：void rt_hw_lcd_draw_hline(const char *pixel, int x1, int x2, int y);
    参数说明：pixel，像素点颜色；
             x1，液晶的x轴起始坐标；
             x2，液晶的x轴结束坐标；
             y，液晶的y轴坐标。

    函数功能：画竖直线
    函数定义：void rt_hw_lcd_draw_vline(const char *pixel, int x, int y1, int y2);
    参数说明：pixel，像素点颜色；
             x，液晶的x轴坐标；
             y1，液晶的y轴起始坐标；
             y2，液晶的y轴结束坐标。

    函数功能：画起始水平线
    函数定义：void rt_hw_lcd_draw_raw_hline(const char *pixel, int x, int y, rt_size_t size);
    参数说明：pixel，像素点颜色；
             x，液晶的x轴坐标；
             y，液晶的y轴坐标；
             size，液晶水平轴上的像素个数。

**向RTT注册LCD设备，初始化并打开LCD设备**

    写完以上7个模块以后，调用rt_hw_lcd_init()，向RTT注册LCD设备，初始化并打开LCD设备，便可以在RT-Thread中正常的使用LCD了。
~~~
    void rt_hw_lcd_init(void)
    {
    	rt_device_t lcd;
    
        /*初始化LCD设备结构体*/
        lcd_device.parent.type      = RT_Device_Class_Graphic;  /*设备类型*/
        lcd_device.parent.init      = lcd_init;
        lcd_device.parent.open      = lcd_open;
        lcd_device.parent.close     = lcd_close;
        lcd_device.parent.read      = RT_NULL;
        lcd_device.parent.write     = RT_NULL;
        lcd_device.parent.control   = lcd_control;
        lcd_device.parent.user_data = &lcd_ili_ops; /*设置用户私有数据*/
    
        lcd_device.pixel_format   = RTGRAPHIC_PIXEL_FORMAT_RGB565P;
        lcd_device.bits_per_pixel = LCD_BITS_PER_PIXEL;
    
        /*注册图形设备驱动*/
        rt_device_register(&(lcd_device.parent), 
                         "lcd",
                          RT_DEVICE_FLAG_RDWR| T_DEVICE_FLAG_STANDALONE);
        lcd = rt_device_find("lcd");  /*查找LCD设备               */
    
        /*初始化*/
        rt_device_init(lcd);
        rt_device_open(lcd,RT_DEVICE_OFLAG_RDWR);
    }
~~~

**以STM32为例，编写一个LCD驱动程序**

按照上述操作步骤，编写一个RT-Thread的LCD驱动程序。

单片机：STM32F407IGT6，使用FSMC端口

LCD模块：原子哥的2.4/2.8寸液晶，液晶驱动芯片ILI9341

***编写硬件初始化函数***

    - 配置单片机引脚端口

        RCC_Configuration();
        GPIO_Configuration();
        FSMC_Configuration();

    - 设置FSMC驱动液晶的显存地址和寄存器地址

        #define LCD_REG (*(volatile rt_uint16_t*)(rt_uint32_t)0x60000000)   /*显存地址，用于写显示数据*/
        #define LCD_RAM (*(volatile rt_uint16_t*)(rt_uint32_t)0x60010000)   /*寄存器地址，用于写显示器的配置寄存器*/

    - 液晶的寄存器操作函数

        void LCD_WR_RAM(rt_uint16_t Val)     /*写显示寄存器，用于显示数据*/
        void LCD_WR_REG(rt_uint16_t index)   /*写显示寄存器，用于配置显示器*/
        rt_uint16_t LCD_RD_DATA(void)        /*读显示寄存器，读取当前显示的内容*/

***初始化lcd设备结构体***

    struct rt_lcd_device
    {
        struct rt_device    parent;

        rt_uint8_t  pixel_format;
        rt_uint8_t  bits_per_pixel;
        rt_uint16_t width;
        rt_uint16_t height;
        rt_uint8_t  direction;
        rt_uint16_t id;

        void *framebuffer;
    };
    struct rt_lcd_device  lcd_device;

***编写2个硬件接口函数***
~~~
    /*  LCD设备初始化  */
    static rt_err_t lcd_init(rt_device_t dev)
    {
        RCC_Configuration();
        GPIO_Configuration();
        FSMC_Configuration();
        
        /*LCD硬件设备初始化代码*/
    
        return (RT_EOK);
    }
    
    /*  LCD设备控制  */
    static rt_err_t lcd_control(rt_device_t dev, rt_uint8_t cmd, void *args)
    {
        switch(cmd)
        {
            case RTGRAPHIC_CTRL_GET_INFO:  /* 返回LCD屏幕信息 */
            {
                struct rt_device_graphic_info *info;
                
                info = (struct rt_device_graphic_info*)args;
                RT_ASSERT(info != RT_NULL);
                
                info->bits_per_pixel = lcd_device.bits_per_pixel;
                info->pixel_format   = lcd_device.pixel_format;
                info->framebuffer    = RT_NULL;
                info->width          = lcd_device.width;
                info->height         = lcd_device.height;
            }
            break;
            
            default:
                break;
        }
        
        return (RT_EOK);
    }
~~~
***编写5个基本的LCD操作函数，并用它们初始化rt_device_graphic_ops***
~~~
    /*画点*/
    void rt_hw_lcd_set_pixel(const char * pixel, int x, int y)
    {
        LCD_SetCursor(x,y);
        LCD_WriteRAM_Prepare();
        
        LCD_WriteRAM(*(rt_uint16_t*)pixel);
    }

    /*获取点颜色*/
    void rt_hw_lcd_get_pixel(char * pixel, int x, int y)
    {
        *(rt_uint16_t*)pixel = LCD_ReadPoint(x,y);
    }
    
    /*画水平线*/
    void rt_hw_lcd_draw_hline(const char * pixel, int x1, int x2, int y)
    {
        LCD_SetCursor(x1,y);
        LCD_WriteRAM_Prepare();
        
        while(x1<x2)
        {
            LCD_WriteRAM(*(rt_uint16_t*)pixel);
            x1 ++;
        }
    }
    
    /*画垂直线*/
    void rt_hw_lcd_draw_vline(const char * pixel, int x, int y1, int y2)
    {
        LCD_SetCursor(x,y1);
        LCD_WriteRAM_Prepare();
        while(y1<y2)
        {
            LCD_WriteRAM(*(rt_uint16_t*)pixel);
            y1 ++;
        }
    }
        
    /*画起始水平线*/
    void rt_hw_lcd_draw_raw_hline(const char * pixels, int x, int y, rt_size_t size)
    {
        rt_uint16_t * ptr;
        ptr = (rt_uint16_t*)pixels;
        
        LCD_SetCursor(x,y);
        LCD_WriteRAM_Prepare();
        
        while(size)
        {
            LCD_WriteRAM(*ptr++);
            size --;
        }
    }
    
    /*实现图形设备操作结构体*/
    struct rt_device_graphic_ops  lcd_ili_ops = 
    {
        rt_hw_lcd_set_pixel,              /*画点*/
        rt_hw_lcd_get_pixel,              /*获取点颜色*/
        rt_hw_lcd_draw_hline,             /*画水平线*/
        rt_hw_lcd_draw_vline,             /*画垂直线*/
        rt_hw_lcd_draw_raw_hline          /*画起始水平线（填充一行数据）*/
    };
~~~
***调用rt_hw_lcd_init()向RTT注册LCD设备，初始化并打开LCD设备***
~~~
    Void  rt_hw_lcd_init(void)
    {
        rt_device_t lcd;
    	
        /*初始化LCD设备结构体*/
        lcd_device.parent.type      = RT_Device_Class_Graphic;  /*设备类型*/
        lcd_device.parent.init      = lcd_init;
        lcd_device.parent.open      = lcd_open;
        lcd_device.parent.close     = lcd_close;
        lcd_device.parent.read      = RT_NULL;
        lcd_device.parent.write     = RT_NULL;
        lcd_device.parent.control   = lcd_control;
        lcd_device.parent.user_data = &lcd_ili_ops; /*设置用户私有数据*/
    	
        lcd_device.pixel_format   = RTGRAPHIC_PIXEL_FORMAT_RGB565P;
    	lcd_device.bits_per_pixel = LCD_BITS_PER_PIXEL;
        
        /*注册图形设备驱动*/
        rt_device_register(&(lcd_device.parent), 
                                "lcd",
                                RT_DEVICE_FLAG_RDWR | RT_DEVICE_FLAG_STANDALONE);
        lcd = rt_device_find("lcd");  /*查找LCD设备               */
        
        /*初始化*/
        rt_device_init(lcd);
        rt_device_open(lcd,RT_DEVICE_OFLAG_RDWR);
    }
~~~
***验证LCD设备驱动程序***

到此为止，已经成功将LCD设备添加到RTT的设备管理器中，可以在UIENGINE中操作LCD。

下面通过在液晶上绘制一个窗口对驱动进行验证：

    - 在void rt_init_thread_entry(void* parameter)函数最后添加以下代码，初始化LCD。
~~~
    	rt_device_t lcd; 

    	rt_hw_lcd_init();             /*初始化并注册LCD设备       */

    	lcd = rt_device_find("lcd");  /*查找LCD设备               */
        if(lcd == RT_NULL)
        {
            rt_kprintf("no graphic device in the system.\n");
        }
        rtgui_graphic_set_device(lcd); /*设置LCD设备为GUI驱动     */
        
        rtgui_system_server_init(); /*启动UIENGINE系统服务 */
~~~
    - 在application.c中新建一个LCD测试线程入口函数
~~~
    void  LCD_demo_entry(void* parameter)
    {
            
        struct rtgui_app *app;
        struct rtgui_win *win;
        struct rtgui_box *box;
        
        struct rtgui_rect rect={0,20,240,320};
        
        app = rtgui_app_create("MyApp");
        RT_ASSERT(app != RT_NULL);
        
        win = rtgui_win_create(RT_NULL, "MyWindow", &rect, RTGUI_WIN_STYLE_DEFAULT);
        
        box = rtgui_box_create(RTGUI_HORIZONTAL, 20);
        rtgui_container_set_box(RTGUI_CONTAINER(win), box);
        
        rtgui_container_layout(RTGUI_CONTAINER(win));
        
        rtgui_win_show(win, RT_FALSE);
        
        rtgui_app_run(app);
        
        rtgui_win_destroy(win);
        rtgui_app_destroy(app);
        
        rt_kprintf("MyApp Quit.\n");
    }
~~~
    - 将下面代码添加到rt_application_init()函数最后。
~~~
     #ifdef RT_USING_GUIENGINE
            tid = rt_thread_create("gui_demo",
                                gui_application_entry,
                                RT_NULL,
                                2048*8,
                                25,
                                10);
        	if(tid != RT_NULL)
            	rt_thread_startup(tid);
     #endif
~~~
将程序下载到测试板上，可以在LCD中看到绘制好的窗口：

### framebuffer帧缓冲屏驱动
RT_Thread对天生具备支持frambuffer帧缓冲屏的功能。在RT-Thread下编写frambuffer帧缓冲屏的驱动程序需要遵循以下框架。

**LCD设备结构体rt_lcd_device**

framebuffer帧缓冲屏的rt_lcd_device结构体包含LCD的基本信息，其原型如下：
~~~
    struct rt_lcd_device
    {
        rt_uint8_t   pixel_format;       /*像素格式*/
        rt_uint8_t   bits_per_pixel;     /*每个像素占用位数*/
        rt_uint16_t  width;              /*LCD宽度，横向像素个数*/
        rt_uint16_t  height;             /*LCD高度，纵向像素个数*/
        
        void *framebuffer;               /*帧缓存*/
    };
    typedef struct rt_lcd_device rt_lcd_t;
    
    struct rt_lcd_device  lcd_device;
~~~

**硬件接口函数**

    这两个接口函数是所有RTT管理的设备都具备的公共函数，RTT的设备还具备其它的公共函数，但对于LCD设备，我们只需要填充这两个函数。

    函数功能：①.初始化和lcd连接的处理器的接口；②.初始化液晶硬件。
    函数定义：static rt_err_t lcd_init(rt_device_t dev)。
    参数说明：dev，lcd设备。

    函数功能：设置lcd的基本参数。
    函数定义：static rt_err_t lcd_control(rt_device_t dev, rt_uint8_t cmd, void *args)。
    参数说明：dev，lcd设备；
             cmd，操作命令；
             args，操作参数。

**向RTT注册LCD设备，初始化并打开LCD设备**

    写完以上3个内容以后，调用rt_hw_lcd_init()，向RTT注册LCD设备，初始化并打开LCD设备，便可以在RT-Thread中正常的使用LCD了。
~~~
    void rt_hw_lcd_init(void)
    {
        rt_device_t lcd ;

        lcd_device.bits_per_pixel = 16;
        lcd_device.pixel_format = RTGRAPHIC_PIXEL_FORMAT_RGB565P;
        lcd_device.framebuffer = (void*)rt_lcd_framebuffer;
        lcd_device.width = LCD_WIDTH;
        lcd_device.height = LCD_HEIGHT;

        /* init device structure */
        lcd->type = RT_Device_Class_Graphic;
        lcd->init = rt_lcd_init;
        lcd->open = RT_NULL;
        lcd->close = RT_NULL;
        lcd->control = rt_lcd_control;
        lcd->user_data = (void*)&lcd_device;

        /* register lcd device to RT-Thread */
        rt_device_register(lcd, "lcd", RT_DEVICE_FLAG_RDWR);


        lcd = rt_device_find("lcd");  /*查找LCD设备               */

        /*初始化*/
        rt_device_init(lcd);
        rt_device_open(lcd,RT_DEVICE_OFLAG_RDWR);
    }
~~~

**以S3C2440为例，编写一个LCD驱动程序**

按照上述操作步骤，编写一个RT-Thread的LCD驱动程序。

开发板：JZ2440V2，4.3寸LCD

***定义LCD帧缓存***
~~~
    #define LCD_WIDTH 480   // xres
    #define LCD_HEIGHT 272  // yres
    static volatile rt_uint16_t rt_lcd_framebuffer[LCD_WITH][LCD_HEIGHT];
~~~

***初始化lcd设备结构体***
~~~
    struct rt_lcd_device
    {
        rt_uint8_t  pixel_format;
        rt_uint8_t  bits_per_pixel;
        rt_uint16_t width;
        rt_uint16_t height;
        
        void *framebuffer;
    };
    struct rt_lcd_device  lcd_device;
~~~
***编写2个硬件接口函数***
~~~
    /*  LCD设备初始化  */
    static rt_err_t rt_lcd_init(rt_device_t dev)
    {
        LCD_Port_Configuration();  //配置LCD屏幕连接的引脚端口
        LCD_Ctrl_Configuration();  //配置LCD控制器
        
        LCD_Enable();  //使能LCD显示
        
        return (RT_EOK);
    }
    
    /*  LCD设备控制  */
    static rt_err_t rt_lcd_control(rt_device_t dev, rt_uint8_t cmd, void *args)
    {
        switch (cmd)
        {
        case RTGRAPHIC_CTRL_RECT_UPDATE:
            break;
        case RTGRAPHIC_CTRL_POWERON:
            break;
        case RTGRAPHIC_CTRL_POWEROFF:
            break;
        case RTGRAPHIC_CTRL_GET_INFO:		
            rt_memcpy(args, &lcd_device, sizeof(lcd_device));
            break;
        case RTGRAPHIC_CTRL_SET_MODE:
            break;
        }
        
        return (RT_EOK);
    }
~~~
***调用rt_hw_lcd_init()向RTT注册LCD设备，初始化并打开LCD设备***
~~~
    Void  rt_hw_lcd_init(void)
    {
        rt_device_t lcd ;

        lcd_device.bits_per_pixel = 16;
        lcd_device.pixel_format = RTGRAPHIC_PIXEL_FORMAT_RGB565P;
        lcd_device.framebuffer = (void*)rt_lcd_framebuffer;
        lcd_device.width = LCD_WIDTH;
        lcd_device.height = LCD_HEIGHT;

        /* init device structure */
        lcd->type = RT_Device_Class_Graphic;
        lcd->init = rt_lcd_init;
        lcd->open = RT_NULL;
        lcd->close = RT_NULL;
        lcd->control = rt_lcd_control;
        lcd->user_data = (void*)&lcd_device;

        /* register lcd device to RT-Thread */
        rt_device_register(lcd, "lcd", RT_DEVICE_FLAG_RDWR);


        lcd = rt_device_find("lcd");  /*查找LCD设备               */

        /*初始化*/
        rt_device_init(lcd);
        rt_device_open(lcd,RT_DEVICE_OFLAG_RDWR);
    }
~~~
***验证LCD设备驱动程序***
验证驱动程序和pixel像素屏驱动的验证程序一样，如果屏幕尺寸不一样，只需要调整屏幕的显示范围即可。



## 注意事项
