# 第二章：EwokOS - 小内核，大梦想

## 2.1 架构总览：美味的三明治

如果把 EwokOS 比作一个三明治，它的结构是这样的：

### 1. 底层面包：微内核 (The Kernel)
这是三明治的最底层，也是最硬的一层。它直接铺在盘子（硬件）上。
*   **位置**：`kernel/kernel`
*   **职责**：
    *   **调度**：决定谁能咬一口 CPU。
    *   **IPC**：负责传递层与层之间的酱料（消息）。
    *   **内存管理**：划分地盘，防止大家打架。
*   **特点**：代码量极少，只有几千行。它不认识文件，不认识键盘，甚至不知道怎么在屏幕上画画。它只负责最基本的生存需求。

### 2. 中间夹心：核心服务 (Core Services)
这层是三明治的精华，提供了操作系统的基础口感。
*   **位置**：`system/`
*   **成员**：
    *   **Core**：内核的大管家，负责进程的注册、生杀大权。
    *   **VFS (Virtual File System)**：虚拟文件系统。它是一个服务进程，负责管理所有的文件和设备。在 EwokOS 里，**一切皆文件**，而 VFS 就是这些文件的总管理员。

### 3. 上层配菜：驱动与应用 (Drivers & Apps)
这层是最丰富多彩的，想加什么就加什么。
*   **位置**：`system/basic/drivers`, `system/basic/bin`
*   **成员**：
    *   **驱动程序**：`timerd` (时钟), `uartd` (串口), `fbd` (显卡)。注意！它们是**普通进程**，和计算器、记事本没有本质区别，只是它们有权访问特定的硬件。
    *   **应用程序**：Shell, Finder, 甚至贪吃蛇游戏。

## 2.2 核心组件图解

让我们画一张图来看看它们是怎么互动的：

```mermaid
graph TD
    subgraph User Space [用户空间 (User Space)]
        App[用户应用 (Shell, Game)]
        Driver[驱动进程 (Timer, UART, FB)]
        VFS[VFS 服务进程]
        Core[Core 服务进程]
    end

    subgraph Kernel Space [内核空间 (Kernel Space)]
        Microkernel[微内核 (Scheduler, IPC, MMU)]
    end

    subgraph Hardware [硬件 (Hardware)]
        CPU
        RAM
        IO[外设 (Timer, UART, Screen)]
    end

    App -- 1. 读文件 --> VFS
    VFS -- 2. 转发请求 --> Driver
    Driver -- 3. 读写寄存器 --> Microkernel
    Microkernel -- 4. 操作硬件 --> IO
    IO -- 5. 中断 --> Microkernel
    Microkernel -- 6. 通知 --> Driver
    Driver -- 7. 数据返回 --> VFS
    VFS -- 8. 结果返回 --> App
```

## 2.3 为什么这样设计？

你可能会问，为什么要把驱动放在用户空间？让它们直接在内核里跑不是更快吗？

这就好比：**为什么不让修水管的工人住在你家里？**

*   **安全性**：如果修水管的工人（驱动）住在你家（内核），他万一发疯了（Bug），可能会把你的房子拆了（系统崩溃）。如果他只是上门服务（用户进程），他发疯了，你把他赶出去（Kill 进程）就行了，房子还在。
*   **灵活性**：你想换个修水管的？直接换人就行，不用搬家。在 EwokOS 里，你可以在系统运行时加载、卸载驱动，就像打开关闭 App 一样简单。

当然，代价就是——**沟通成本**。每次修水管都要打电话预约、开门、关门（IPC 通信），确实比住在家里随叫随到要慢一点。但为了安全和灵活，这个代价是值得的。

下一章，我们将深入内核的最深处，看看它是如何像提线木偶一样操控进程的。
