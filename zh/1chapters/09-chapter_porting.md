# 移植 #

想要让RT-Thread在某款芯片上运行起来，必须先要把RT-Thread在这款芯片上移植好。本章讲述RT-Thread移植相关事项，同时以ARM Cortex-M3为例给出如何移植RT-Thread到一个新的芯片上。

## 使用移植还是自己移植 ##

通常大多数人还没分清楚是使用一个移植还是自己进行移植，以为要把RT-Thread跑在自己的板子上都需要进行移植。例如RT-Thread官方已经提供了STM32F103ZET6的移植，而自己的目标板使用的是STM32F103C8T6的芯片，所以觉得自己应该把RT-Thread“移植”到自己的板子上。

孰不知，STM32F103ZET6和STM32F103C8T6对操作系统的内核差别仅在于SRAM的容量大小。如果使用了系统动态内存堆（即上层应用需要使用rt_malloc函数），仅仅是这个系统动态内存堆的结束地址不一样。其他部分，例如ARM Cortex-M3部分是完全一样的。还有不一样，需要自己仔细检查的地方包括：

* 主晶振使用的震荡频率是多少；
* 外设的pin引脚是否相同；

明白这些之后就知道，要想把已经支持STM32F103ZET6的RT-Thread，在STM32F103C8T6上运行起来，只需要修改这些相关的地方即可。和传统意义的移植相差甚远，可以重用STM32F103ZET6的大部分移植代码。

所以在想移植RT-Thread到一款新型号的芯片之前，应该了解下，RT-Thread是否已经支持类似的处理器，如果已经支持了，那么仅需要把不同的地方修改即可。

## 移植前的准备 ##

在移植之前，应该对RT-Thread的目录结构有一定的了解：

    RT-Thread
    +---bsp
    |   \--- stm32f40x
    +---components
    +---include
    +---libcpu
    |   \---arm
    |       \---cortex-m4
    +---src
    \---tools

其中，include和src目录用于放置RT-Thread的实时核心；components目录用于放置各类组件；tools是用于放置RT-Thread的构建环境scons的一些扩展脚本；bsp和libcpu则是移植相关的部分。

**注**

通常来说，对于一个移植除了bsp、libcpu目录以外，其他的目录和文件不应该被修改。只有在你需要支持一种新的编译器，才可能修改到include\\rtdef.h和finsh等相关的代码。当要支持一种新的编译器，同时希望包括在开发分支内时，请联系内核的维护人以解决相关的问题，或给与适当的指导。

在了解了RT-Thread的目录，以及知道自己应该修改哪里的代码后，应该了解RT-Thread移植的两种模式：

* 使用RT-Thread中的libcpu目录；这个时候，和CPU相关的移植放在libcpu目录下的相对应的子目录中，自己的移植通过scons的SConscript脚本或工程文件使用这个目录下的libcpu文件；
* 不使用RT-Thread中的libcpu目录，例如希望使用自己的CPU移植，或这份CPU移植不会放到开发分支上。

对于第二种情况，可以按照如下的方式组织自己的移植：

    YourProject
    +---applications
    +---components
    +---cpu
    +---drivers
    +---documents
    +---Libraries
    \---rt-thread
        +---tools
        +---include
        +---components
        \---src

从这个目录结构可以看到，rt-thread的相关目录被做为一个相对独立的目录放在工程目录下面。同时在自己的工程目录中，包含：

* applications，用于放置用户应用的目录；
* components，用于放置用户自己的组件；
* cpu，替代原来的libcpu目录，放置芯片移植相关的代码、驱动；
* driver，用户自行编写的驱动；
* documents，用户文档；
* Libraries，一些相对固定的库文件；

需要注意的一点是，按照这样的使用方式，需要在SConstruct文件中加入`has_libcpu = True`的选项：

    # prepare building environment
    objs = PrepareBuilding(env, RTT_ROOT, has_libcpu=True)

这样后续不管是使用scons进行编译或者使用scons生成工程文件去编译，都将不会在rt-thread\\libcpu中芯片相关的这部分代码。

## RT-Thread在ARM Cortex M3上的移植 ##

