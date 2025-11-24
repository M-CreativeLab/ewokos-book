# 第六章：打工魂 - 驱动与 VFS

## 6.1 驱动：我也是个普通人

在宏内核里，驱动是拥有至高无上权力的“御前侍卫”。
在微内核里，驱动只是一个普通的“打工仔”。

### 驱动就是进程
EwokOS 的驱动程序（比如 `timerd`, `uartd`）本质上就是**用户进程**。
它们有自己的 `main` 函数，有自己的 PID，甚至可以被 `kill` 掉。

唯一的区别是，它们通过系统调用申请了一些特权：
*   **I/O 端口访问权**：允许读写特定的硬件端口。
*   **MMIO 映射权**：允许把硬件的物理内存映射到自己的虚拟空间。
*   **中断处理权**：允许接收硬件发出的中断信号。

## 6.2 实例分析：`timerd` 的一天

让我们看看时钟驱动 `timerd` (`system/basic/drivers/timerd/timerd.c`) 是怎么工作的。

1.  **上班打卡 (启动)**：
    ```c
    int main(int argc, char** argv) {
        // ... 初始化 ...
        vdevice_t dev;
        strcpy(dev.name, "timer");
        dev.dev_cntl = timer_dcntl; // 设置“业务受理窗口”
        
        // 向 VFS 注册：我负责 /dev/timer 这个地盘
        device_run(&dev, "/dev/timer", FS_TYPE_CHAR, 0666);
        return 0;
    }
    ```

2.  **等待业务 (循环)**：
    `device_run` 内部是一个无限循环，不断调用 `ipc_fetch` 等待 VFS 派活。

3.  **处理业务 (中断)**：
    当硬件产生时钟中断时，内核会通知 `timerd`。
    `timerd` 醒来，更新系统时间，检查有没有闹钟到期了。

## 6.3 VFS：全能前台

**VFS (Virtual File System)** 是整个系统的“前台接待”。
不管你是要读文件、写串口、还是画图，你都得先找 VFS。

EwokOS 遵循 Unix 的哲学：**一切皆文件**。
*   硬盘文件是文件：`/home/test.txt`
*   串口是文件：`/dev/tty0`
*   显卡是文件：`/dev/fb0`
*   甚至进程信息也是文件：`/proc/123`

### 一次 `open` 的旅程

当你调用 `open("/dev/timer", ...)` 时：

1.  **用户进程**：呼叫 VFS，“我要打开 `/dev/timer`”。
2.  **VFS**：查通讯录（挂载表），发现 `/dev/timer` 归 `timerd` 管。
3.  **VFS**：呼叫 `timerd`，“有人要打开你”。
4.  **timerd**：确认没问题，返回“同意”。
5.  **VFS**：给用户进程发一张“通行证”（文件描述符 fd），上面写着：“以后找 `timerd`，直接报这个号”。

### 一次 `write` 的旅程

当你调用 `write(fd, "hello", 5)` 时：

1.  **用户进程**：拿着通行证 (fd) 呼叫 VFS。
2.  **VFS**：看一眼通行证，哦，是找 `timerd` 的。
3.  **VFS**：把数据转发给 `timerd`。
4.  **timerd**：收到数据，执行相应的硬件操作。

虽然中间隔了个 VFS，看起来有点繁琐，但它统一了接口。用户程序不需要知道对面是硬盘还是串口，只需要会 `open/read/write` 三板斧就行了。

下一章，我们将把手弄脏，亲自编译和运行 EwokOS。
