# 第七章：把手弄脏 - 编译与运行

纸上得来终觉浅，绝知此事要躬行。
看完了理论，是时候亲手把 EwokOS 跑起来了！

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
    进入树莓派构建目录：
    ```bash
    cd kernel/build/raspi/pix
    make
    ```
    如果一切顺利，你会看到 `kernel.img` 生成了。

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
cd kernel/build/raspi/pix
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
