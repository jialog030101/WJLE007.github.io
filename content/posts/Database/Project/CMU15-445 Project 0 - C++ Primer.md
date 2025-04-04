---
title: "CMU15-445- Project #0 - C++ Primer"
date: 2023-11-28 23:44:18
categories:
  - CMU15-445
tags:
  - Project
  - CMU15-445
---

# CMU15-445
作为 CMU 数据库的入门课，这门课由数据库领域的大牛 Andy Pavlo 讲授（“这个世界上我只在乎两件事，一是我的老婆，二就是数据库”）。

这是一门质量极高，资源极齐全的 Database 入门课，这门课的 Faculty 和背后的 CMU Database Group 将课程对应的基础设施 (Autograder, Discord) 和课程资料 (Lectures, Notes, Homework) 完全开源，让每一个愿意学习数据库的同学都可以享受到几乎等同于 CMU 本校学生的课程体验。

亮点在于这门课程实现了一个关系型数据库的Demo--bustub，并对组成部分进行修改，最后达到完成整个数据库的目的，非常的有意思。

近年来由于CMU15-445这门课的热度大增，无论是找工作还是保研的简历上都少不了这个课程。到了可以说是人手一个的地步（还有MIT6.824），但是最后的排行榜上真正完成4个项目并且有成绩的不过100余人。在这种背景下也催生出了很多卖课的机构和个人。还有种种乱象。暂且按下不表，进入主题。
## 前言
先附上 [课程链接](https://15445.courses.cs.cmu.edu/fall2022)我原本是想要完成最新的课程即2023fall的，但是2023fall的P0直接就把我劝退了，相较于2022fall的P0我认为难度上升了不止一点。但是其他的Project的实现差别并不是太大，所以我选择了较为容易下手的2022fall的课程进行学习。
# Project0-C++ Primer

还是先放上[项目Project#0-C++primer链接](https://15445.courses.cs.cmu.edu/fall2022/project0/)
## 概述

这是一个相当于先导课程的项目，**旨在培养和检验学生的现代C++编程能力** ，BusTub大量使用C++17，当你上手之后**可能**就会发现和你印象中的C++不一样。

在CMU如果你没能满分通过P0，那么你会被要求退课。
![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129141237.png)
## 所需前置知识
我认为需要的前置知识主要包括[**C++11的新特性**](https://github.com/cmu-db/15445-bootcamp)
这是在2023fall课程的一个可能是助教写的一个小demo包含了几个例子能够帮助你快速的上手c++11的新特性。

熟悉**字典树**能够理解字典树原理能够完成一个字典树的小demo[LeetCode.208实现前缀字典树](https://leetcode.cn/problems/implement-trie-prefix-tree/description/) 可以先把这个做了，了解一下什么是字典树。

![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129144859.png)

上图就是一个字典树的简单示意图
## 文件结构
![image.png|264](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129162120.png)

在文件中作为字典树的架子文件中包含了三个类。
- TrieNode 
- TrieNodeWithValue
- Trie
## Task#1 字典树
先完成一个单线程版本的字典树，后期再考虑并发。
### TrieNode
`TrieNode`定义了一个Trie树的一个节点。
![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129160311.png)
- 一个包含关键字的char类型`key_char_。
- 一个bool类型`is_node`的标志来标记这个节点是否是**值节点**
- 包含一个存储类型为 `char` to `unique_ptr<TrieNode>`映射的哈希表
**需要注意的点：** 
C++中智能指针[std::unique_ptr的特性](https://www.learncpp.com/cpp-tutorial/stdunique_ptr/)所带来的：
The `InsertChildNode` and `GetChildNode` both return a pointer to `unique_ptr`
这个独享类型的智能指针也不支持被复制，所以要么使用`std::move()`把所有权交出去，要么就用`get()`函数把裸指针交出去。
the move constructor `TrieNode(TrieNode &&other_trie_node)` is used to transfer old TrieNode's unique pointers to a new TrieNode。因为是unique_ptr所以在传递的时候不能对指针进行复制。（解决方法就是使用move()函数）。
![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129173037.png)

![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129173224.png)

### TrieNodeWithValue
![image.png|290](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129180408.png)

- 这个是[[CMU15-445 Project 0 - C++ Primer#TrieNode]]类的子类，多了一个能保存任意类型的值T。而且is_end被设定为True。
- 根据不同的情况调用不同的构造函数。如果原本是一个普通节点就调用参数为（TrieNode&&，T），如果是创建一个全新的节点则调用参数为(char,T).
### Trie Class
- 有一个root_节点，是整个树的根节点，而且不存储任何的值。
#### Insert
要插入到trie中，您需要首先使用给定的键遍历trie，如果不存在，则插入TrieNode。请注意，不允许插入重复的键，并且应该返回false。一旦到达键的结束字符，有三种可能性：

1. 到达的节点不存在，需要调用参数为(char,T)的构造函数构造一个全新的值节点。利用“Make use of the fact that a `unique_ptr` to `TrieNode` can also store a `unique_ptr` to `TrieNodeWithValue`”这一特性可以以最后一个节点为模板（无论是不是值节点都能够成功创建）创建一个新的值节点。
2. 到达的节点存在但不是值节点，调用（TrieNode&&，T）为参数的这样一个转换的构造函数，把普通节点转换为值节点。
3. 存在且是值节点，直接返回false。
![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129185358.png)
#### Remove
我认为Remove()是这几个函数中相对来说比较难的，因为涉及到递归删除。

目的就是删除给定路径的节点的值。
1. key不存在直接返回
2. 把终端节点的is_end_的值指定为false。
3. 如果一个节点没有任何孩子那就应该把他删除
4. 遍历trie并递归删除没有子节点的节点。遇到包含子节点的节点时停止。
#### GetValue
没找到或者类型不匹配将success的值设置为false。
![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129190637.png)
个人认为这个的关键点在于节点类型的转换从而获取到节点的值，又因为current是unique_ptr类型所以只能用get()先获取裸指针再进行转换。
## Task #2 - 多线程字典树
简单概括就是涉及读的操作就上读锁，涉及写的操作就上写锁，涉及读写的操作就上读写锁
千万别忘记返回直接解锁，以免造成死锁。
![image.png|200](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231129191231.png)
最后附上通过截图。
