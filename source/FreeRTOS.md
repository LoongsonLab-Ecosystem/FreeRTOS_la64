
# 项目介绍
## FreeRTOS  
FreeRTOS（Free Real-Time Operating System）是一款开源的实时操作系统内核。LoongArch是由我国龙芯中科研发的自主指令系统（龙芯架构）。  
本项目将FreeRTOS移植到LoongArch平台上，在Ubuntu 20.04中通过QEMU模拟器（在PC上模拟LoongArch硬件）编译FreeRTOS并运行。  

## 配置交叉编译环境  
本实验所用的交叉编译工具链为：  
https://github.com/loongson/build-tools/releases/download/2025.08.08/x86_64-cross-tools-loongarch64-binutils_2.45-gcc_15.1.0-glibc_2.42.tar.xz  
下载后通过共享文件夹到Ubuntu  
设置路径：  
```
export PATH="{/path/to/cross-tools}/bin:$PATH"
```
上述的命令只是临时设置环境变量，如需永久设置，可通过修改/etc/profile实现；   
## 编译FreeRTOS
下载FreeRTOS-loongarch
```
git clone git@github.com:bzy666-666/FreeRTOS-loongarch.git
cd FreeRTOS-loongarch
```
开始编译
```
make all
```
运行
```
make run
```

## 结构目录
```
FreeRTOS_loongArch/  
├── Makefile                  
├── link.ld                   
├── start.S                    
├── FreeRTOSConfig.h           
│  
├── app/                       # [应用层]  
│   └── main.c                 
│  
└── FreeRTOS-Kernel/           # [内核层] 官方源码 (不要动!)  
    ├── tasks.c                # 任务调度核心  
    ├── queue.c                # 队列通信  
    ├── list.c                 # 链表实现  
    ├── timers.c               # 软件定时器  
    │  
    ├── include/               # [头文件]   
    │   ├── FreeRTOS.h  
    │   ├── task.h  
    │   └── ...    
    │  
    └── portable/              # [移植层]   
        ├── MemMang/  
        │   └── heap_4.c       # 内存管理 (pvPortMalloc)  
        │  
        └── GCC/    
            └── LoongArch64/   # [核心移植]    
                ├── portmacro.h # 数据类型定义  
                ├── port.c      # C接口: 栈初始化, 定时器配置    
                └── portASM.S   # 汇编接口: 上下文切换, ISR入口   
```

# 移植主要工作

## port.c
### 初始化任务栈
```c
StackType_t *pxPortInitialiseStack( StackType_t *pxTopOfStack, TaskFunction_t pxCode, void *pvParameters )
{
    uint64_t ullStackTop = ( uint64_t ) pxTopOfStack;
    ullStackTop &= ~( ( uint64_t ) 0x0f );
    pxTopOfStack = ( StackType_t * ) ullStackTop;

    pxTopOfStack -= 34; // 32 GPRs + ERA + PRMD

    for (int i = 0; i < 34; i++) {
        pxTopOfStack[i] = 0;
    }
    
    pxTopOfStack[0] = 0;

    pxTopOfStack[1] = (StackType_t) prvTaskExitError;

    pxTopOfStack[4] = (StackType_t) pvParameters;

    pxTopOfStack[22] = 0;

    pxTopOfStack[32] = (StackType_t) pxCode;

    pxTopOfStack[33] = 0x4;

    return pxTopOfStack;
}
```
对传入的pxTopOfStack进行16字节对齐，预留出32个通用寄存器+ERA寄存器（存跳转的地址）和PRMD寄存器（设置优先级），将预留了上下文的栈顶地址传回。
  
