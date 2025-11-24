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

## 6.4 驱动注册流程详解

让我们深入了解驱动是如何注册到系统中的：

**步骤 1：驱动初始化**
```c
int main(int argc, char** argv) {
    // 初始化设备结构
    vdevice_t dev;
    memset(&dev, 0, sizeof(vdevice_t));
    strcpy(dev.name, "timer");
    
    // 设置设备控制回调函数
    dev.dev_cntl = timer_dcntl;
    
    // 向 VFS 注册设备
    device_run(&dev, "/dev/timer", FS_TYPE_CHAR, 0666);
    return 0;
}
```

**步骤 2：VFS 挂载**
`device_run()` 内部会：
1.  通过 IPC 连接到 VFS 守护进程。
2.  发送注册请求，包含设备路径和类型。
3.  VFS 在虚拟文件树中创建设备节点。
4.  进入事件循环，等待 VFS 分发请求。

**步骤 3：设备控制回调**
```c
int timer_dcntl(int from_pid, int cmd, proto_t* in, proto_t* out, void* p) {
    switch(cmd) {
        case DEV_CMD_READ:
            // 读取当前时间
            uint64_t time = get_system_time();
            proto_add_uint64(out, time);
            return 0;
        case DEV_CMD_WRITE:
            // 设置定时器
            uint64_t timeout = proto_read_uint64(in);
            set_timer(timeout);
            return 0;
        default:
            return -1;
    }
}
```

## 6.5 常见驱动实例

### SD 卡驱动 (`sdd`)
负责读写 SD 卡存储。它实现了块设备接口，支持：
*   按扇区读写（通常是 512 字节）
*   DMA 传输（Direct Memory Access），提高效率
*   多分区支持

### USB 驱动 (`usbd`)
EwokOS 支持 USB 设备！USB 驱动实现了：
*   USB 主机控制器接口（HCI）
*   设备枚举和识别
*   USB 键盘、鼠标支持
*   USB 存储设备支持

### Framebuffer 驱动 (`fbd`)
图形显示的基础，提供：
*   像素缓冲区映射
*   图形模式切换
*   双缓冲（避免闪烁）
*   支持多种分辨率（如 1024x768）

## 6.6 VFS 挂载机制

VFS 维护了一棵虚拟文件树。挂载点可以是：

**根文件系统**：
```
mount("/", "ext2", "/dev/mmcblk0p1");  // 挂载 SD 卡第一分区为根
```

**设备文件系统**：
```
/dev/
├── timer    -> timerd
├── tty0     -> uartd
├── fb0      -> fbd
├── mmcblk0  -> sdd
└── ...
```

**进程信息文件系统**：
```
/proc/
├── 1/       -> Core 进程信息
├── 2/       -> VFS 进程信息
└── ...
```

**实际挂载过程**：
1.  文件系统驱动（如 `ext2d`）向 VFS 注册。
2.  VFS 调用驱动的 `mount()` 方法。
3.  驱动读取分区超级块，建立目录索引。
4.  VFS 将挂载点加入全局挂载表。

虽然中间隔了个 VFS，看起来有点繁琐，但它统一了接口。用户程序不需要知道对面是硬盘还是串口，只需要会 `open/read/write` 三板斧就行了。
