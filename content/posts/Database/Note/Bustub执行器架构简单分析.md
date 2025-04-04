---
title: Bustub架构简单分析
date: 2024-02-21
categories:
  - CMU15-445
tags:
  - Note
  - CMU15-445
---
## 写在前面

在这篇文章中我们先抛开SQL在bustub的中的历程，直接快进到最后开始执行的阶段，这篇文章只关注设计上的细节。

至于如何理顺这些细节在[[CMU15-445 Project3 - Query Execution]]文章中会有详细的介绍，这里就不再赘述。😃

## 架构总览

![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221212132.png)

上图中catalog其实不是特别准确，按照我的理解应该是下面这种情况👇。通过一个table_id双向映射表的名字和实体table。
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221212842.png)

接下来就按照从上到下的顺序依次介绍各个部分的结构。

## Catalog

下面这个图是是整个catalog关于table的。👇这里引出了`TableInfo`.
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221221311.png)

Catalog中不仅仅包含table的信息还储存了index的信息。这里引出了`IndexInfo`,但是后边用的不是很多。
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221222700.png)


接下来看`TableInfo`的组成。（TableInfo的定义就在Catalog.h文件中）

## TableInfo

![主要成员变量|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221223744.png)

- name_:就是表的名字
- table_: 是一个每个节点都是tablepage的双向链表
- oid_: 顾名思义就是表的id
- Schema_：其实到现在我都没有很弄懂这个是干什么的东西，但是chat给出了答案。

### 什么是Schema
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221224249.png)
按照我的理解应该是一个类似于表头的包含各种配置信息的一个抽象集合。现在就看看源码。👇
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221225429.png)

- 在bustub中schema的信息似乎没有包含很多，就是对每一个列的数据类型约束进行了记录。以及对所有的列进行一个汇总，方便获取到每一个列。
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221225647.png)

==Column==

实际上每个列的实现实体是`Column`，在这个实体中包含**列名，数据类型，长度**等基本信息。
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221230318.png)

⚠️值得注意的是column对于`varchar`单独做了处理🙂,变长数组终究还是不一样啊。
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221231559.png)

## TableHeap

实质上就是配合`TablePage`里面的信息构成的双向链表，我觉得这个设计真的非常的巧妙。只用了基本的`pre_page_id` 和`next_page_id`这两个变量就把双向链表给建立起来而且耦合度非常的低。

![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240221233636.png)
- first_page_id_: 就是第一个pageid
- bufferpool：所有的page都需要从bufferpool中去取。

### TablePage的结构

![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222001937.png)


![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222001143.png)

ok看到这里有疑问了？ 那么多的信息都存在那里， 这继承的加上初始化的也不够啊。
答案在这里👇
![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222001417.png)

通过`偏移量`直接在page的剩余空间里去定义各种变量。

==通过InsertTuple（）函数了解详细结构==

直接看源代码：

```cpp
 if (tuple.size_ + 32 > BUSTUB_PAGE_SIZE) { 
  // larger than one page size
    txn->SetState(TransactionState::ABORTED);
    return false;
  }
  auto cur_page = static_cast<TablePage *>(buffer_pool_manager_->FetchPage(first_page_id_));
  if (cur_page == nullptr) {
    txn->SetState(TransactionState::ABORTED);
    return false;
  }
```

- 这主要是判断一些前置条件的合法性

*为什么要用tuple的大小+32进行比较呢？* 文末给出答案

![image.png|800](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222002633.png)

- 在第一页中进行插入，如果没有足够的空间就开一个新页然后插进去
- ⚠️需要注意的是，在离开第一页的时候**仍然要持有写锁**，因为还要写`next_page_id`变量.

💡跟随代码跳转到`TablePage::InserTuple()`.

```cpp
auto TablePage::InsertTuple(const Tuple &tuple, RID * rid, Transaction * txn, LockManager * lock_manager,LogManager * log_manager) -> bool{}
```
👆是函数的定义。

![image.png|725](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222003627.png)

- 开始主要还是判断空间是否够，不够的话直接返回插入失败。
- 剩下的情况就是空间足够可以进行插入操作然后返回成功。

![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222004046.png)

- 然后就是对内部的一个变量的改变。主要是数量的改变和指针指向的改变
- 数量自然是加一，指向空位置的指针也要往回走这个tuple.size()的大小。

💡返回失败之后那自然就要找下一个page，那么就会出现两种情况。
- 下一个有page那就直接取然后插入
- 下一个没page那就new一个page然后初始化

![image.png|600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222011534.png)


![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222011735.png)

看到这其实整个bustub的架构已经差不多清晰了

## Tuples

每一次插入的最小单元`tuple`到底是什么东西呢？

![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222015143.png)