### 启动调度器 
```c
BaseType_t xPortStartScheduler( void )
{
    uxCriticalNesting = 0;

    vPortSetupTimerInterrupt();

    xPortStartFirstTask();

    return 0;
}

void vPortEndScheduler( void )
{

}
```
配置硬件定时器中断
```c
void vPortSetupTimerInterrupt( void )
{
    unsigned long ulConstFreq = configCPU_CLOCK_HZ; 
    unsigned long ulTickValue = ulConstFreq / configTICK_RATE_HZ;
    unsigned long ulTCFG = (ulTickValue << 2) | 3;
    __asm__ volatile ( "csrwr %0, 0x41" : : "r"(ulTCFG) );


    unsigned long ulECFG;
    __asm__ volatile ( "csrrd %0, 0x4" : "=r"(ulECFG) );
    ulECFG |= (1 << 11);
    __asm__ volatile ( "csrwr %0, 0x4" : : "r"(ulECFG) );
}
```
在启动前，将嵌套计数清零，防止死锁，配置硬件定时中断，配置时间中断配置寄存器TCFG，以及例外配置寄存器ECFG,最后启动第一个任务。

### 临界区管理
```c
void vPortEnterCritical( void )
{
    portDISABLE_INTERRUPTS();
    uxCriticalNesting++;
}

void vPortExitCritical( void )
{
    if( uxCriticalNesting > 0 )
    {
        uxCriticalNesting--;
        if( uxCriticalNesting == 0 )
        {
            portENABLE_INTERRUPTS();
        }
    }
}
```
进入临界区禁用中断、增加嵌套计数，确保关键代码段不被中断打断，退出临界区，减少嵌套计数，当计数值为0，说明完全退出，可以重新启用中断。  

## portASM.S
### 启动第一个任务
```
xPortStartFirstTask:

RESTORE_CONTEXT  
    
.align 4
```
RESTORE_CONTEXT
```
.macro RESTORE_CONTEXT
    la.global $t0, pxCurrentTCB
    ld.d      $t1, $t0, 0       
    ld.d      $sp, $t1, 0       

    ld.d      $t0, $sp, 8*33
    csrwr     $t0, 0x1          /* CSR_PRMD */
    ld.d      $t0, $sp, 8*32
    csrwr     $t0, 0x6          /* CSR_ERA */

    ld.d    $r1,  $sp, 8*1
    ld.d    $r2,  $sp, 8*2
    ld.d    $r4,  $sp, 8*4
    ld.d    $r5,  $sp, 8*5
    ld.d    $r6,  $sp, 8*6
    ld.d    $r7,  $sp, 8*7
    ld.d    $r8,  $sp, 8*8
    ld.d    $r9,  $sp, 8*9
    ld.d    $r10, $sp, 8*10
    ld.d    $r11, $sp, 8*11
    ld.d    $r12, $sp, 8*12
    ld.d    $r13, $sp, 8*13
    ld.d    $r14, $sp, 8*14
    ld.d    $r15, $sp, 8*15
    ld.d    $r16, $sp, 8*16
    ld.d    $r17, $sp, 8*17
    ld.d    $r18, $sp, 8*18
    ld.d    $r19, $sp, 8*19
    ld.d    $r20, $sp, 8*20
    ld.d    $r21, $sp, 8*21
    ld.d    $r22, $sp, 8*22
    ld.d    $r23, $sp, 8*23
    ld.d    $r24, $sp, 8*24
    ld.d    $r25, $sp, 8*25
    ld.d    $r26, $sp, 8*26
    ld.d    $r27, $sp, 8*27
    ld.d    $r28, $sp, 8*28
    ld.d    $r29, $sp, 8*29
    ld.d    $r30, $sp, 8*30
    ld.d    $r31, $sp, 8*31

    addi.d  $sp, $sp, 272

    ertn
.endm
```
从pxCurrentTCB的第0偏移读取八字节得到SP，恢复特殊寄存器（CSR_PRMD、CSR_ERA），以及通用寄存器，最后回收栈空间，通过ertn跳转。