### 建立RealView MDK工程 ###

在bsp目录下新建project目录。在RealView MDK中新建立一个工程文件（用菜单创建），名称为project，保存在bsp\\your_board目录下。创建工程时一次的选项如下： CPU选择STMicroelectronics的STM32F103ZE：
 
提问复制STM32的启动代码到工程目录，确认Yes
 
然后选择工程的属性，如下图

Select Folder for Objects目录选择到bsp\\your_board\\build，Name of Executable为rtthread-stm32
 
同样Select Folder for Listings选择stm32radio\\examples\\project\\objs目录，如下图所示：
 
C/C++编译选项标签页中，因为在项目中使用了ST的STM32固件库，需要在Define中添加STM32F10X_HD, USE_STDPERIPH_DRIVER的定义；在Include Paths（头文件搜索路径） 中添加上..\\..\\STM32F10x_Libraries\\STM32F10x_StdPeriph_Driver\\inc;..\\..\\STM32F10x_Libraries\\CMSIS\\CM3\\CoreSupport;..\\..\\STM32F10x_Libraries\\CMSIS\\CM3\\DeviceSupport\\ST\\STM32F10x的路径包含；以及RT-Thread的头文件路径和STM32移植目录的路径：..\\..\\rt-thread\\include;..\\..\\rt-thread\\stm32目录，如下图所示：
 
Asm，Linker，Debug和Utilities选项使用初始配置即可。

### 添加源文件 ###

* 添加STM32固件库源文件

新添加Group：STM32_StdPeriph，然后把stm32radio\\STM32F10x_Libraries\\CMSIS和stm32radio\\STM32F10x_Libraries\\STM32F10x_StdPeriph_Driver\\src目录中的所有C源文件都添加到Group中。

* 添加RT-Thread相关源文件

对工程中初始添加的Source Group1改名为Startup，并添加Kernel，STM32的Group，开始建立工程时产生的STM32F10x.s重命名为start_rvds.s并放到stm32radio\\rt-thread\\libcpu\\arm\\ stm32目录中。
Kernel Group中添加所有stm32radio\\rt-thread\\kernel下的C源文件； Startup Group中添加startup.c，board.c文件（放于stm32radio\\examples\\project目录中）； STM32中添加context_rvds.s，start_rvds.s，fault_rvds.s，cpu.c,fault.c, interrupt.c, serial.c, stack.c等文件（放于stm32radio\\rt-thread\\stm32目录中）；

* 添加RT-Thread配置头文件

在stm32radio\\examples\\project目录中添加rtconfig.h文件，内容如下：

~~~{.c}
/* RT-Thread配置文件 */
#ifndef __RTTHREAD_CFG_H__
#define __RTTHREAD_CFG_H__

/* 内核对象名称最大长度 */
#define RT_NAME_MAX             8
/* 数据对齐长度 */
#define RT_ALIGN_SIZE           4
/* 最大支持的优先级：32 */
#define RT_THREAD_PRIORITY_MAX  32
/* 每秒的节拍数 */
#define RT_TICK_PER_SECOND      100

/* SECTION: 调试选项 */
/* 打开RT-Thread的ASSERT选项 */
#define RT_DEBUG
/* 打开RT-Thread的线程栈溢出检查 */
#define RT_USING_OVERFLOW_CHECK
/* 使用钩子函数 */
#define RT_USING_HOOK

/* SECTION: 线程间通信选项  */
/* 支持信号量 */
#define RT_USING_SEMAPHORE
/* 支持互斥锁 */
#define RT_USING_MUTEX
/* 支持事件标志 */
#define RT_USING_EVENT
/* 支持邮箱 */
#define RT_USING_MAILBOX
/* 支持消息队列 */
#define RT_USING_MESSAGEQUEUE

/* SECTION: 内存管理  */
/* 支持静态内存池 */
#define RT_USING_MEMPOOL
/* 支持动态内存堆管理 */
#define RT_USING_HEAP
/* 使用小型内存管理算法 */
#define RT_USING_SMALL_MEM

