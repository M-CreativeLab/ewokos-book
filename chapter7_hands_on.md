# 第七章：把手弄脏 - 编译与运行

纸上得来终觉浅，绝知此事要躬行。

前面六章，我们从理论层面理解了 EwokOS 的方方面面：架构、进程、IPC、内存、驱动。现在，让我们从读者变成实践者，亲手把 EwokOS 运行起来。

就像学习游泳，看再多视频也比不上跳进水里试一次。操作系统也是一样——只有当你看到内核从零启动，看到 Shell 提示符出现，你才会真正理解这些抽象概念是如何变成现实的。

准备好了吗？让我们开始这段实践之旅。

## 7.1 准备工作

你需要一台 Linux 或 macOS 电脑。Windows 用户建议使用 WSL2。

### 安装工具链
EwokOS 是为 ARM 架构编写的，所以我们需要一个**交叉编译器**。
我们需要 `arm-none-eabi-gcc`。

**macOS (Homebrew):**
```bash
brew tap PX4/homebrew-px4
brew install gcc-arm-none-eabi-49
brew install qemu
```

**Ubuntu/Debian:**
```bash
sudo apt-get install gcc-arm-none-eabi qemu-system-arm
```

## 7.2 编译内核

EwokOS 支持多种硬件平台，最方便的是在 QEMU 模拟器上运行 Raspberry Pi (树莓派)。

1.  **下载源码**：
    ```bash
    git clone https://github.com/MisaZhu/ewokos.git
    cd ewokos
    ```

2.  **编译内核**：
    进入树莓派构建目录（以 Raspberry Pi 2 为例）：
    ```bash
    cd kernel/build/raspi2/pix
    make
    ```
    如果一切顺利，你会看到 `kernel7.img` 生成了。

3.  **编译文件系统**：
    EwokOS 需要一个根文件系统 (RootFS) 才能启动。
    ```bash
    cd ../../../system
    make
    ```
    这会编译所有的驱动和应用程序，并打包成 `root.ext2`。

## 7.3 运行！

回到构建目录，启动 QEMU：

```bash
cd kernel/build/raspi2/pix
make run
```

你会看到一个黑色的窗口弹出，EwokOS 的启动日志开始刷屏。
几秒钟后，你会看到登录提示：

```
login: root
password: <直接回车>
```

恭喜你！你已经成功进入了 EwokOS 的世界。

## 7.4 接下来去哪儿？

现在你已经跑起来了，可以试着修改一下代码：
1.  **改改欢迎语**：去 `system/basic/bin/login/login.c`，把 "Welcome to EwokOS" 改成你的名字。
2.  **写个小程序**：在 `system/basic/bin` 下新建一个目录，写个 `hello.c`，在 `Makefile` 里加上它，编译运行看看。
3.  **读读内核**：从 `kernel/kernel/src/kernel.c` 开始，跟着代码走一遍启动流程。

微内核的世界博大精深，EwokOS 只是为你打开了一扇窗。
希望这本小册子能让你对操作系统有更直观的认识。
现在，去探索代码的海洋吧！

## 7.5 真实硬件部署

在模拟器上玩够了？来点真的！

### 7.5.1 制作 SD 卡

EwokOS 支持直接从 SD 卡启动。你需要：

1.  **准备 SD 卡**：至少 1GB，建议 4GB 以上。
2.  **分区**：
    ```bash
    # 使用 fdisk 创建分区
    # 分区 1: FAT32, 64MB (启动分区)
    # 分区 2: EXT2, 剩余空间 (根文件系统)
    ```

3.  **复制文件**（以 Raspberry Pi 2 为例）：
    ```bash
    # 挂载启动分区
    mount /dev/sdX1 /mnt/boot
    
    # 复制固件和内核
    cp bootcode.bin start.elf config.txt /mnt/boot/
    cp kernel/build/raspi2/pix/kernel7.img /mnt/boot/
    
    # 挂载根文件系统分区
    mount /dev/sdX2 /mnt/root
    
    # 写入根文件系统
    cd system/build
    sudo dd if=root.ext2 of=/dev/sdX2 bs=1M
    ```

4.  **插入 Raspberry Pi，上电启动！**

### 7.5.2 串口调试

如果 EwokOS 启动失败，你需要串口来看内核日志：

**硬件连接**：
*   USB 转 TTL 线
*   TX -> GPIO14 (Pin 8)
*   RX -> GPIO15 (Pin 10)
*   GND -> GND (Pin 6)

**软件配置**：
```bash
# Linux/Mac
sudo minicom -D /dev/ttyUSB0 -b 115200

# 或使用 screen
screen /dev/ttyUSB0 115200
```

你会看到内核启动日志：
```
[kernel] EwokOS v0.x starting...
[kernel] MMU enabled
[kernel] SMP: 4 cores detected
[kernel] Loading init...
[init] Mounting root filesystem...
[core] Starting VFS...
...
```

### 7.5.3 支持的硬件平台

EwokOS 已移植到多种硬件：

**树莓派系列**：
*   Raspberry Pi 1 (ARMv6, 单核)
*   Raspberry Pi 2 (ARMv7, 4 核)
*   Raspberry Pi 3 (ARMv8, 64 位, 4 核)
*   Raspberry Pi 4 (ARMv8, 64 位, 4 核)
*   Raspberry Pi 5（开发中）