### 异常处理入口函数
```
freertos_loongarch_trap_handler:
    SAVE_CONTEXT

    csrrd   $t0, 0x5
    
    srli.d  $t1, $t0, 16
    andi    $t1, $t1, 0x3F
    li.d    $t2, 0xB
    beq     $t1, $t2, handle_yield

    li.d    $t2, 0x800      /* 1 << 11 */
    and     $t1, $t0, $t2
    bnez    $t1, handle_tick

    b       exit_trap

handle_yield:
    ld.d    $t0, $sp, 8*32  /* 从栈里读出保存的 ERA */
    addi.d  $t0, $t0, 4     /* 跳过当前指令 (4字节) */
    st.d    $t0, $sp, 8*32  /* 写回栈里 */

    bl      vTaskSwitchContext
    b       exit_trap

handle_tick:
    li.d    $t0, 1
    csrwr   $t0, 0x44 

    bl      xTaskIncrementTick

    beq     $r4, $zero, exit_trap
    
    bl      vTaskSwitchContext
    b       exit_trap

exit_trap:
    RESTORE_CONTEXT
```
SAVE_CONTEXT
```
.macro SAVE_CONTEXT
    addi.d  $sp, $sp, -272

    st.d    $r1,  $sp, 8*1   /* ra */
    st.d    $r2,  $sp, 8*2   /* tp */

    st.d    $r4,  $sp, 8*4   /* a0 */
    st.d    $r5,  $sp, 8*5
    st.d    $r6,  $sp, 8*6
    st.d    $r7,  $sp, 8*7
    st.d    $r8,  $sp, 8*8
    st.d    $r9,  $sp, 8*9
    st.d    $r10, $sp, 8*10
    st.d    $r11, $sp, 8*11
    st.d    $r12, $sp, 8*12
    st.d    $r13, $sp, 8*13
    st.d    $r14, $sp, 8*14
    st.d    $r15, $sp, 8*15
    st.d    $r16, $sp, 8*16
    st.d    $r17, $sp, 8*17
    st.d    $r18, $sp, 8*18
    st.d    $r19, $sp, 8*19
    st.d    $r20, $sp, 8*20
    st.d    $r21, $sp, 8*21
    st.d    $r22, $sp, 8*22  /* fp */
    st.d    $r23, $sp, 8*23
    st.d    $r24, $sp, 8*24
    st.d    $r25, $sp, 8*25
    st.d    $r26, $sp, 8*26
    st.d    $r27, $sp, 8*27
    st.d    $r28, $sp, 8*28
    st.d    $r29, $sp, 8*29
    st.d    $r30, $sp, 8*30
    st.d    $r31, $sp, 8*31

    csrrd   $t0, 0x6   /* CSR_ERA */
    st.d    $t0, $sp, 8*32
    csrrd   $t0, 0x1   /* CSR_PRMD */
    st.d    $t0, $sp, 8*33

    la.global $t0, pxCurrentTCB
    ld.d      $t1, $t0, 0       
    st.d      $sp, $t1, 0       
.endm
```
在start.S指定这里为EENTRY 指向的地方，当有异常就跳转到这里，首先保存现场SAVE_CONTEXT，首先开辟出34*8=272字节的空间，保存34个寄存器，再将当前的SP保存到TCB中。读取CSR_ESTAT判断是谁触发的： A. 检查 Syscall (Yield) -> ECODE 位 (21:16) 是否为 0xB，进入handle_yield； B. 检查 Timer 中断 -> 检查中断标志位，进入handle_tick。
```
handle_yield:
    ld.d    $t0, $sp, 8*32 
    addi.d  $t0, $t0, 4     
    st.d    $t0, $sp, 8*32  

    bl      vTaskSwitchContext
    b       exit_trap
```
从栈里读出保存的era地址，加载下一条指令的地址（4字节），保存到ern寄存器。切换上下文
```
handle_tick:
    li.d    $t0, 1
    csrwr   $t0, 0x44 

    bl      xTaskIncrementTick

    beq     $r4, $zero, exit_trap
    
    bl      vTaskSwitchContext
    b       exit_trap
```
将定时中断清除寄存器的0位置1，清楚中断标记，告诉FreeRTOS时间流逝，如果 xTaskIncrementTick 返回非0，说明需要切换任务，然后切换上下文。

