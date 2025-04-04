---
title: 虚拟化--进程管理API：fork, execve, exit
date: 2025-02-02
categories:
  - OS
tags:
  - xv6
  - 虚拟化
---
# 操作系统上的进程

**背景回顾**：有关状态机、并发和中断的讨论给我们真正理解操作系统奠定了基础，现在我们正式进入操作系统和应用程序的 “边界” 了。让我们把视角回到单线程应用程序，即 “执行计算指令和系统调用指令的状态机”，开始对操作系统和进程的讨论。

**本讲内容**：操作系统上的进程

- 操作系统上的第一个进程
- UNIX/Linux 进程管理 API: fork, execve, exit

## fork（）

![PixPin_2025-02-03_14-37-19.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-03_14-37-19.png)

![PixPin_2025-02-03_14-51-04.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-03_14-51-04.png)


**理解 fork()**: fork() 会完整复制状态机；新创建的状态机返回值为 0，执行 fork() 的进程会返回子进程的进程号。同时，操作系统中的进程是并行执行的。程序的精确行为并不显然——model checker 可以帮助我们理解它。

在这个例子中，我们还发现执行 `./a.out` 打印的行数和 `./a.out | wc -l` 得到的行数不同。根据 “机器永远是对的” 的原则，我们可以通过提出假设 (libc 缓冲区影响) 求证、对比 strace 系统调用序列等方式，最终理解背后的原因。标准输入输出的缓冲控制可以通过 setbuf(3) 和 stdbuf(1) 实现。

## execve（）

![PixPin_2025-02-03_14-54-18.png|525](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-03_14-54-18.png)

![PixPin_2025-02-03_14-55-00.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-03_14-55-00.png)


## exit（）