/* SECTION: 设备模块选项 */
/* 支持设备模块 */
#define RT_USING_DEVICE
/* 支持串口1设备 */
#define RT_USING_UART1
/* SECTION: 控制台选项  */
#define RT_USING_CONSOLE
/* 控制台缓冲区大小 */
#define RT_CONSOLEBUF_SIZE  128

#endif
~~~

* 启动代码

在Keil MDK自动生成的启动代码中，由于STM32的中断处理方式是以中断向量表的方式进行，所以将不再使用中断统一入口的方式进行，启动代码可以大部分使用这份启动代码。主要修改在：

1. 栈尺寸

MDK生成的启动代码默认的栈尺寸是0x200，如果中断服务例程使用的栈尺寸需要不高，可以使用默认值。
Stack_Size      EQU     0x00000200

2. PenSV异常。由于PenSV异常在context_rvds.S中实现，需要更改名称为rt_hw_pend_sv。

3. HardFault异常

HardFault异常直接保留代码也没关系，只是当系统出现了fault异常时，并不容易看到。为完善代码起见，把它更改为rt_hw_hard_fault。

4. 时钟中断

OS时钟在Cortex-M3中使用了统一的中断方式：SysTick。需要把它指向RT-Thread中正确的时钟中断处理函数：rt_hw_timer_handler。

相应的更改如下：（rt-thread\\libcpu\\arm\\stm32\\start_rvds.S更改部分代码清单）

~~~{.c}
    IMPORT  rt_hw_hard_fault
    IMPORT  rt_hw_pend_sv
    IMPORT  rt_hw_timer_handler

; Vector Table Mapped to Address 0 at Reset
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size

__Vectors     DCD     __initial_sp                  ; Top of Stack
                DCD     Reset_Handler               ; Reset Handler
                DCD     NMI_Handler                 ; NMI Handler
                DCD     rt_hw_hard_fault            ; Hard Fault Handler
                DCD     MemManage_Handler           ; MPU Fault Handler
                DCD     BusFault_Handler            ; Bus Fault Handler
                DCD     UsageFault_Handler          ; Usage Fault Handler
                DCD     0                           ; Reserved
                DCD     0                           ; Reserved
                DCD     0                           ; Reserved
                DCD     0                           ; Reserved
                DCD     SVC_Handler                 ; SVCall Handler
                DCD     DebugMon_Handler            ; Debug Monitor Handler
                DCD     0                           ; Reserved
                DCD     rt_hw_pend_sv               ; PendSV Handler in RT-Thread
                DCD     rt_hw_timer_handler         ; SysTick Handler in RT-Thread
~~~

* 栈初始化代码

栈的初始化代码用于创建线程或初始化线程，“手工”的构造一份线程运行栈，相当于在线程栈上保留了一份线程从初始位置运行的上下文信息。在Cortex-M3体系结构中，当系统进入异常时，CPU Core会自动进行R0 – R3以及R12、psr、pc、lr等压栈，所以手工构造这个初始化栈时，也相应的把这些寄存器初始值放置到正确的位置。

rt-thread\\libcpu\\arm\\stm32\\stack.c程序清单：

~~~{.c}
#include <rtthread.h>

/**
 * @addtogroup STM32
 */
/*@{*/

/**
 * 这个函数用于初始化线程栈
 *
 * @param tentry 线程的入口函数
 * @param parameter 线程入口函数的参数
 * @param stack_addr 栈的初始化地址
 * @param texit 当线程退出时的处理函数
 *
 * @return 返回准备好的栈初始指针
 */
rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter,
    rt_uint8_t *stack_addr, void *texit)
{
    unsigned long *stk;

    stk      = (unsigned long *)stack_addr;
    *(stk)   = 0x01000000L;                     /* PSR */
    *(--stk) = (unsigned long)tentry;           /* pc : 函数入口 */
    *(--stk) = (unsigned long)texit;            /* lr : 线程退出时运行位置 */
    *(--stk) = 0;                               /* r12 */
    *(--stk) = 0;                               /* r3 */
    *(--stk) = 0;                               /* r2 */
    *(--stk) = 0;                               /* r1 */
    *(--stk) = (unsigned long)parameter;        /* r0 : 入口参数 */

    *(--stk) = 0;                               /* r11 */
    *(--stk) = 0;                               /* r10 */
    *(--stk) = 0;                               /* r9 */
    *(--stk) = 0;                               /* r8 */
    *(--stk) = 0;                               /* r7 */
    *(--stk) = 0;                               /* r6 */
    *(--stk) = 0;                               /* r5 */
    *(--stk) = 0;                               /* r4 */

    /* 返回栈初始位置 */
    return (rt_uint8_t *)stk;
}

/*@}*/
~~~

最终形成的线程栈情况如下：

* 上下文切换代码

代码清单rt-thread\\libcpu\\arm\\stm32\\context_rvds.S：
在RT-Thread中，中断锁是完全有芯片移植来实现的，参见 线程间同步与通信章节。以下是Cortex-M3上的开关中断实现，它是使用CPSID指令来实现的。

~~~{.c}
; rt_base_t rt_hw_interrupt_disable();
; 关闭中断
rt_hw_interrupt_disable    PROC
        EXPORT  rt_hw_interrupt_disable
        MRS     r0, PRIMASK                     ; 读出PRIMASK值，即返回值
        CPSID   I                               ; 关闭中断
        BX      LR
        ENDP

; void rt_hw_interrupt_enable(rt_base_t level);
; 恢复中断
rt_hw_interrupt_enable    PROC
        EXPORT  rt_hw_interrupt_enable
        MSR     PRIMASK, r0                     ; 恢复R0寄存器的值到PRIMASK中
        BX      LR
        ENDP
~~~

在Cortex M3微处理器中，当系统进入异常时，CPU Core会自动进行R0 – R3以及R12、PSR、PC、LR等压栈，所以CM3的线程上下文切换正可以利用硬件压栈的特点，让机器自动帮助完成部分工作：在线程正常的上下文切换时，触发一个PenSV中断，从而进入到PenSV处理程序中。

~~~{.c}
; void rt_hw_context_switch(rt_uint32 from, rt_uint32 to);
; void rt_hw_context_switch_interrupt(rt_uint32 from, rt_uint32 to);
; r0 --> from
; r1 --> to
; 在Cortex M3移植中，这两个函数的内容都是相同的，
; 因为正常模式的切换也采取了触发PenSV异常的方式进行
rt_hw_context_switch_interrupt
        EXPORT rt_hw_context_switch_interrupt
rt_hw_context_switch    PROC
        EXPORT rt_hw_context_switch

        ; 设置参数rt_thread_switch_interrput_flag为1，
        ; 代表将要发起线程上下文切换
        LDR     r2, =rt_thread_switch_interrput_flag
        LDR     r3, [r2]
        CMP     r3, #1                          ; 参数已经置1，说明线程切换已经触发
        BEQ     _reswitch
        MOV     r3, #1
        STR     r3, [r2]

        LDR     r2, =rt_interrupt_from_thread   ; 保存切换出线程栈指针
        STR     r0, [r2]                        ; （切换过程中需要更新到当前位置）

_reswitch
        LDR     r2, =rt_interrupt_to_thread     ; 保存切换到线程栈指针
        STR     r1, [r2]

        LDR     r0, =NVIC_INT_CTRL
        LDR     r1, =NVIC_PENDSVSET
        STR     r1, [r0]                        ; 触发PendSV异常
        BX      LR
        ENDP

