---
title: Lab1 Xv6 && system calls
date: 2025-02-25
categories:
  - OS
tags:
  - xv6
  - CPP
---
## Using gdb

平时我们用的调试工具其实都是图形化后的gdb，使用起来非常的方便，但是熟悉原生的gdb会使我们的效率进一步提升。我认为学习好用gdb调试是一项非常重要的技能。

### 编译并启动

==make qemu-gdb==

使用上述命令**编译项目**并且直接以**调试**模式启动。但是这个项目启动的其实是在本地的一个远程gdb，通过另一个窗口进行调试。
![PixPin_2025-02-25_13-54-10.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_13-54-10.png)

下面有几个关于系统调用的小问题

**Looking at the backtrace output, which function called syscall?**

![PixPin_2025-02-25_16-13-11.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-13-11.png)
![PixPin_2025-02-25_16-19-46.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-19-46.png)
显然是usertrap（）

**What is the value of p->trapframe->a7 and what does that value represent? (Hint: look user/initcode.S, the first user program xv6 starts.)**
![PixPin_2025-02-25_16-26-31.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-26-31.png)
通过提示找到trapframe的地址，在kernel/proc.h中找到a7寄存器对应的偏移地址。
![PixPin_2025-02-25_16-28-03.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-28-03.png)
把168换算成16进制
![PixPin_2025-02-25_16-29-13.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-29-13.png)
尝试着打印**0x87f560a8**
![PixPin_2025-02-25_16-30-11.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-30-11.png)
这个就是寄存器a7的值。其代表的具体含义通过所给的提示到对应文件中查找之后也是十分的明了。
![PixPin_2025-02-25_16-32-21.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-32-21.png)
应该就是系统的调用号。

**What was the previous mode that the CPU was in?**

![PixPin_2025-02-25_16-43-14.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_16-43-14.png)
spp位表示其在什么状态。 通过图示得出，spp在二进制的第8bit.
![PixPin_2025-02-25_17-10-18.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_17-10-18.png)
由图可知8bit是0，所以之前是用户状态 。
![PixPin_2025-02-25_17-26-23.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_17-26-23.png)


![PixPin_2025-02-25_17-26-05.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-25_17-26-05.png)

##  System call tracing


## Attack xv6
这个task主要是利用xv6故意留下的bug然后获取到销毁内存但是保留了脏页的数据。然后通过一些比较hack的手段把字段给找出来。
### page组成

![PixPin_2025-02-28_20-58-24.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-28_20-58-24.png)
上图是一个简化的逻辑页表，下一个实验也会用到，这里理顺一下逻辑有利于分析代码的组成。
![PixPin_2025-02-28_21-01-20.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-02-28_21-01-20.png)
上图是地址转化的细节。
页表项（PTE）包含标志位，告诉硬件应该如何使用这些虚拟地址。![PixPin_2025-03-02_10-35-10.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-02_10-35-10.png)
这些相关位的定义都在(kernel/riscv.h)中

书中这一段包含的最重要的信息是xv6采用了sv39的riscv架构。
![PixPin_2025-03-02_12-01-01.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-02_12-01-01.png)
这个很好的解释了sv39的组成。显然sv39是三级页表的架构。