# 测试代码


## 以app/simple_main.c为例介绍FreeRTOS运行过程
```c
int main(void) {

    uart_init();

    uart_print("\r\n=== LoongArch64 FreeRTOS Demo ===\r\n");



    xTaskCreate(vTask1, "Task1", 1024, "Task1", 1, NULL);

    xTaskCreate(vTask2, "Task2", 1024, "Task2", 2, NULL);



    uart_print("Starting Scheduler...\r\n");

    vTaskStartScheduler();



    /* 不应该运行到这里 */

    for(;;);

    return 0;

}
```
由link.ld，进入_start，做了：
1. 设置栈指针
2. 将异常函数入口地址存入CSR_EENTRY寄存器
3. 清空bss段
4. 跳转到main函数

通过xTaskCreate创建两个任务（进程）
1. 参数1是实现函数，参数2是函数名称，参数3是栈得深度，参数4是任务优先级，参数5是任务句柄（不需要就是NULL）
2. 在task.c中调用xTaskCreate完成了：
a. 创建任务prvCreateTask：申请TCB的内存以及申请的1024栈深度的内存，返回指向分配的内存块的起始地址的指针，然后调用prvInitialiseNewTask初始化这个任务，将该任务的栈顶设置为刚才分配的1024内存指向的地址（需要地址对齐），同样设置了栈底。将任务名称、优先级、放到TCB里，初始化状态链表和事件链表(看一下实现)，最后进入port.c的pxPortInitialiseStack初始化任务栈
b. 添加任务到就序列表prvAddNewTaskToReadyList：如果当前的tcb为NULL，则创建的任务为当前就绪任务，否则当调度器还没有调度时，判断当前任务和添加的任务的优先级，选择优先级高的为当前的就绪任务。
3. 在task.c中调用vTaskStartScheduler完成调度的开启，最后调用xPortStartScheduler，启动优先级最高的任务（这里是Task2）。

当Task2调用vTaskDelay(pdMS_TO_TICKS(500))时，会将Task2从就序列表放到延迟列表并设置延迟时间，然后触发taskyield，进入异常处理函数的入口，进行上下文切换判断是否需要切换到Task1。

完整测试代码默认为app/main.c。运行结果如下图：  
![这是图片](./imgs/f-2.png "完整测试代码结果图")    

## 任务通知
我们使用队列、信号量、事件组等等方法时，并不知道对方是谁。使用任务通知时，可以明确指定：通知哪个任务。
定义：
#define xTaskNotify( xTaskToNotify, ulValue, eAction ) \
    xTaskGenericNotify( ( xTaskToNotify ), ( tskDEFAULT_INDEX_TO_NOTIFY ), ( ulValue ), ( eAction ), NULL )
#define ulTaskNotifyTake( xClearCountOnExit, xTicksToWait ) \
    ulTaskGenericNotifyTake( ( tskDEFAULT_INDEX_TO_NOTIFY ), ( xClearCountOnExit ), ( xTicksToWait ) )
```c
BaseType_t xTaskGenericNotify( TaskHandle_t xTaskToNotify,
                               UBaseType_t uxIndexToNotify,
                               uint32_t ulValue,
                               eNotifyAction eAction,
                               uint32_t * pulPreviousNotificationValue )
```
xTaskToNotify：要通知的目标任务句柄  
uxIndexToNotify：通知索引（支持多个通知）  
ulValue：通知携带的值  
eAction：如何处理通知值  
pulPreviousNotificationValue：输出参数，返回之前的值  
eAction参数说明：  
:::{list-table} eNotifyAction 取值与功能详解
:widths: 25 75
:header-rows: 1

* - eNotifyAction取值
  - 说明
* - eNoAction
  - 仅仅是更新通知状态为“pending”，未使用 `ulValue`。这个选项相当于轻量级的、更高效的二进制信号量。
* - eSetBits
  - 通知值 = 原来的通知值 | `ulValue`，按位或。相当于轻量级的、更高效的事件组。
