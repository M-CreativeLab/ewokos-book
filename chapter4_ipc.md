# 第四章：喂，听得见吗？ - IPC

## 4.1 为什么需要 IPC？

在微内核的世界里，进程之间是**老死不相往来**的。
*   进程 A 看不到进程 B 的内存。
*   进程 B 改不了进程 A 的变量。

这种隔离保证了安全，但也带来了麻烦。
比如，Shell 想要读取文件，它不能直接去读硬盘，因为它没有权限。它必须找“文件系统服务”帮忙。
但是，它怎么告诉文件系统“我要读这个文件”呢？

这就需要 **IPC (Inter-Process Communication，进程间通信)**。

## 4.2 电话 vs 邮件：同步与异步

IPC 主要有两种模式：

1.  **异步 (Asynchronous) - 寄信**：
    *   我把信扔进邮筒，然后该干嘛干嘛。
    *   对方什么时候收到，我不知道。
    *   对方回信了，我再去信箱拿。
    *   **优点**：我不被阻塞，效率高。
    *   **缺点**：逻辑复杂，不知道信丢没丢。

2.  **同步 (Synchronous) - 打电话**：
    *   我拨通电话，在那儿等着。
    *   对方接了，我们聊完，我挂电话。
    *   在通话期间，我什么也干不了，只能专心打电话。
    *   **优点**：简单直接，打完就知道结果。
    *   **缺点**：我会被阻塞 (Block)。

**EwokOS 主要使用同步 IPC。** 为什么？因为它简单、可靠。对于微内核来说，简单就是美。

## 4.3 EwokOS 的 IPC 流程

让我们看看一次典型的 IPC 通话（比如 Shell 呼叫 VFS）发生了什么：

1.  **拨号 (Client Call)**：
    *   Shell (Client) 调用 `ipc_call(vfs_pid, CMD_READ, ...)`。
    *   内核收到请求，把 Shell **冻结 (BLOCK)**，让它去睡觉。
    *   内核生成一个任务单 `ipc_task_t`，塞进 VFS (Server) 的信箱里。

2.  **接听 (Server Fetch)**：
    *   VFS (Server) 平时就在循环里等着，调用 `ipc_fetch()` 检查信箱。
    *   发现有新任务！VFS 拿到任务单，开始干活（读文件）。

3.  **回复 (Server Return)**：
    *   VFS 干完活了，把结果写好，调用 `ipc_end()`。
    *   内核收到通知，把结果搬运给 Shell。
    *   内核**唤醒 (WAKEUP)** Shell。

4.  **挂断 (Client Resume)**：
    *   Shell 醒来，发现 `ipc_call` 返回了，手里拿着 VFS 给的数据。
    *   Shell 继续干活。

## 4.4 核心代码赏析

IPC 的核心代码在 `kernel/kernel/src/ipc.c`。

```c
// 客户端发起呼叫
int32_t ipc_call(int32_t pid, int32_t call_id, proto_t* data) {
    // 1. 准备参数
    // 2. 陷入内核 (System Call)
    // 3. 内核将当前进程状态设为 BLOCK
    // 4. 切换到其他进程
}

// 服务端处理
void ipc_handle_loop() {
    while(true) {
        // 1. 检查有没有任务
        int id = ipc_fetch(); 
        if (id > 0) {
            // 2. 处理任务
            do_work();
            // 3. 告诉内核完事了
            ipc_end();
        }
    }
}
```

## 4.5 消息数据结构：`proto_t`

IPC 传递的数据被封装在 `proto_t` 结构中，它是一个灵活的数据包格式：

```c
typedef struct {
    void*    data;      // 数据缓冲区指针
    uint32_t size;      // 数据大小
    uint32_t offset;    // 读写偏移量
    uint32_t capacity;  // 缓冲区容量
} proto_t;
```

**使用方式**：
*   **写入数据**：使用 `proto_add_int()`, `proto_add_str()` 等函数追加数据。
*   **读取数据**：使用 `proto_read_int()`, `proto_read_str()` 等函数按顺序读取。
*   **序列化**：数据在发送前会被序列化成字节流，接收后再反序列化。

**示例**：
```c
// 客户端构造请求
proto_t* req = proto_new(NULL, 0);
proto_add_int(req, CMD_READ);
proto_add_str(req, "/dev/timer");
proto_add_int(req, 1024);  // 读取长度

// 发送 IPC 请求
proto_t* resp = ipc_call(server_pid, req);

// 解析响应
int status = proto_read_int(resp);
char* data = proto_read_str(resp, NULL);
proto_free(resp);
```

## 4.6 共享内存：更快的数据传递

对于大量数据的传递，每次通过 IPC 复制会很慢。EwokOS 提供了**共享内存 (Shared Memory)** 机制：

1.  **创建共享内存**：进程 A 创建一块共享内存区域。
2.  **映射**：进程 A 和进程 B 都将这块物理内存映射到各自的虚拟地址空间。
3.  **直接访问**：两个进程都可以直接读写这块内存，无需通过内核复制。
4.  **同步**：使用锁或信号量保证并发访问的安全性。

**优点**：
*   零拷贝 (Zero-Copy)，效率极高。
*   适合视频流、大文件等场景。

**缺点**：
*   需要手动同步，容易出错。
*   失去了 IPC 的进程隔离保护。

通过这种机制，EwokOS 将一个个独立的孤岛（进程）连接成了一个繁忙的群岛网络。

下一章，我们将看看这些岛屿的地基——内存管理。
