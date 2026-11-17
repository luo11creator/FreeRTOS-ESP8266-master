# 基于 STM32F407 + FreeRTOS + ESP8266 的物联网实时气象站系统

[![MCU](https://img.shields.io/badge/MCU-STM32F407-blue.svg)](https://www.st.com/)
[![RTOS](https://img.shields.io/badge/RTOS-FreeRTOS%20(CMSIS--RTOS%20v1)-brightgreen.svg)](https://www.freertos.org/)
[![Network](https://img.shields.io/badge/WiFi-ESP8266%20(AT)-orange.svg)](https://www.espressif.com/)
[![Parser](https://img.shields.io/badge/JSON-cJSON-yellow.svg)](https://github.com/DaveGamble/cJSON)
[![License](https://img.shields.io/badge/License-SLA0044-lightgrey.svg)](https://www.st.com/SLA0044)

## 📖 目录

1. [项目简介](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-1-项目简介)

2. [系统硬件架构与外设分配](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-2-系统硬件架构与外设分配)

3. [软件工作流程与核心协议](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-3-软件工作流程与核心协议)

4. FreeRTOS 核心基础知识与本项目深度结合剖析

   (🔥核心重点)

   - [4.1 为什么本项目需要 FreeRTOS？（裸机 vs RTOS）](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#41-为什么本项目需要-freertos裸机-vs-rtos)
   - [4.2 双时钟源机制：SysTick 与 TIM1 的职责分离](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#42-双时钟源机制systick-与-tim1-的职责分离)
   - [4.3 任务管理（Task Management）与状态机](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#43-任务管理task-management与状态机)
   - [4.4 延时机制剖析：`HAL_Delay` vs `osDelay`（开机提速的秘密）](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#44-延时机制剖析hal_delay-vs-osdelay开机提速的秘密)
   - [4.5 中断管理与延迟处理：二值信号量（Binary Semaphore）](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#45-中断管理与延迟处理二值信号量binary-semaphore)
   - [4.6 FreeRTOS 内存管理与 cJSON 冲突的终极解决方案](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#46-freertos-内存管理与-cjson-冲突的终极解决方案)
   - [4.7 其他必备 FreeRTOS 核心组件（队列、互斥量、事件组）](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#47-其他必备-freertos-核心组件队列互斥量事件组)

5. [核心代码解析与健壮性优化建议](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-5-核心代码解析与健壮性优化建议)

6. [实现效果](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-6-实现效果)

7. [更新日志](https://arena.ai/c/01a1e995-f7fa-7a0c-86c7-2de20a1131a6#-7-更新日志)

------

## 📌 1. 项目简介

本项目是一个基于 **STM32F407（Cortex-M4 内核）** 微控制器，搭载 **FreeRTOS 实时操作系统** 的物联网（IoT）实时气象站终端。

系统通过 **ESP8266 Wi-Fi 模块** 接入互联网，以 TCP 透传模式向 **心知天气（Seniverse）API** 发起 HTTP GET 请求获取实时气象报文；利用轻量级 **cJSON** 解析库提取关键气象数据（天气现象、实时气温等）；同时结合片上 **RTC（实时时钟）** 提供高精度万年历/计时功能；最终通过 **淘晶驰（TJC）HMI 串口屏** 实现数据的可视化交互显示。

------

## 🛠️ 2. 系统硬件架构与外设分配

### 2.1 所需硬件设备

- **主控芯片**：STM32F407ZGT6 / VET6 开发板（主频配置：HSE 8MHz →→ PLL倍频 →→ HCLK 84MHz）
- **显示终端**：淘晶驰（TJC）智能串口屏
- **通信模块**：ESP8266-01S / NodeMCU Wi-Fi 模块（烧录官方 AT 固件）
- **调试工具**：ST-Link 下载器、USB 转 TTL 串口模块

### 2.2 外设与引脚资源映射表

根据 `main.c` 中的初始化配置，片上外设分配如下：

| 硬件外设    | 连接模块                 | 波特率/配置    | 触发方式                    | 核心功能描述                                            |
| :---------- | :----------------------- | :------------- | :-------------------------- | :------------------------------------------------------ |
| **USART1**  | PC 上位机 (串口助手)     | 115200, 8-N-1  | 轮询发送 (`printf` 重定向)  | 打印系统运行日志、AT指令回显、调试状态输出              |
| **USART2**  | ESP8266 Wi-Fi 模块       | 115200, 8-N-1  | **单字节接收中断 (`IT`)**   | 发送 AT 控制指令，接收心知天气返回的 JSON 数据流        |
| **USART3**  | 淘晶驰 HMI 串口屏        | 9600/115200    | **单字节接收中断 + 信号量** | 发送 UI 控件更新指令，接收触摸屏下发的控制指令          |
| **RTC**     | 内部 LSI 时钟源 (~32kHz) | 异步/同步分频  | 任务周期轮询读取            | 提供系统本地实时时钟（时、分、秒计时功能）              |
| **TIM1**    | 内部时钟                 | 1kHz (1ms周期) | 定时器更新中断              | **作为 HAL 库的基础时基（Timebase）**，维护 `uwTick`    |
| **SysTick** | Cortex-M4 内核滴答定时器 | 1kHz (1ms周期) | 内核异常中断                | **作为 FreeRTOS 系统心跳节拍（OS Tick）**，驱动任务调度 |

### 2.3 系统架构拓扑图

text



```
+-------------------------------------------------------------------------+
|                            STM32F407 微控制器                           |
|                                                                         |
|  +--------------+     UART2 (AT指令/JSON流)      +-------------------+  |
|  |              | <============================> |   ESP8266 Wi-Fi   |  |
|  |              |      中断接收至 my_re_buf       +---------+---------+  |
|  |              |                                          | TCP / HTTP |
|  |   FreeRTOS   |     UART3 (HMI指令/0xFF结尾)   +---------v---------+  |
|  |  实时操作系统 | <============================> |  心知天气云服务器  |  |
|  |  (任务调度、  |   中断检测\r\n释放 myBinarySem01 +-------------------+  |
|  |   内存管理、  |                                                       |
|  |   信号量同步) |     UART3 TX (刷新UI控件)      +-------------------+  |
|  |              | =============================> | 淘晶驰 HMI 串口屏 |  |
|  |              |                                +-------------------+  |
|  |              |     UART1 TX (printf调试日志)  +-------------------+  |
|  |              | =============================> |   PC 串口调试助手  |  |
|  +------+-------+                                +-------------------+  |
|         | 读取时间                                                      |
|  +------v-------+                                                       |
|  |  片上 RTC    |                                                       |
|  +--------------+                                                       |
+-------------------------------------------------------------------------+
```

------

## 🔄 3. 软件工作流程与核心协议

### 3.1 ESP8266 AT 指令状态机流程

系统通过串口 2 依次发送以下 AT 指令将 ESP8266 配置为 **TCP 透传客户端**：

1. `AT+CWMODE=1\r\n`：设置为 Station（STA）工作模式。

2. `AT+RST\r\n`：软复位模块使模式生效。

3. `AT+CWJAP="SSID","PASSWORD"\r\n`：连接指定的路由器热点。

4. `AT+CIPMUX=0\r\n`：开启单路连接模式（透传模式前提）。

5. `AT+CIPSTART="TCP","api.seniverse.com",80\r\n`：与心知天气服务器（端口80）建立 TCP 握手连接。

6. `AT+CIPMODE=1\r\n`：开启透明传输模式（Transparent Transmission）。

7. `AT+CIPSEND\r\n`：启动透传数据发送，此后串口收发的数据直接等同于 TCP 载荷。

8. 发送 HTTP GET 请求报文

   ：

   http

   

   ```
   GET https://api.seniverse.com/v3/weather/now.json?key=您的私钥&location=chengdu&language=zh-Hans&unit=c
   ```

### 3.2 心知天气 JSON 数据结构与 cJSON 解析

服务器响应的标准 JSON 格式如下：

JSON



```
{
  "results": [
    {
      "location": {
        "id": "WM6N2PM3WY2K",
        "name": "成都",
        "country": "CN",
        "path": "成都,成都,四川,中国",
        "timezone": "Asia/Shanghai",
        "timezone_offset": "+08:00"
      },
      "now": {
        "text": "多云",
        "code": "4",
        "temperature": "26"
      },
      "last_update": "2020-06-27T14:20:00+08:00"
    }
  ]
}
```

在代码中，我们定义了多级 `cJSON` 指针（`json` →→ `Array_obj1` →→ `Array` →→ `Array_obj2` 等），通过层层剥离字典（Object）与数组（Array），最终提取出 `"text"`（天气现象）和 `"temperature"`（温度）字段。

### 3.3 淘晶驰（TJC）串口屏通信协议

淘晶驰串口屏采用**属性赋值指令 + 3字节 `0xFF` 结束符**的通信协议：

- **发送文本更新指令**：在 `HMISends()` 中发送形如 `page0.t0.txt="12:30:00"` 的字符串。
- **发送指令结束符**：在 `HMISendb(0xff)` 中，通过 `for(i=0; i<3; i++)` 连续向串口 3 发送三个 `0xFF` 字节（即 `0xFF 0xFF 0xFF`），通知串口屏一帧指令传输完毕并立即执行渲染。

------

## 🧠 4. FreeRTOS 核心基础知识与本项目深度结合剖析

> **💡 导读**：本章节将脱离单纯的 API 调用，结合本项目的 `main.c` 源码，深度剖析 FreeRTOS 的内核机制、任务调度、同步通信及内存管理原理。

### 4.1 为什么本项目需要 FreeRTOS？（裸机 vs RTOS）

在传统的**裸机开发（前后台系统 / Super Loop）**中，程序结构通常是一个庞大的 `while(1)` 循环加上各种中断服务函数（ISR）。如果用裸机实现本气象站，会面临致命痛点：

1. **长耗时阻塞导致 UI 卡死**：ESP8266 连接 Wi-Fi 和等待 HTTP 响应通常需要数秒时间（代码中使用了 `3500ms` 等待）。在裸机中使能 `HAL_Delay(3500)` 会导致 CPU 原地空转，此时 **RTC 秒表计时会完全停滞**，屏幕触摸按键也会失去响应。
2. **模块耦合度极高**：网络重连、JSON 解析、屏幕刷新、RTC 读取全部揉杂在一起，任何一个环节卡顿都会拖垮全局实时性。

**引入 FreeRTOS 后的优势**：
FreeRTOS 是一个**可剥夺型（抢占式）实时多任务内核**。它将系统的不同功能划分为独立的**任务（Task/Thread）**。每个任务拥有自己独立的栈空间（Stack）和优先级（Priority）。当网络任务在等待 ESP8266 响应时，它会主动让出 CPU 使用权（进入阻塞态），调度器（Scheduler）会立即切换到 RTC 时钟任务去刷新屏幕上的秒表，从而在单核 CPU 上实现**宏观上的“并发执行”**。

------

### 4.2 双时钟源机制：SysTick 与 TIM1 的职责分离

仔细观察 `main.c` 底部的这段代码：

C



```
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM1) {
    HAL_IncTick();
  }
}
```

#### ❓ 为什么要专门用硬件定时器 `TIM1` 来调用 `HAL_IncTick()`？

- **滴答定时器（SysTick）被 FreeRTOS 独占**：FreeRTOS 需要一个周期性的硬件中断作为系统的**心跳节拍（OS Tick，通常配置为 1000Hz 即 1ms 一次）**。每次 SysTick 中断触发时，FreeRTOS 的 `xPortSysTickHandler()` 会递增系统节拍计数器 `xTickCount`，并检查是否有延时到期的任务需要唤醒，或者是否需要进行同优先级任务的时间片轮转调度（Time Slicing）。
- **HAL 库也需要时基**：STM32 的 HAL 库内部大量外设驱动（如 `HAL_UART_Transmit` 的超时判断、`HAL_Delay`）依赖于全局变量 `uwTick` 的自增。
- **冲突解决**：如果在 SysTick 中同时运行 FreeRTOS 调度和 HAL 库时基，当在中断或临界区中调用某些 HAL 函数时，极易引发死锁或优先级翻转。因此，STM32CubeMX 的最佳实践是：**将 `SysTick` 完全移交给 FreeRTOS 管理（中断优先级设为最低 `15`），另外开辟一个硬件定时器（本项目中为 `TIM1`，中断优先级设高）专门负责每 1ms 调用一次 `HAL_IncTick()`**。

------

### 4.3 任务管理（Task Management）与状态机

在 `main()` 函数中，硬件初始化完成后，调用了两个关键函数：

C



```
MX_FREERTOS_Init(); // 创建任务、信号量、队列等内核对象
osKernelStart();    // 启动 FreeRTOS 任务调度器（底层调用 vTaskStartScheduler()）
```

一旦执行 `osKernelStart()`，CPU 的控制权就正式移交给了 FreeRTOS 调度器，程序**永远不会**执行到后面的 `while(1)` 循环中。

#### 1. 任务的四种基本状态（Task States）

在我们的气象站系统中，各个任务（如：*天气获取任务*、*RTC时钟刷新任务*、*串口屏交互任务*）时刻在以下四种状态间切换：

text



```
                   被调度器选中 (获得CPU控制权)
         +---------------------------------------------+
         |                                             v
   +-----+-----+                                 +-----------+
   |  就绪态    | <-------------------------------+  运行态    |
   |  (Ready)  |   被更高优先级抢占 / 时间片耗尽    | (Running) |
   +-----+-----+                                 +-----+-----+
         ^                                             |
         | 事件发生 (如串口3释放信号量)                  | 调用 osDelay(3500) 或
         | 或 延时时间到期                              | 调用 osSemaphoreWait() 等待数据
         |                                             v
   +-----+-----+                                 +-----------+
   |  阻塞态    |                                 |  挂起态    |
   | (Blocked) |                                 |(Suspended)|
   +-----------+                                 +-----------+
   (不消耗任何CPU资源)                           (仅通过 osThreadSuspend 进出)
```

- **运行态（Running）**：当前正在使用 Cortex-M4 CPU 内核寄存器的任务（同一时刻只能有一个）。
- **就绪态（Ready）**：任务已经具备运行条件，没有被阻塞，只是因为当前有更高优先级的任务正在运行，所以在就绪列表中排队等待。
- **阻塞态（Blocked）**：任务正在等待某个**时间事件**（如调用了 `osDelay(3500)` 等待 ESP8266 响应）或**外部同步事件**（如等待串口屏发来指令的信号量 `myBinarySem01Handle`）。**处于阻塞态的任务完全不占用 CPU 时间！**
- **挂起态（Suspended）**：类似“暂停/休眠”，调度器完全忽略它，除非其他任务显式调用 `osThreadResume()` 恢复它。

#### 2. 调度算法（Scheduling Algorithm）

本项目默认采用 **抢占式（Preemptive） + 时间片轮转（Time Slicing）** 调度机制：

- **高优先级抢占**：一旦高优先级任务（例如：处理触摸屏紧急指令的任务）因获得信号量从 `Blocked` 变为 `Ready`，内核会立刻触发 `PendSV` 异常，保存当前低优先级任务的现场（压栈到其任务栈 TCB 中），并强制让高优先级任务进入 `Running` 状态。
- **同优先级时间片轮转**：如果两个任务优先级相同且都处于 `Ready` 状态，它们将轮流执行，每次执行 1 个系统节拍（1ms）。

------

### 4.4 延时机制剖析：`HAL_Delay` vs `osDelay`（开机提速的秘密）

在项目的 `main.c` 中，您编写了两组功能极其相似的函数，这恰恰反映了您在 **2020.6.27 更新日志（“修复了开机启动慢的问题，加快了开机速度”）** 中的核心重构逻辑！

让我们对比这两段代码：

C



```
/* 裸机阻塞式发送（旧方案） */
void command(char Command_AT[]) {
    int Lenth = strlen(Command_AT);
    HAL_UART_Transmit(&huart2, (uint8_t*)Command_AT, Lenth, 0xff);
    printf("%s", Command_AT);
    HAL_Delay(3500); // ⚠️ 致命的死等！
}

/* FreeRTOS 阻塞式发送（新方案） */
void FreeRTOS_command(char Command_AT[]) {
    int Lenth = strlen(Command_AT);
    HAL_UART_Transmit(&huart2, (uint8_t*)Command_AT, Lenth, 0xff);
    printf("%s", Command_AT);
    osDelay(3500);   // ✅ 优雅地让出 CPU！
}
```

#### 🔍 深度原理对比：

1. `HAL_Delay(3500)` 的本质（忙等待 / Busy-Waiting）

   ：

   - 它的底层源码是一个 `while((HAL_GetTick() - tickstart) < wait)` 的死循环。
   - 如果在 `main()` 函数启动调度器之前调用 `ESP8266_Init()`（包含 7 条 AT 指令，每条 `HAL_Delay(3500)`），系统在开机阶段就会死死卡住 7×3.5s=24.5秒7×3.5s=24.5秒！在这近半分钟内，串口屏无法更新、RTC 时间走字卡死，用户体验就是“开机极其缓慢，像死机了一样”。
   - 如果在 FreeRTOS 任务中调用 `HAL_Delay()`，由于任务一直处于 `Running` 状态死循环读变量，所有比它优先级低的任务将**永远得不到执行（任务饿死 Starvation）**。

2. `osDelay(3500)` 的本质（任务级非忙等待阻塞）

   ：

   - `osDelay()` 是 CMSIS-RTOS 对 FreeRTOS 原生函数 `vTaskDelay()` 的封装。
   - 当任务调用 `osDelay(3500)` 时，FreeRTOS 内核会将当前任务从**就绪列表（Ready List）**中移除，挂载到**延时列表（Delayed Task List）**中，并将任务状态标记为 `Blocked`。
   - 随后立即触发一次上下文切换（Context Switch）。在等待 ESP8266 联网的这 24.5 秒内，CPU 被完全释放出来去执行 **RTC 时钟显示任务** 和 **UI 交互任务**。在用户看来，设备一上电屏幕立刻开始正常走时，后台静默完成 Wi-Fi 连接后天气数据自动弹出，从而彻底解决了“开机启动慢/假死”的问题！

------

### 4.5 中断管理与延迟处理：二值信号量（Binary Semaphore）

在 `main.c` 的串口接收完成中断回调函数 `HAL_UART_RxCpltCallback` 中，展示了 FreeRTOS 中经典的 **“中断延迟处理（Deferred Interrupt Processing）”** 架构：

C



```
extern osSemaphoreId myBinarySem01Handle; // 二值信号量句柄

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    /* ... 省略 UART2 代码 ... */
    else if(huart == &huart3) // 淘晶驰串口屏发来数据
    {	
        RxBuffer[Uart2_Rx_Cnt++] = aRxBuffer;
        // 判断是否接收到帧尾 \r\n (0x0D 0x0A)
        if((RxBuffer[Uart2_Rx_Cnt-1] == 0x0A) && (RxBuffer[Uart2_Rx_Cnt-2] == 0x0D)) 
        {
            HAL_UART_Transmit(&huart1, (uint8_t *)&RxBuffer, Uart2_Rx_Cnt, 0xFFFF);
            while(HAL_UART_GetState(&huart1) == HAL_UART_STATE_BUSY_TX);
            printf("您所收到的数据是：");
            
            // 🔥 核心操作：在中断中释放二值信号量，唤醒后台处理任务！
            osSemaphoreRelease(myBinarySem01Handle); 
        }
        HAL_UART_Receive_IT(&huart3, (uint8_t *)&aRxBuffer, 1); // 重新开启下一次中断
    }
}
```

#### 1. 什么是二值信号量（Binary Semaphore）？

二值信号量可以被看作是一个**长度为 1 的特殊队列**，它的计数值只有 `0`（空/无信号）和 `1`（满/有信号）两种状态。它不用于传输具体的海量数据，而是专门用于**任务与任务之间**，或者**中断（ISR）与任务之间**的**同步（Synchronization）**。

#### 2. 为什么要用信号量，而不是直接在中断里处理业务逻辑？

实时系统设计的第一铁律：**中断服务函数（ISR）必须尽可能短小精悍（快进快出）！**
如果我们在串口中断里直接去做复杂的字符串比对、修改 RTC 时间或重发网络请求，会导致中断执行时间过长，屏蔽掉其他同级或低优先级中断（导致 UART2 接收 ESP8266 的 JSON 数据丢包！）。

因此，标准的 RTOS 生产者-消费者模型如下：

text



```
 [硬件中断层: UART3_IRQHandler]               [RTOS 任务层: HMI_Process_Task]
            |                                               |
  1. 逐字节接收存入 RxBuffer                         (平时处于 Blocked 阻塞态，
  2. 检测到 "\r\n" 帧尾                              调用 osSemaphoreWait 死等，
  3. 调用 osSemaphoreRelease()                       不消耗 1% 的 CPU 算力)
     将信号量令牌置为 1  =====================> 4. 信号量变为 1，任务瞬间被唤醒！
  4. 立即退出中断，恢复现场                       5. 信号量自动清 0，进入 Running 态
                                                6. 安全地解析 RxBuffer 并执行业务
```

#### ⚠️ 进阶知识点：FreeRTOS 的中断安全 API 与优先级阈值

- **FromISR 后缀机制**：在原生 FreeRTOS 中，普通任务调用的 API（如 `xSemaphoreGive()`）与中断里调用的 API（如 `xSemaphoreGiveFromISR()`）是严格区分的！因为中断中没有任务上下文，不能发生阻塞，且唤醒高优先级任务时需要手动标记 `pxHigherPriorityTaskWoken` 并在退出中断前调用 `portYIELD_FROM_ISR()`。您代码中使用的 `osSemaphoreRelease()` 是 ST 封装的 CMSIS-RTOS v1 接口，它内部通过读取 Cortex-M4 的 `IPSR` 寄存器自动判断当前是否处于中断中，从而自动调用 `xSemaphoreGiveFromISR()`。
- **中断优先级红线（`configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`）**：在 STM32CubeMX 中，这一项默认配置为 `5`。**任何调用了 FreeRTOS API（如 `osSemaphoreRelease`）的硬件中断（如本项目中的 `USART3_IRQn`），其抢占优先级数值必须大于或等于 5（即逻辑优先级低于或等于 5，如设为 5~15）！** 如果设为 0~4，当 FreeRTOS 进入临界区关闭中断时将无法屏蔽该串口中断，直接破坏内核链表数据结构，触发 `configASSERT` 死机或 `HardFault`！

------

### 4.6 FreeRTOS 内存管理与 cJSON 冲突的终极解决方案

您在原说明中非常敏锐地指出了本项目最大的坑点：

> *“FreeRTOS和cJSON有冲突，需要将cJSON申请内存空间的函数替换成FreeRTOS申请内存的函数，每次解码后，**一定要释放内存**，否则解码会卡死，而且需要把Heap_size设置稍微大一点，推荐设置为4096”*

下面我们从底层编译原理和 RTOS 内核角度，彻底讲透**为什么会冲突？为什么会卡死？以及如何科学配置？**

#### 1. 为什么标准库 `malloc/free` 与 FreeRTOS 冲突？

cJSON 在解析 JSON 字符串（调用 `cJSON_Parse()`）时，由于不知道 JSON 树有多深、有多少个节点，它会在底层极其频繁地调用动态内存分配函数（默认是 C 标准库 `<stdlib.h>` 中的 `malloc` 和 `free`）来构建双向链表树。
在 FreeRTOS 中直接使用标准 `malloc/free` 有三大致命缺陷：

1. **非线程安全（Not Thread-Safe）**：标准库的 `malloc` 操作全局堆链表时没有加锁保护。如果两个任务同时分配内存，或者分配途中发生任务切换，堆链表指针就会错乱崩溃。
2. **默认系统堆（System Heap）极小**：在 STM32 的启动汇编文件 `startup_stm32f407xx.s` 中，分配给 C 标准库 `malloc` 的 `Heap_Size` 默认仅仅只有 **`0x00000200`（即 512 字节）**！而心知天气返回的一串 JSON 解析树结构动辄需要 1KB~2KB 的节点空间，一调 `cJSON_Parse()` 瞬间就会返回 `NULL`，接着代码对空指针解引用（`Array_obj1->valuestring`）就会直接触发 **HardFault_Handler 硬件错误死机**！
3. **内存碎片化（Memory Fragmentation）**与执行时间不确定性。

#### 2. FreeRTOS 的五种内存管理算法（`heap_1` ~ `heap_5`）

FreeRTOS 抛弃了 C 标准库的堆，自己在 RAM 中开辟了一个大数组 `uint8_t ucHeap[configTOTAL_HEAP_SIZE]`（即您在 CubeMX 中设置的 `TOTAL_HEAP_SIZE`，推荐设为 `4096` 或 `8192` 字节以上），并提供了专门的线程安全分配函数 `pvPortMalloc()` 和释放函数 `vPortFree()`。

|          内存方案          | 是否允许释放 (`vPortFree`) |       碎片合并算法       | 适用场景与特点                                               |
| :------------------------: | :------------------------: | :----------------------: | :----------------------------------------------------------- |
|         **heap_1**         |          ❌ 不支持          |            无            | 最简单，只分配不释放，适合从不删除任务/队列的安全关键系统    |
|         **heap_2**         |           ✅ 支持           |    ❌ 不合并相邻空闲块    | 采用最佳匹配算法，若频繁分配不同大小的内存（如 cJSON），会迅速产生大量内存碎片导致耗尽 |
|         **heap_3**         |           ✅ 支持           |     依赖编译器标准库     | 只是简单地给标准 `malloc/free` 套了一层挂起调度器（`vTaskSuspendAll`）的线程安全外壳，仍受限于启动文件 `.s` 的堆大小 |
| **heap_4** *(🔥本项目默认)* |           ✅ 支持           | ✅ **自动合并相邻空闲块** | **首次适应算法 + 空闲块合并机制**。能有效防止 cJSON 反复申请/释放不同大小节点带来的内存碎片问题！ |
|         **heap_5**         |           ✅ 支持           |   ✅ 自动合并相邻空闲块   | 在 `heap_4` 基础上支持跨多个不连续物理内存区域（如内部 SRAM + 外部 CCMRAM/SDRAM）构建堆 |

#### 3. 完美解决冲突的代码实现步骤

**第一步：重定向 cJSON 的内存钩子函数（Hook）**
在系统初始化阶段（或修改 `cJSON.c` 源码中的 `cJSON_InitHooks`），将内存分配接口替换为 FreeRTOS 的 `pvPortMalloc` 和 `vPortFree`：

C



```
#include "FreeRTOS.h"
#include "cJSON.h"

void cJSON_FreeRTOS_Init(void)
{
    cJSON_Hooks hooks;
    hooks.malloc_fn = pvPortMalloc; // 绑定 FreeRTOS 线程安全内存申请函数
    hooks.free_fn   = vPortFree;    // 绑定 FreeRTOS 线程安全内存释放函数
    cJSON_InitHooks(&hooks);
}
```

**第二步：严格遵循“申请-判空-使用-释放根节点”闭环（防止内存泄漏 Memory Leak）**
cJSON 构建的是一棵多叉树结构，每一次调用 `cJSON_Parse()` 都会在 FreeRTOS 的 `ucHeap` 中划走近 1KB 的空间。如果只申请不释放，即使 `Heap_Size` 设为 `4096`，最多解析 3~4 次天气数据，内存池就会彻底枯竭（`xPortGetFreeHeapSize()` 降为 0），系统随即卡死！

**注意：只需对最顶层的根节点 `json` 调用一次 `cJSON_Delete(json)`，它内部就会通过递归自动释放下面挂载的所有子节点（`Array_obj1`, `Array_obj2` 等）！切勿对子节点重复调用 Delete！**

C



```
/* 标准安全的 FreeRTOS + cJSON 解析范式 */
json = cJSON_Parse((const char *)my_re_buf); // 1. 申请内存并构建解析树
if (json != NULL)                            // 2. 必须判空！防止网络乱码导致解析失败进 HardFault
{
    Array_obj1 = cJSON_GetObjectItem(json, "results");
    Array      = cJSON_GetArrayItem(Array_obj1, 0);
    Array_obj2 = cJSON_GetObjectItem(Array, "now");
    
    Array_obj3 = cJSON_GetObjectItem(Array_obj2, "text");        // 天气现象
    Array_obj4 = cJSON_GetObjectItem(Array_obj2, "temperature"); // 温度数值
    
    // 拷贝数据到全局缓冲或直接发送给串口屏...
    
    cJSON_Delete(json); // 3. 🔥 核心铁律：递归释放整棵 JSON 树占用的 FreeRTOS 堆内存！
    json = NULL;        // 4. 预防野指针
}
```

------

### 4.7 其他必备 FreeRTOS 核心组件（队列、互斥量、事件组）

为了使本项目的 FreeRTOS 知识体系更加完整，以下补充另外三个在气象站升级扩展中至关重要的内核机制：

1. 消息队列（Message Queue）

   ：

   - **概念**：任务间传输真实数据的 FIFO（先进先出）环形缓冲区，支持**值传递（拷贝数据）\**或\**指针传递**。
   - **在本项目中的应用场景**：目前串口 2 接收数据直接堆在全局变量 `my_re_buf[2000]` 中。更规范的 RTOS 做法是创建一个天气数据结构体 `typedef struct { char weather[16]; char temp[8]; } Weather_t;`，解析任务解析完后通过 `osMessagePut()` 发送到队列，屏幕刷新任务通过 `osMessageGet()` 从队列读取并刷新淘晶驰屏幕，实现**网络层与 UI 层的彻底解耦**。

2. 互斥量（Mutex）与优先级翻转（Priority Inversion）

   ：

   - **概念**：用于保护**共享硬件资源（临界资源）**的特殊二值信号量，具备**优先级继承（Priority Inheritance）**机制。
   - **在本项目中的应用场景**：`USART3`（串口屏通信接口）是一个典型的共享资源！*RTC 时钟任务* 每秒钟都要通过 `USART3` 发送 `page0.t0.txt="xx:xx:xx"`，而 *天气更新任务* 也要通过 `USART3` 发送温度和天气图标更新指令。如果两个任务同时调用 `HMISends()`，没有互斥锁保护的话，串口线上发出的指令就会互相穿插拼接（例如变成 `page0.t0.page0.t1.txt="26"txt="12:00"`），导致串口屏指令解析错误！因此，在调用 `HMISends()` 前后应当加上互斥锁：`osMutexWait(uart3MutexHandle, osWaitForever)` 和 `osMutexRelease(uart3MutexHandle)`。

3. 任务栈溢出检测（Stack Overflow Detection）

   ：

   - 每个任务都有自己独立的栈（在 `MX_FREERTOS_Init` 中设置的 `Stack Size`，单位是 **Word 即 4字节**，例如填 `128` 代表 128×4=512128×4=512 字节）。
   - 如果在任务里定义了大型局部数组（如 `char buf[1024]`）或频繁调用 `printf`、`cJSON` 递归函数，极易撑爆任务栈！建议在调试阶段使用 `uxTaskGetStackHighWaterMark(NULL)` 查看任务的**栈历史剩余最小值（高水位线）**，据此合理调大包含 cJSON 解析任务的栈大小（推荐该任务 `Stack Size` ≥512≥512 Words）。

------

## 🔍 5. 核心代码解析与健壮性优化建议

通过对您提供的 `main.c` 源码进行静态代码审查，这里整理了核心模块的精妙之处以及未来可进一步提升系统长期运行稳定性的优化建议：

### 5.1 串口 3 中断中的阻塞隐患优化

在当前的 `HAL_UART_RxCpltCallback` 中：

C



```
if((RxBuffer[Uart2_Rx_Cnt-1] == 0x0A)&&(RxBuffer[Uart2_Rx_Cnt-2] == 0x0D))
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&RxBuffer, Uart2_Rx_Cnt,0xFFFF); // 轮询发送
    while(HAL_UART_GetState(&huart1) == HAL_UART_STATE_BUSY_TX);
    printf("您所收到的数据是：");
    osSemaphoreRelease(myBinarySem01Handle);
}
```

- **优化建议**：在中断回调函数里调用 `HAL_UART_Transmit` 和 `printf` 属于耗时轮询操作。建议在中断里**只保留** `osSemaphoreRelease(myBinarySem01Handle);`，将 `HAL_UART_Transmit`、打印提示语以及将计数器 `Uart2_Rx_Cnt = 0;` 清零的操作全部移到等待该信号量的 FreeRTOS 任务中去执行，这才是标准的“中断下半部（Bottom Half）”处理方式。

### 5.2 串口 2 接收缓冲区的越界保护与空闲中断（IDLE + DMA）升级

当前接收 ESP8266 数据的逻辑为：

C



```
if(huart == &huart2)
{
    my_re_buf[pt++] = aRxBuffer2;
    HAL_UART_Receive_IT(&huart2, (uint8_t *)&aRxBuffer2, 1);
}
```

- **边界保护**：建议增加防溢出判断 `if(pt >= sizeof(my_re_buf) - 1) pt = 0;`，并在每次重新发起 HTTP GET 请求前，使用 `memset(my_re_buf, 0, sizeof(my_re_buf)); pt = 0;` 清空缓冲区和游标。
- **高阶架构演进（UART IDLE + DMA）**：ESP8266 返回一帧 JSON 数据大约有 300~500 字节，单字节中断方式需要频繁进入 500 次中断。进阶优化可开启 STM32F407 的 **串口空闲中断（`__HAL_UART_ENABLE_IT(&huart2, UART_IT_IDLE)`）配合 DMA 接收（`HAL_UART_Receive_DMA`）**。当一整包 JSON 数据流接收完毕总线空闲时，仅触发 **1 次** 中断，在 IDLE 中断中释放一个二值信号量通知 `cJSON` 任务直接解析，CPU 占用率将骤降至接近 0%！

------

## 📸 6. 实现效果

系统上电后，淘晶驰串口屏初始化显示时间框架，后台 FreeRTOS 调度器启动，RTC 时钟任务实时走字，同时网络任务驱动 ESP8266 自动完成 Wi-Fi 握手与心知天气服务器连接，实时拉取并展示当前城市的天气状态与气温数据：

![实现效果](https://img-blog.csdnimg.cn/20200625171702401.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25hbWVfbG9uZ21pbmc=,size_16,color_FFFFFF,t_70)

------

## 📅 7. 更新日志

|    更新时间    |  版本号  | 更新内容与技术细节                                           |
| :------------: | :------: | :----------------------------------------------------------- |
| **2020.06.27** | `v1.1.0` | **修复了开机启动慢的问题，大幅加快了开机响应速度**：<br>将 ESP8266 初始化函数中的阻塞式死等延时 `HAL_Delay(3500)` 替换为 FreeRTOS 任务级非阻塞延时 `osDelay(3500)`（即 `FreeROTS_ESP8266_Init`），在等待 AT 指令响应期间释放 CPU 控制权，使 UI 界面与 RTC 时钟任务得以在开机瞬间并发运行。 |
| **2020.06.26** | `v1.0.0` | **项目首个正式版本发布**：<br>完成 STM32F407 + FreeRTOS + ESP8266 + 淘晶驰串口屏的联调；解决 FreeRTOS `heap_4` 与 `cJSON` 动态内存分配冲突问题；具备心知天气与气温实时检测系统，并附加基于片上 RTC 的实时时间系统。 |

------

## 📄 开源许可 (License)

本项目底层 HAL 驱动及 CMSIS-RTOS 封装库遵循 STMicroelectronics **Ultimate Liberty license (SLA0044)** 协议。应用层代码仅供物联网与嵌入式实时操作系统（FreeRTOS）学习与技术交流使用。

> ⚠️ **安全提示**：源码中的 Wi-Fi 账号密码（`AT+CWJAP`）及心知天气 API 私钥（`key=...`）在部署至您自己的硬件前，请替换为您个人的热点信息与心知天气开发者账号密钥。