* - eIncrement
  - 通知值 = 原来的通知值 + 1，未使用 `ulValue`。相当于轻量级的、更高效的二进制信号量、计数型信号量。相当于 xTaskNotifyGive() 函数。
* - eSetValueWithoutOverwrite
  - 不覆盖。如果通知状态为“pending”(表示有数据未读)，则此次调用 `xTaskNotify` 不做任何事，返回 `pdFAIL`。如果通知状态不是“pending”(表示没有新数据)，则：通知值 = `ulValue`。
* - eSetValueWithOverwrite
  - 覆盖。无论如何，不管通知状态是否为“pending”，通知值 = `ulValue`。
:::

```c
uint32_t ulTaskGenericNotifyTake( UBaseType_t uxIndexToWaitOn,
                                  BaseType_t xClearCountOnExit,
                                  TickType_t xTicksToWait )
```
uxIndexToWaitOn：要等待的通知索引  
xClearCountOnExit：退出时如何处理计数值  
xTicksToWait：最大等待时间  
### 测试代码
```c
xTaskCreate(vTaskReceiver, "Receiver", 1024, NULL, 4, &xReceiverTaskHandle);
xTaskCreate(vTaskSender,   "Sender",   1024, NULL, 3, NULL);

void vTaskSender(void *pvParameters)
{
    uint32_t ulLoopCount = 0;

    for( ;; )
    {
        vTaskDelay( 2000 );
        ulLoopCount++;

        if( xReceiverTaskHandle != NULL )
        {
            uart_print("Sender: Raw Generic Notify Call...\r\n");

            if( ulLoopCount % 2 != 0 )
            {
                xTaskGenericNotify( 
                    xReceiverTaskHandle, 
                    0, 
                    0,              /* 忽略 */
                    eIncrement,     /* ++ */
                    NULL 
                );
            }
            /* 2. 偶数次循环：发送具体数据 (Index 0, Action=Overwrite) */
            else
            {
                xTaskGenericNotify( 
                    xReceiverTaskHandle, 
                    0, 
                    0x88888888,                 
                    eSetValueWithOverwrite,     
                    NULL 
                );
            }
        }
    }
}


void vTaskReceiver(void *pvParameters)
{
    uint32_t ulNotifiedValue;

    for( ;; )
    {
      
        ulNotifiedValue = ulTaskGenericNotifyTake( 
            0,              /* uxIndexToWaitOn: 等待 Index 0 */
            pdTRUE,         /* xClearCountOnExit: 必须为 TRUE (二值模式/全清模式) */
            portMAX_DELAY   /* xTicksToWait: 死等 */
        );

        if( ulNotifiedValue == 0x88888888 )
        {
            uart_print("Receiver: Got DATA overwrite: 0x88888888\r\n");
        }
        else if( ulNotifiedValue > 0 )
        {
            uart_print("Receiver: Got SEMAPHORE increment. Value is now cleared.\r\n");
        }
        else
        {
            uart_print("Receiver: Timeout (Wait returned 0)\r\n");
        }
    }
}
```
xTaskGenericNotify通过对传入的任务句柄的tcb进行操作（pxTCB = xTaskToNotify）。
当奇数次循环，对eAction进行判断为eIncrement，对传入的ulValue（0），进行++操作，此时ulvalue为1，如果任务的原始通知状态是等待，就放入就绪列表，然后触发任务切换，当偶数次循环对eAction进行判断为eSetValueWithOverwrite强制覆盖通知值ulvalue为0x88888888，同样如果任务的原始通知状态是等待，就放入就绪列表，然后触发任务切换，此时高优先级的vTaskReceiver开始调度，首先判断如果没有收到数据并且需要等待，则把当前任务从就绪列表 (Ready List) 移除，如果有数据将ulReturn设置为发送者传入的数据，再将数据清零，返回ulReturn。  
### 运行结果
![这是图片](./imgs/f-1.png "任务通知结果图")   