![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222014021.png)
这个可以简单的理解为表中的一行。
👇是tuple初始化的过程，
![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222015023.png)

💡 ==RID==
![image.png|650](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240222015237.png)
- 由一个slot_num_和page_id_组成。相当于一个坐标，快速定位在page中的位置。

目前就这么多，剩下的细节等待以后再补充吧。


## 复杂算子的结构

task1中基本上所有的算子实现都已经给出了，而且整体结构也都比较简单直白。
接下来主要梳理一下task2的算子结构，🙃。

不记录一下感觉源码直接从大脑溜走了，一点也记不住。
### Aggregation & Join Executors

#### Aggregation

这个算子相对来说就是比较复杂的一个东西了，我现在其实还不是很明白这到底是是个什么东西。
![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240225213804.png)
这个算子的一个特别之处就是：Aggregation 是 pipeline breaker。也就是说，Aggregation 算子会打破 iteration model 的规则。原因是，在 Aggregation 的 `Init()` 函数中，我们就要将所有结果全部计算出来。原因很简单，比如下面这条 sql：
```sql
SELECT t.x, max(t.y) FROM t GROUP BY t.x;
```


`SimpleAggregationHashTable`,在这个hash表中维护的key和value分别是`AggregateKey`,`AggregaValue`.
![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240225223229.png)
其实到这里就已经能大致猜到这两个类型是什么东西。但是为了一探究竟我们还是看到底。

==AggregaKey==

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240225223840.png)
从这里我们可以得到两个信息，
- 类型是`std::vector<Value>`
- 看注释的的话这个字段就是`group by`的字段。

==AggregaValue==

![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240225224847.png)
得到的信息和上面的类似
- 类型是`std::vector<Value>`
- 这个字段是 value 则是需要 aggregate 的字段的数组。

`InsertCombine()`


![image.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20240225232225.png)

- 上边的`CombinAggregateValues()`是需要自己进行实现的。但是根据留出来的空大致也能猜出来是根据不同的聚合类型把新插入的数据按照给定的规则进行`更新`.
- `InsertCombin()`:如果原来的hash表中没有给定的规则，那就创建一个新的记录。如果有的话就调用上述的函数，讲哈希表中已经存在的按照既定规则出来的值进行更新。

为什么在插入的过程中需要`更新`呢？ 原因也是非常简单了---> 加入规则是找一个最大的数，这个时候进来的这个数比表中原来的最大的数大，那自然就需要进行改变。

所以上述的这个函数实现我想大致思路已经出来了，把每一个新的进来的数据和原来的里面存的数据进行比较，按照规则判定是否需要更新。

---

==举个例子==

```sql
SELECT min(t.z), max(t.z), sum(t.z) , count(t.z) FROM t GROUP BY t.x, t.y;
```

在这个上述过程中函数中每个变量对应的是什么？

`AggregateKey`  上文提到是group的字段 在这个sql中对应的自然就是`「 t.x ，t.y」`
`AggregateValue`  对应的是四个类型 `「t.z t.z t.z t.z 」`
`AggregationType`  对应的规则为 `「min ，max ，sum ，count」`

通过相对位置来确定对应的规则和值。

#### NestedLoopJoin

在chatGPT中对这个循环嵌套给出了如下的解释。但是想实现好确并不容易。

![Clip_2024-03-10_19-05-10.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/Clip_2024-03-10_19-05-10.png)

Andy 在 Lecture 里也详细地量化地对比了各种 Join 的 costs，有兴趣可以看看。


[risinglight/src/executor/nested\_loop\_join.rs at main · risinglightdb/risinglight · GitHub](https://github.com/risinglightdb/risinglight/blob/main/src/executor/nested_loop_join.rs)

参照上述的join实现比较的优雅。这也是Bustub的作者参与的一个开源项目，参考原作者的实现我想会有意想不到的奇效。

==思路整理==

`第一版`

按照课上讲的思路，就是两个循环直接进行嵌套发现有一样的就直接返回

```cpp
while(leftchild->next(lefttuple)){
for(auto rightchild : righttuple){
   if(leftchild==rightchild){
   *tuple = ...;
   return true;
   }
}
}
```

但是按照上述的思路我们会发现一个很致命的问题，如果右边的tuple里面有重复的值与左面的值匹配的话上述代码只会匹配`第一个`相等的tuple然后就返回了。

👇举个例子
```txt
 t1.x | t2.x
	-----
	1 | 0
	2 | 1 <---
	3 | 1
	4 | 2
```

按照正确的应该t1.x的1和 t2.x 中的两个1进行匹配
但是上述代码的逻辑只会匹配到箭头指的地方然后就返回了。





	




---

+32去比较的原因是父类page和tablepage的head一共需要预留32b的位置。