**其他开发板**：
*   ARM Versatile PB (QEMU)
*   Orange Pi
*   Miyoo 掌机
*   Rockchip RK3128, RK3506

每个平台的编译路径不同：
```bash
# Raspberry Pi 2
cd kernel/build/raspi2/pix

# Raspberry Pi 3 (64 位)
cd kernel/build/raspi3/pix64

# Versatile PB
cd kernel/build/versatilepb/pix
```

## 7.6 调试技巧

### 使用 GDB 调试内核

EwokOS 支持 GDB 远程调试：

1.  **启动调试服务器**：
    ```bash
    cd kernel/build/raspi2/pix
    make debug
    ```
    QEMU 会监听 1234 端口。

2.  **连接 GDB**：
    ```bash
    # 另开一个终端
    make gdb
    ```
    或手动连接：
    ```bash
    arm-none-eabi-gdb kernel.elf
    (gdb) target remote localhost:1234
    (gdb) break kernel_main
    (gdb) continue
    ```

### 常用 GDB 命令

```
(gdb) info registers       # 查看寄存器
(gdb) x/16x 0x8000        # 查看内存
(gdb) bt                   # 查看调用栈
(gdb) list                 # 查看源代码
(gdb) step                 # 单步执行
```

## 7.7 性能分析

### 查看进程信息

```bash
# 列出所有进程
ps

# 查看进程详细信息
cat /proc/1/info

# 查看 CPU 使用率
top
```

### 内存使用

```bash
# 查看内存统计
free

# 查看进程内存映射
cat /proc/<pid>/maps
```

微内核的世界博大精深，EwokOS 只是为你打开了一扇窗。
希望这本小册子能让你对操作系统有更直观的认识。
现在，去探索代码的海洋吧！

## 7.8 扩展阅读

想深入了解 EwokOS？这里有一些资源：

*   **官方仓库**：https://github.com/MisaZhu/ewokos
*   **文档目录**：`docs/` 下有各平台的详细部署文档
*   **源码导读**：
    *   从 `kernel/kernel/src/kernel.c` 开始，这是内核入口
    *   看 `system/basic/init/init.c`，了解用户空间启动
    *   读 `system/basic/sbin/vfsd/vfsd.c`，理解 VFS 实现
    *   研究 `system/basic/drivers/` 下的驱动代码

**学习建议**：
1.  先在 QEMU 上跑起来，熟悉系统
2.  试着修改内核，添加自己的系统调用
3.  写一个简单的驱动（比如 LED 闪烁）
4.  移植到真实硬件，体验完整流程

操作系统不再是黑盒，你已经掌握了打开它的钥匙！

## 7.9 你的操作系统之旅才刚刚开始

恭喜你！你已经完成了这本 EwokOS 入门指南。从理论到实践，从架构到代码，你已经掌握了微内核操作系统的核心知识。

但这只是开始。操作系统是一个博大精深的领域，还有无数有趣的话题等待探索：

### 继续学习的方向

**1. 深入 EwokOS 源码**
*   研究调度器的实现细节，理解抢占式调度
*   跟踪一次系统调用的完整路径
*   分析内存管理的页表数据结构
*   阅读 VFS 的源码，理解文件系统如何工作

**2. 编写自己的驱动**
*   为 LED 编写一个简单的驱动
*   实现一个虚拟的字符设备
*   移植一个新的文件系统（如 FAT32）
*   为新的硬件平台适配 EwokOS

**3. 性能优化**
*   分析系统的性能瓶颈
*   优化 IPC 的效率
*   实现更先进的调度算法（如 CFS）
*   改进内存分配器

**4. 对比其他操作系统**
*   阅读 Linux 内核源码，对比宏内核的设计
*   研究 Minix 3，Andrew Tanenbaum 的现代微内核
*   了解 Fuchsia，Google 的下一代操作系统
*   学习 seL4，世界上第一个形式化验证的微内核

### 一些建议

**从简单开始**：不要试图一次理解所有东西。选择一个感兴趣的模块（如调度器），深入研究它。

**动手实践**：修改代码，看看会发生什么。故意引入 bug，然后用 GDB 调试。失败是最好的老师。

**阅读经典**：
*   《Operating System Concepts》（恐龙书）- 理论基础
*   《Operating Systems: Design and Implementation》（Tanenbaum）- Minix 的设计
*   《Linux Kernel Development》- Linux 内核实践

**加入社区**：
*   EwokOS GitHub 仓库：https://github.com/MisaZhu/ewokos
*   提交 Issue，贡献代码
*   与其他爱好者交流

### 最后的话

操作系统不再是一个黑盒子。你已经看到了它的内部运作，理解了它的设计哲学。无论你将来从事什么方向的开发——Web、移动、嵌入式、游戏——这些知识都会让你成为更优秀的工程师。

记住，每一个伟大的操作系统都始于一个简单的想法：
*   Unix 始于"简单即美"
*   Linux 始于一个学生的业余项目
*   EwokOS 始于对微内核的热爱

也许有一天，你会创造出属于自己的操作系统。那时，请记得这段学习之旅。

**操作系统的世界永远欢迎你。继续探索，继续创造！**
