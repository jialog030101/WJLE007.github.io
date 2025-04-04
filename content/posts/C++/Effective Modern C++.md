---
title: Effective Modern C++ 笔记
date: 2023-12-09
categories:
  - C++
tags:
  - CPP
  - Note
---
## auto


在概念上auto已经极简了，但是实际上仍然要微妙许多。它可以节约声明类型，也可以避免许多手动类型的正确性和声明问题。但是从结果的角度来说，尽管auto很努力在做事了，但是仍然可能是错误的。如果出现这种情况我们要知道如何去引导auto让他成为正确的类型，因为退回使用手动声明类型仍然是下下策。

接下来的内容会涵盖`auto`的所有细节

### Item 5: Prefer auto to explicit type declarations.

不仅可以避免出现为初始化的变量，避免啰嗦又繁杂的类型声明，能直接持有闭包，还可以避免一些因为“类型捷径”出现的问题。
eg1.

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231210204514.png)


	有一些程序员会对类型发生误判，到时用范围较小的类型在32位机器上能够运行但是在64位机器上发生了改变，导致程序在移植的时候出现问题。

eg2.
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231210205424.png)


上述代码看起来没什么问题，但是当实际运行之后并没有对哈希表m进行操作。原因在于哈希表中的kay值是const类型的，手动声明的类型不一样的话编译器会进行一个神奇的操作，它会讲m中的内容复制成为临时变量将key的类型改为和声明一致的，再将p绑定到临时变量上。


#### Summary

auto说到底只是一个可选项罢了，不是必选项，如果你觉得你的项目使用显式的声明能够使得项目变得更加的可读和高效，当然可以继续使用。但是c++引入auto并不是一个多新鲜的东西，只是一个在其他语言中被称为“类型推导”的东西罢了。在其他的静态类型语言中类型推导都或多或少存在。而且动态语言还为类型推导积累了大量的经验。而且此类技术并不会于大型的工程项目产生冲突。

一些人觉得用完auto之后会让变量的类型变得不是一眼可以识别，但是这个问题随着ide的优化和适配已经被解决的相当完美了。

事实上手动声明变量经常是在画蛇添足，无论是正确率还是效率上。


### Item 6: Use the explicitly typed initializer idiom when auto deduces undesired types.

纵使auto有万般好，但是auto也会出现推断的类型和你心目中期待的不一样的情况（当然也不完全是auto错了哦😀）
下面举一个auto推断不符合预期的例子。


![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231210214843.png)

上述声明了一个返回vector< bool >类型的函数，使用了auto之后虽然仍然能够直接进行编译和运行但是结果却不是所想要的。

而发生这样错误的原因是，在c++的设计当中回避了bit的引用所以返回的不是bool&，实际上c++中设计了一个**代理类**来完成向bool的转换操作，这就不展开细说了，但是代理类并不少见，我们经常使用的两个智能指针就是一种代理类。

代理类的设计在使用的时候尽量少的对程序员暴露内部细节，这些代理类的使用往往会在文档中标识出来，如果在文档中没有体现的话，也避免不了在头文件中漏出一些破绽，最不济在debug的时候可能也会发觉是使用了代理类。

但是这些都不重要了，现在重要的事怎么把auto引导到正确的道路上🙃。

![image.png|700](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231210220544.png)

就是使用一个强制的类型转换，让它去到该去的位置，虽然看起来有点滑稽，给人一种头痛医脚的感觉🤣。虽然在这些时候你也可以放弃使用auto，但是在这个踩坑的过程中不是也收获了新的知识么?😁

总之记住以下两点
![image.png|700](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231210220919.png)

- 隐形的代理类可能会让auto推断出错误的类型。
- 显式的类型转换可以让auto走向正确道路。


## Moving to Modern C++

接下来这一章会详细介绍现代c++的一些细节特性。我自己会记录几个我认为比较实用的。并不是全部。

### Item 8: Prefer nullptr to 0 and NULL.

非常显然的是0的类型是**int**，不是一个指针，但是在一个本应该出现指针类型的地方出现了0，编译器也会勉强吧0解释为空指针，但这毕竟是不得已而为之的行为🙉， 总之0是int，不是指针。

>nullptr’s advantage is that it doesn’t have an integral type. To be honest, it doesn’t have a pointer type, either, but you can think of it as a pointer of all types. nullptr’s actual type is std::nullptr_t, and, in a wonderfully circular definition, std::nullptr_t is defined to be the type of nullptr. The type std::nullptr_t implicitly converts to all raw pointer types, and that’s what makes nullptr act as if it were a pointer of all types.

