麻雀虽小五脏俱全的一个小而全的类unix教学用操作系统，由MIT开发

多核的操作系统，大约6000行还有大约300行的汇编，代码简洁易懂，有很多设计都值得借鉴。

## 一些特性
1. 有进程管理功能
2. 有虚拟地址和空间，pagetable
3. 文件系统
4. 时间片
5. 21个syscall

### 用户程序
sh cat echo grep kill 
能够被视为一个真正的操作系统

### 缺失的功能
1. 用户ID
2. 登录功能
3. 文件保护
4. 虚拟内存
5. 无法联网
6. 只有两个设备驱动


## xv6特性

### SMP 多线程共享内存


### 设备

1. UART 发送字节流，从一端到另一端
2. disk 磁盘驱动器
3. 定时器
4. PLIC 平台级中断控制器，哪个核心应该被告诉中断
5. CLINT 本地中断控制器

### 内存管理

1. pagesize 4096B
2. 一个freelist
3. 没有可变的内存分配
4. 没有“malloc”
5. 三级页表 sv39，在pgtbl 巨页中有用到

### 调度器

基本的循环调度
时间片的大小是固定的
所有的核心共享一个就绪队列，循环的寻找可以执行的进程
不是完全的循环，如果在时间片内结束，会把这个进程放进队列然后换一个来

### 启动顺序

加载核心代码到固定的地址0x8000_0000,没有bios，bootloader bootblock

### 锁
1. 自旋锁，sleep（），wakeup（）

### param.h
![PixPin_2025-03-21_15-04-00.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_15-04-00.png)


## 用户地址空间

![PixPin_2025-03-21_14-48-26.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_14-48-26.png)

函数的参数将会被放在栈中。
### xv6的虚拟地址
sv39，三级页表之地
![PixPin_2025-03-21_14-54-26.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_14-54-26.png)
![PixPin_2025-03-21_14-55-13.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_14-55-13.png)
实际上xv6只使用了38位所以最大的是256GB

## xv6启动过程和组织

![PixPin_2025-03-21_14-56-23.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_14-56-23.png)
![PixPin_2025-03-21_14-57-53.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_14-57-53.png)
控制权逐步的转移
### 数据的类型
![PixPin_2025-03-21_15-03-25.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_15-03-25.png)



## 自旋锁
![PixPin_2025-03-21_15-05-59.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_15-05-59.png)
循环不断的检查，直到能够获取锁为止
![PixPin_2025-03-21_15-08-52.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_15-08-52.png)
test and set，原子的操作。
释放的时候直接修改值就行，可能看起来不是很原子，但是在内存里这被视为一个单独的操作。

不应该长时间被持有，获取锁之后应该尽快释放。可以使用sleep和wakeup进行长时间的获取锁。

### 一个使用自旋锁的例子 buffer


### pushoff popoff




## 内存管理

### kalloc and kfree

![PixPin_2025-03-21_15-48-06.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_15-48-06.png)
将在初始化的时候第一个使用


## 系统调用

### 在用户模式下

==kill（）==
![PixPin_2025-03-21_16-27-51.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_16-27-51.png)


## risc-v架构

### 寄存器

![PixPin_2025-03-21_16-25-11.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_16-25-11.png)
一共32个寄存器
a0用于存放返回函数返回值

### 三种模式

1. 机器模式（machine mode），是最高权限的模式，但是用的并不多，基本是两种情况，一个是启动和初始化，另一个是定时器中断，程序会很快的转变为对supervisor mode的中断。
2. supervisor mode，所有的内核态代码和一些特权指令（只能在S 和M模式下执行的）。
3. 用户模式，我们可以说在supervisor 模式下运行的代码都是核心代码，反之亦然
![PixPin_2025-03-21_16-47-34.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_16-47-34.png)


![PixPin_2025-03-21_16-46-24.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_16-46-24.png)
### 19个寄存器
![PixPin_2025-03-21_16-55-18.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_16-55-18.png)



## 页表结构

![PixPin_2025-03-21_17-05-15.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-05-15.png)
![PixPin_2025-03-21_17-06-24.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-06-24.png)
更改satp的时候需要更新flash

![PixPin_2025-03-21_17-08-08.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-08-08.png)
只用了38位

![PixPin_2025-03-21_17-09-10.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-09-10.png)

![PixPin_2025-03-21_17-13-41.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-13-41.png)

![PixPin_2025-03-21_17-21-58.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-21-58.png)

![PixPin_2025-03-21_17-23-49.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-23-49.png)
![PixPin_2025-03-21_17-24-22.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-24-22.png)

在user mode下
![PixPin_2025-03-21_17-26-46.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-26-46.png)
![PixPin_2025-03-21_17-27-31.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-27-31.png)

在机器模式下
![PixPin_2025-03-21_17-30-08.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-30-08.png)
![PixPin_2025-03-21_17-30-43.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-30-43.png)
![PixPin_2025-03-21_17-34-16.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-34-16.png)

![PixPin_2025-03-21_17-34-37.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-34-37.png)

## 上下文切换
![PixPin_2025-03-21_17-45-28.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-45-28.png)
![PixPin_2025-03-21_17-48-32.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-48-32.png)
![PixPin_2025-03-21_17-48-53.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-48-53.png)
![PixPin_2025-03-21_17-51-00.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-21_17-51-00.png)


## trapfram 和trampoline

## 调度函数

### switch.s

![PixPin_2025-03-22_11-45-57.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-22_11-45-57.png)


## UART.C 

16550a芯片与外部设备进行交互

## 虚拟内存的操作函数

### 页表结构
![PixPin_2025-03-24_11-15-09.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_11-15-09.png)
一个页表树
![PixPin_2025-03-24_11-15-44.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_11-15-44.png)
 标志位只在最后一级页表是最重要的，前两级都只有有效位是有用的，其他都为0.

### 一些函数

==walk（）==

![PixPin_2025-03-24_11-18-15.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_11-18-15.png)
 返回页表项的地址。
 
==mappages（）==

![PixPin_2025-03-24_11-20-10.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_11-20-10.png)

==walkaddr（）==
把虚拟地址转换成物理地址。
必须是可用的而且是 用户页
![PixPin_2025-03-24_11-24-26.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_11-24-26.png)
![PixPin_2025-03-24_15-24-17.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_15-24-17.png)
![PixPin_2025-03-24_16-27-32.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-03-24_16-27-32.png)
