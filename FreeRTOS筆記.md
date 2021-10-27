==Task Switching Latency== means the time gap between the triggering of an event and the time at which the task, which takes care of that event, is allowed to run on the CPU. 

==Interrupt Latency== is the time gap between the time at which interrupt triggers and the time at which ISR (Interrupt Service Routine) is started executing on the CPU.

The time gap between the moment at which ISR or task releases the CPU (t3) and the time the new task is switching into the CPU (t4) is called a ==Scheduling Latency==.


both the Scheduling Latency and the Interrupt Latency of the RTOS are bounded, and it won’t increase as the system load increases. But in the GPOS, it is unbounded, and the latency may vary as the system load increases.

[http://fastbitlab.com/rtos-vs-gpos-latency/](https://)

Cortex-M 內核快速中斷指令

PRIMASK
FAULTMASK
BASEPRI


portDISABLE_INTERRUPTS():此函數不能在中斷中使用



---

FreeRTOS是設計給許多不同的處理器架構的RTOS，很多人會跑在ARM Cortex上，但正因為他是設計通用不同的處理器架構的，所以有些東西要特別注意。

第一件事就是優先權的數量，這是處理器的製造商決定的，就算都是使用Cortex-M的處理器，製造商允許的優先權數量也不同。如同我們一開始所講的，Cotex-M本身允許高達256個優先權。但大多數處理器製造商並不會全部實作出來，你通常只能使用3~5bits來設定優先權，也就是8、16、32個不同的優先權數量，取決於微處理器的製造商。

要了解你的處理器使用多少優先權，可以去看你廠商的文件，或是直接看你使用的CMSIS Library header files。可以看core_cm4.h這個檔中定義了__NVIC_PRIO_BITS的數值。如果是stm32f429，這個值是4，也就是允許16個不同大小的優先權。

[擷取自..](http://opass.logdown.com/posts/248297-talking-about-the-priority-from-the-arm-set-cortex-m-to-freertos)


---


FreeRTOS的優先權分成兩類。一類是受RTOS管理，不會影響到critcal section的，另一類是不理會RTOS的。這兩者將以configMAX_SYSCALL_INTERRUPT_PRIORITY所設定的值為界。

configKERNEL_INTERRUPT_PRIORITY
configMAX_SYSCALL_INTERRUPT_PRIORITY



根據Cortex-M內核自身的情況進行設置，要以最高有效位對齊。比如某微控制器使用中斷優先級寄存器中的3位，設置configKERNEL_INTERRUPT_PRIORITY的值爲5，則代碼爲：

#define     configKERNEL_INTERRUPT_PRIORITY  （5<<(8-3)）
[擷取自..](https://www.twblogs.net/a/5eee9fb4418820fe02fa05ad)





以stm32f407為例，cpu只允許中斷優先級寄存器被使用4個bit，從FreeRTOSConfig.h可以看到

/* Cortex-M specific definitions. */
#ifdef __NVIC_PRIO_BITS
	/* __BVIC_PRIO_BITS will be specified when CMSIS is being used. */
	#define configPRIO_BITS       		__NVIC_PRIO_BITS
#else
	#define configPRIO_BITS       		4        /* 15 priority levels */
#endif

#define configKERNEL_INTERRUPT_PRIORITY 		( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

因為是四個位元，所以順位最低的數值為0xf，然而又因為Cortex-M內核要以最高有效位對齊，所以會先往左邊位移


設定完configKERNEL_INTERRUPT_PRIORITY參數後，在FreeRTOS中SysTick和PenSV的優先順序都是最低的，因此剛好可以使用configKERNEL_INTERRUPT_PRIORITY來設定這兩個中斷的暫存器。

/*PendSV優先級設置寄存器地址爲0xe000ed22
 SysTick優先級設置寄存器地址爲0xe000ed23*/

一個寄存器(register)地址為32-bit，空間為8-bit

#define portNVIC_SHPR3_REG                    ( *( ( volatile uint32_t * ) 0xe000ed20 ) )

#define portNVIC_PENDSV_PRI                   ( ( ( uint32_t ) configKERNEL_INTERRUPT_PRIORITY ) << 16UL )
往左邊shift 16bits，因為記憶體空間是連續的，所以等於從原本的0xe000ed20移動到0xe000ed22，下面的

#define portNVIC_SYSTICK_PRI                  ( ( ( uint32_t ) configKERNEL_INTERRUPT_PRIORITY ) << 24UL )

portNVIC_SHPR3_REG |= portNVIC_PENDSV_PRI;
portNVIC_SHPR3_REG |= portNVIC_SYSTICK_PRI;




---

BASEPRI:
在FreeRTOS中，對中斷的開或關都是透過寫入暫存器BASEPRI，意思是大於或等於BASEPRI的值的中斷都會被屏蔽。

==FreeRTOS臨界區的作用是保證位於臨界區內的代碼在執行過程中不被其它中斷或者任務打斷==。臨界區的作用類似於互斥鎖，不同的是臨界區是通過直接操作寄存器來屏蔽中斷，而互斥鎖只是通過軟件來實現對共享資源的保護。

那些需要在中斷調用時保護的API函數，FreeRTOS使用寄存器BASEPRI實現中斷保護臨界區。當進入臨界區時，將寄存器BASEPRI的值設置成configMAX_SYSCALL_INTERRUPT_PRIORITY，當退出臨界區時，將寄存器BASEPRI的值設置成0。很多Bug反饋都提到，當退出臨界區時不應該將寄存器設置成0，應該恢復它之前的狀態（之前的狀態不一定是0）。但是Cortex-M NVIC決不會允許一個低優先級中斷搶占當前正在執行的高優先級中斷，不管BASEPRI寄存器中是什麼值。==與進入臨界區前先保存BASEPRI的值，退出臨界區再恢復的方法相比，退出臨界區時將BASEPRI寄存器設置成0的方法可以獲得更快的執行速度==。

RTOS內核通過寫configMAX_SYSCALL_INTERRUPT_PRIORITY的值到BASEPRI寄存器的方法創建臨界區。中斷優先級0（具有最高的邏輯優先級）不能被BASEPRI寄存器屏蔽，因此，configMAX_SYSCALL_INTERRUPT_PRIORITY絕不可以設置成0。
[擷取自..](https://blog.csdn.net/zhzht19861011/article/details/50135449)



---

portDISABLE_INTERRUPTS():此例中configMAX_SYSCALL_INTERRUPT_PRIORITY = 0x50, 及優先級大於5的中斷都會被屏蔽

```
#define portDISABLE_INTERRUPTS()                  vPortRaiseBASEPRI()

portFORCE_INLINE static void vPortRaiseBASEPRI( void )
    {
        uint32_t ulNewBASEPRI;

        __asm volatile
        (
            "	mov %0, %1												\n"\
            "	msr basepri, %0											\n"\
            "	isb														\n"\
            "	dsb														\n"\
            : "=r" ( ulNewBASEPRI ) : "i" ( configMAX_SYSCALL_INTERRUPT_PRIORITY ) : "memory"
        );
    }

```

    


---

中斷級進入前保存當前的BASEPRI值，並給其賦值configMAX_SYSCALL_INTERRUPT_PRIORITY，實現關閉比configMAX_SYSCALL_INTERRUPT_PRIORITY優先級低的中斷。退出的時候需要給BASEPRI賦值進入臨界區代碼前的值。也就是中斷級無法嵌套。

任務級進入前關閉中斷，而不保存當前BASEPRI的初始值，每次嵌套都要保存嵌套了幾次。當全部嵌套退出後，給BASEPRI賦值0，打開中斷。


taskENTER_CRITICAL()和taskEXIT_CRITICAL()是任務級的臨界區程式碼保護，一個是進入臨界區，一個是退出臨界區，這兩個函數一定要成對的使用。

taskENTER_CRITICAL_FROM_ISR()和taskEXIT_CRITICAL_FROM_ISR()是中斷級別臨界區程式碼保護，是用在中斷服務程式中的，而且這個中斷的優先順序一定要低於預先設定的值。


每次發生tick interrupt都會呼叫xTaskIncrementTick()
变量uxSchedulerSuspended是定义在tasks.c文件中的静态变量，记录调度器运行状态