; PendSV异常处理
; r0 --> swith from thread stack
; r1 --> swith to thread stack
; psr, pc, lr, r12, r3, r2, r1, r0 等寄存器已经被自动压栈
; 到切换出线程栈中
rt_hw_pend_sv   PROC
        EXPORT rt_hw_pend_sv

        ; 为了保护线程切换，先关闭中断
        MRS     r2, PRIMASK
        CPSID   I

        ; 获得rt_thread_switch_interrupt_flag参数，
        ; 以判断pendsv是否已经处理过
        LDR     r0, =rt_thread_switch_interrput_flag
        LDR     r1, [r0]
        CBZ     r1, pendsv_exit             ; pendsv已经被处理，直接退出

        ; 清除参数：rt_thread_switch_interrput_flag为0
        MOV     r1, #0x00
        STR     r1, [r0]

        LDR     r0, =rt_interrupt_from_thread
        LDR     r1, [r0]
        CBZ     r1, swtich_to_thread        ; 如果切换出线程为0，这是第一次上下文切换

        MRS     r1, psp                     ; 获得切换出线程栈指针
        STMFD   r1!, {r4 - r11}             ; 对剩余的R4 – R11寄存器压栈
        LDR     r0, [r0]
        STR     r1, [r0]                    ; 更新切换出线程栈指针

    swtich_to_thread
        LDR     r1, =rt_interrupt_to_thread
        LDR     r1, [r1]
        LDR     r1, [r1]                    ; 载入切换到线程的栈指针到R1寄存器

        LDMFD   r1!, {r4 - r11}             ; 恢复R4 – R11寄存器
        MSR     psp, r1                     ; 更新程序栈指针寄存器

    pendsv_exit
        ; 恢复中断
        MSR     PRIMASK, r2

        ORR     lr, lr, #0x04               ; 构造LR以返回到Thread模式
        BX      lr                          ; 从PendSV异常中返回
        ENDP

; void rt_hw_context_switch_to(rt_uint32 to);
; r0 --> to
; 切换到函数，仅在第一次调度时调用
rt_hw_context_switch_to    PROC
        EXPORT rt_hw_context_switch_to
rt_hw_context_switch_to    PROC
        EXPORT rt_hw_context_switch_to
        LDR     r1, =rt_interrupt_to_thread     ; 设置切换到线程
        STR     r0, [r1]

        LDR     r1, =rt_interrupt_from_thread   ; 设置切换出线程栈为0
        MOV     r0, #0x0
        STR     r0, [r1]

        LDR     r0, =NVIC_SYSPRI2               ; 设置优先级
        LDR     r1, =NVIC_PENDSV_PRI
        STR     r1, [r0]

        LDR     r0, =NVIC_INT_CTRL
        LDR     r1, =NVIC_PENDSVSET
        STR     r1, [r0]                        ; 触发PendSV异常

        CPSIE   I                               ; 使能中断以使PendSV能够正常处理
        ENDP
~~~

正常模式下的线程上下文切换的过程可以用下图来表示：
 
当要进行切换时（假设从Thread from 切换到Thread to），通过rt_hw_context_switch()函数触发一个PenSV异常。异常产生时，Cortex M3会把PSR，PC，LR，R0 – R3，R12自动压入当前线程的栈中，然后切换到PenSV异常处理。到PenSV异常后，Cortex M3工作模式（从Thread模式）切换到Handler模式，由函数rt_hw_pend_sv进行处理。rt_hw_pend_sv函数会载入切换出线程（Thread from）和切换到线程（Thread to）的栈指针，如果切换出线程的栈指针是0那么表示这是系统启动时的第一次线程上下文切换，不需要对切换出线程做压栈动作。如果切换出线程栈指针非零，则把剩余未压栈的R4 – R11寄存器依次压栈；然后从切换到线程栈中恢复R4 – R11寄存器。当从PendSV异常返回时，PSR，PC，LR，R0 – R3，R12等寄存器由Cortex M3自动恢复。

因为中断而导致的线程切换可用下图表示：

当中断达到时，当前线程会被中断并把PC，PSR，R0 – R3，R12等压到当前线程栈中，工作模式（从Thread模式）切换到Handler模式。

在运行中断服务例程期间，如果发生了线程切换（调用rt_schedule()），会先判断当前工作模式是否是Handler模式（依赖于全局变量rt_interrupt_nest），如果是则调用rt_hw _context_switch_interrupt函数进行伪切换：
在rt_hw_context_switch_interrupt函数中，将把当前线程栈指针赋值到rt_interrupt_from_thread变量上，把要切换过去的线程栈指针赋值到rt_interrupt_ to_thread变量上，并设置中断中线程切换标志rt_thread_switch_interrput_flag为1。
在最后一个中断服务例程结束时，Cortex M3将去处理PendSV异常，因为PendSV异常的优先级是最低的，所以只要触发过PendSV异常，它将总是在最后得到处理。
Fault处理代码
Fault处理代码并不是必须的，为了系统的完整性，实现fault处理代码无疑对系统出错时定位问题提供非常有利的帮助。
rt-thread\\libcpu\\arm\\stm32\\fault.S代码清单：