所以用nullptr总会的到正确类型的空指针，而不用编译器进行不情愿的转换。不仅如此，使用nullptr还会提高代码的**可读性**。

![image.png|625](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231211190700.png)

![image.png|625](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231211190718.png)

通过以上两段代码的对比我相信很快就能感觉到差距所在。其实最重要的是在使用auto的时候0和null可能会进行错误的推导。

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231211191119.png)

- 尽量采用nullptr而不是0或者null
- 避免在整形和指针类型之间的重载。


### Item 12: Declare overriding functions override.

在需要重写的函数中添加override声明，好处就是能减少出错的几率吧，算是一个编程的好习惯。

### Item 13: Prefer const_iterators to iterators.

在c++11之前中标准库中缺少cbegin和cend这样带有c语言风格的函数，但是实际上const的begin（）和cbegin（）几乎是没有任何区别的，不知道为什么要这样设计，据说是成员函数设计与自由函数设计之争🙃？不是很懂。

总之使用const_iterators是有好处的，起码你的代码变得标准起来了不是么？



### Item 16: Make const member functions thread safe.

当我们用一个类表示多项式会非常的方便，一般会有一个计算多项式根的函数，这个多项式一般不会造成多项式值的改动，所以把这个函数声明为const成员函数是一件非常正常的事情。

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213001240.png)

（item9是说优先使用别名而不是typedefs）

由于计算机计算多项式解的过程开销是非常大的，所以即使是计算也会把结果缓存起来，减小开销，以下是一种基本的做法：

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213002615.png)


在一般的const函数中不会改变任何值，但是在本例中，由于根的组成部分不只是值还有一些标志位，在求值的过程中标志位会发生改变。

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213002937.png)

这两个线程就会产生不安全的行为。最常见的解决办法就是加入互斥变量。

但是用latch似乎有点杀鸡用牛刀了。

如果只是记录函数的调用次数并且保证其他函数能够观测到这一行为可以使用 
`std::atomic counter`，下面将演示如何使用这个原子的计数器。
![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213003544.png)

但是需要谨慎的使用这个原子的计数器，以下是两个反例。


![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213003652.png)

在这个例子中假如第一个线程进行到=true的部分，这时第二个线程进入发现标识位为true于是返回了一个不是经过计算的值。

如果将这两个句子的顺序颠倒会发生一些不一样的东西么？
答案是不会。这个函数仍然是不完美的

因为延迟修改标志位，所以有可能会出现重复的计算。

对于拥有两个以上的需要同步的变量尽量采用mutex吧。
以下是一个好的示范。

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213004059.png)


![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231213004125.png)
- 保证const成员函数的线程安全
- 使用原子的计数器，但是需要谨慎。

## Smart Pointers

原始指针虽然非常的强大，但是往往伴随着以下几点问题：

1. 指针的声明并不能直观看出是指向一个对象或者一个数组。eg.一个int和一个int数组的指针都是int*。
2. 不知道在使用之后是否应该销毁指针所指的对象。也就是说不知道指针是否拥有所指对象。
3. 即使知道了所指的对象也不确定具体的销毁方法，是用delete还是其他特殊的析构函数。
4. 如果知道了用delete是可行的，但是由于原因1，所以不知道delete一个对象还是delete[]一个数组。
5. 没有什么方法能可靠的检测悬空的指针。

**智能指针**是解决这种问题的一种方法，智能指针是对原始指针的一种包装。行为很类似，却能避免很多坑，所以在日常的代码中应该优先选用智能指针而非裸指针。

在我接触到的代码中智能指针基本上就会用两个：`std::unique_ptr`和`std::shared_ptr`。

### Item 18: Use std::unique_ptr for exclusive-ownership resource management.

当你需要使用智能指针的时候`std::unique_ptr`几乎就是首选，它拥有和裸指针差不多的大小，各种操作的效率也差不多。如果裸指针能够满足要求的话，那么`std::unique_ptr`也一定可以的。

`std::unique_ptr`实现的是专属所有权语义。一个非空的`std::unique_ptr`总是拥有其对对象的所有权，除非用`std::move()`函数把所有权从原指针“移动到”目标指针，原指针则会被置为空。但是`std::unique_ptr`绝对**不允许进行复制**。

同时也可以使用`get（）`函数返回裸指针

`std::unique_ptr`可以非常方便的转换为`std::shared_ptr`
![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231215222029.png)

  这就为代码的规范化编写提供了很多的方便。


![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231215222130.png)

- `std::unique_ptr`是一种高速，小巧，只具有移动语义的对对象想有专属所有权的智能指针。
- 默认情况下，析构函数是delete，但是也可以手动指定析构函数。
- 将其转化为`std::shared_ptr`是非常容易的