~~~{.c}
    AREA |.text|, CODE, READONLY, ALIGN=2
    THUMB
    REQUIRE8
    PRESERVE8

    IMPORT rt_hw_hard_fault_exception

rt_hw_hard_fault    PROC
    EXPORT rt_hw_hard_fault

    ; 获得线程栈上压入的上下文，并赋值到R0
    MRS     r0, psp
    PUSH    {lr}                        ; 压入LR寄存器
    BL      rt_hw_hard_fault_exception  ; 调用fault处理的C函数
    POP     {lr}

    ORR     lr, lr, #0x04               ; 从fault中返回
    BX      lr
    ENDP

    END
~~~

rt-thread\\libcpu\\arm\\stm32\\fautl.c代码清单：

~~~{.c}
#include <rtthread.h>

/* CM3硬件压栈时的寄存器结构 */
struct stack_contex
{
    rt_uint32_t r0;
    rt_uint32_t r1;
    rt_uint32_t r2;
    rt_uint32_t r3;
    rt_uint32_t r12;
    rt_uint32_t lr;
    rt_uint32_t pc;
    rt_uint32_t psr;
};

extern void list_thread(void);
extern rt_thread_t rt_current_thread;
void rt_hw_hard_fault_exception(struct stack_contex* contex)
{
    /* 输出出错时的寄存器情况，PC、LR、PSR应重点关注  */
    rt_kprintf("psr: 0x%08x\n", contex->psr);
    rt_kprintf(" pc: 0x%08x\n", contex->pc);
    rt_kprintf(" lr: 0x%08x\n", contex->lr);
    rt_kprintf("r12: 0x%08x\n", contex->r12);
    rt_kprintf("r03: 0x%08x\n", contex->r3);
    rt_kprintf("r02: 0x%08x\n", contex->r2);
    rt_kprintf("r01: 0x%08x\n", contex->r1);
    rt_kprintf("r00: 0x%08x\n", contex->r0);

    /* 输出当前线程的名称 */
    rt_kprintf("hard fault on thread: %s\n", rt_current_thread->name);
#ifdef RT_USING_FINSH
    /* 列表系统中的线程情况 */
    list_thread();
#endif
    /* 进入死循环，如果去掉while(1)，那么系统将能够从fault中返回 */
    while (1);
}
~~~

## RT-Thread/STM32其他部分说明 ##

RT-Thread/STM32移植是基于RealView MDK开发环境进行移植的（GNU GCC编译器和IAR ARM编译亦支持），和STM32相关的代码大多采用RealView MDK中的代码，例如start rvds.s是从RealView MDK自动添加的启动代码中修改而来。
和RT-Thread以往的ARM移植不一样的是，系统底层提供的rt_hw 系列函数相对要少些，建议可以考虑使用成熟的库（例如针对STM32芯片，可以采用ST官方的固件库）。RTThread/STM32工程中已经包含了STM32f10x系列3.1.x的库代码，可以配套使用。
和中断相关的rt_hw函数（异常与中断章节中的大多数函数）本移植中并不具备，所以可以跳过OS层直接操作硬件。在编写中断服务例程时，推荐使用如下的模板：

~~~{.c}
void rt_hw_interrupt_xx_handler(void)
{
    /* 通知RT-Thread进入中断模式 */
    rt_interrupt_enter();
    
    /* ... 中断处理 */
    
    /* 通知RT-Thread离开中断模式 */
    rt_interrupt_leave();
}
~~~

rt_interrupt_enter()函数会通知OS进入到中断处理模式（相应的线程切换行为会有些变化）；rt_interrupt_leave()函数会通知OS离开了中断处理模式。
