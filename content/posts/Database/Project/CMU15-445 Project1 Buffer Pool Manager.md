---
title: "CMU15-445- Project #1 - Buffer Pool Manager"
categories:
  - CMU15-445
date: 2023-12-08
tags:
  - Project
  - CMU15-445
---
## 环境搭建

按照P0的指导中进行环境的搭建，由于我本人现在时WIN平台，课程没有提供一键配置环境的脚本，所以我选择云服务器进行环境的构建，很稳。

但是我的云服务器配置只有2核2G在编译的时候会爆内存。这里还有个故事，在爆内存之后我已经意识到这个问题，我给阿里云客服提了个工单试图能白嫖一些配置，客服还是很有水平的，直接就分析出来我是内存爆了导致的，但是直接建议我加钱升级配置，不告诉我设置swap分区，有点不厚道了。

话说回来，设置完swap分区之后就没有再出现过爆内存的现象。用VScode的远程开发，体验还是很不错的。

## Overview

p1主要是内存管理部分的内容，这门课程的课本和录制视频还是很有必要看一下的。

主要的任务是以下三个部分：

- 可扩展的哈希表
- LRU-K的淘汰策略的视线
- Buffer Pool的一个实例

在开始梳理项目之前我想要先解释清楚一下三个任务之间的关系，以及各个部分的负责。

frame更像是一个**载体**，而page是其中的内容。打个比方来说，一个仓库（buffer pool）只有100辆货车（frame），而货物（page）成千上万种，每次都是仓库告诉货车需要什么货物然后由货车带着货物来到仓库，而可扩展的哈希表就是登记货车和货物的**对应关系**。因为只有100辆货车，当每辆车都是满的时候（没有闲置的车辆free_list不为空）就需要根据task2的LRU-K算法挑选出一个符合要求的货车清空后去装载指定的货物。
而task3就是负责整个的管理和交互。

大概解释了一下，可能比方不是很恰当，做的时候一定要多看文档和注释，因为漏看注释给我带来了极大的痛苦。


## Task #1 -ExtendibleHashTable 


### 相关函数

- `Find(K,V):` 查询一个Key是否存在，如果存在则将其V指针指向相关的值，返回true，否则返回false
- `Insert(K,V):` 插入一个（K，V），如果插入失败需要进行一下步骤的重试：
	1. 插入失败肯定是桶满了，但是桶满了分为两种不同的情况。
	2. 一种是global深度和桶的local深度不相同的时候，需要对桶进行重新的分配。
	3. 当两个深度相同的时候说明桶是真的满了，需要对哈希表进行扩展
- `Remove(K):` 在哈希表中移除对应的（K，V）对，但是需要进行哈希表缩小的操作。
- `IndexOf(K):`  当前global深度下的映射规则。




### 项目架构

首先要清楚这个是自己动手实现一个可扩展哈希表用于保存frame和page的**映射关系**的。我觉得首先要明白frame和page之间的关系。不清楚的可以回看上文的解释。

### Bucket 存储桶

在可可以扩展的哈希表种引入和Bucket的概念，用于解决**哈希冲突**的问题，每一个Bucket的大小都是固定的，当一个Bucket满了的时候就需要进行哈希表的**扩展**。

![image.png|200](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231205232548.png)


### Bucket的组成
![image.png|325](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231205233054.png)

- **depth_**：表示本地深度的一个标识，用于识别当前的深度是否和全局深度相同。要特别注意这个变量。
- **list_**：发生哈希冲突时保存多个（K，V）映射的双向链表。
- **size**：为了保证效率，list的长度不能无限延伸，所以达到一定长度的时候要进行哈希表的扩展。

###  可扩展的哈希表结构

![image.png|325](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231205234304.png)

- Bucket_size:用于限制每个bucket的list的长度
- dir_:用于哈西运算后保存映射关系。
- global_depth_:全局深度，在**插入**的时候根据**全局深度**和**局部深度**来确定这个桶是否要进行重新的分配。
- num_buckets:用于统计现在一共有多少bucket。

### ReHash的过程详解


> 1.hash的规则是什么样的

```c++
template <typename K, typename V>

auto ExtendibleHashTable<K, V>::IndexOf(const K &key) -> size_t {
  int mask = (1 << global_depth_) - 1;
  return std::hash<K>()(key) & mask;

}
```

上述函数就是该哈希表的映射规则，总的来说就是取hash函数值的**低`global_depth`位**

假如现在global_depth_的值为3。
那么 `1 << global_depth_`的值就是8转换为二进制就是 1000 。mask=8-1=7，转换为二进制就是 111. 

mask 的作用就是取一个位数和global_depth_一样的全1的二进制数，为了取低global_depth_位做铺垫。

现在假如key的值进行完hash之后是 10110100011010 和 mask进行进行 & 的操作之后就变成了 010 （低3位）

刚开始的时候我看这个函数设计左移操作和&操作很复杂的样子，我就没怎么在意，不是很懂这个函数在干什么。但是现在想来 **mask** 应该是**掩码**的意思，所以和计网里掩码的作用是一样的，都是取二进制数位的作用。

这个函数没搞懂真的给我带来很大的困扰，一直搞不懂到底是怎么映射的。

> 2.什么时候进行ReHash，以及详细过程。


当一个桶内的list长度达到规定上限的时候进行Rehash。

但**不是**立刻将所有桶内的映射关系**都立刻**进行重新分配。因为当哈希表比较大的时候这样会带来瞬时的高负载，给机器带来不必要的负担。

那么什么时候对扩展哈希表后的桶内的（k,v）进行重新的映射呢？

可以认为是什么时候用上了什么时候再进行重新的映射。这里就提现了**局部深度**和**全局深度**的作用。如果我们将每一次哈希表的扩展都立即为一次**版本的迭代**的话。那么局部深度和全局深度就分别代表了**桶所处的版本**和**现在的最近版本**。这样就能实现对哈希表的线性时间扩展。

我们可以想象这样一个过程，现在全局版本为V3（global_depth_=3）而某个桶的局部版本为V2（local_depth_=2）. 这时候我们刚好需要向这个桶中插入一个数据，这时候发现不仅这个桶满了而且还落后版本，我们首先要做的就是更新桶的版本，把桶里面不符合新的映射规则（比原来多一位）的给送到新桶里。

>这里有一个小坑：哈希表扩展后，取哈希的规则随之而改变，所以当要Find一个page的时候就有可能会映射到新的位置上，但是原来的桶内在没插满之前是不会进行重新分配的这时该如何保证能找到这个数据呢？


下面用一个例子解释一下这个过程。
![image.png|300](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231206190032.png)

假设现在取低 1 位 也就是global=1，假设现在来了一个101需要插入而刚好满了需要进行扩容操作。
![image.png|325](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231206204244.png)


这里也给了上边那个小坑的答案，就是在扩展哈希表的时候后面新扩展出来的仍然指向原来的桶。

插入101之后原来 1 对应的桶被重新分配 分别对应01 和 11而原来的 0 对应的桶因为还没有需要插入的数据所以不进行重新分配。当需要的时候再进行重新的分配。

这样就保证了扩展是线性的，不会每一次扩展的时候负载激增。 可以理解为“懒扩展”。



### 具体代码实现

因为教授的要求不能放出全部的源代码所以我只给出关键部分

```c++
template <typename K, typename V>
auto ExtendibleHashTable<K, V>::RedistributeBucket(std::shared_ptr<Bucket> bucket) -> void {
  bucket->IncrementDepth();
  size_t deepth = bucket->GetDepth();
  num_buckets_++;
  std::shared_ptr<Bucket> new_bucket = std::make_shared<Bucket>(bucket_size_, deepth);
  size_t pre_hash_sign = std::hash<K>()((bucket->GetItems().begin())->first) & ((1 << (deepth - 1)) - 1);
  // 遍历原来的bucket进行符合要求就是新老哈希值不一样的就放进新桶里
  for (auto i = bucket->GetItems().begin(); i != bucket->GetItems().end();) {
    size_t temp_hash_sign = std::hash<K>()(i->first) & ((1 << (deepth)) - 1);
    if (pre_hash_sign != temp_hash_sign) {
      new_bucket->Insert(i->first, i->second);
      bucket->GetItems().erase(i++);
    } else {
      i++;
    }
  }
  // 就该把新桶放在后边那新位置
  // size_t new_hash_sign= std::hash<K>()((new_bucket->GetItems().begin())->first) & ((1 << (deepth)) -1 );
  for (size_t i = 0; i < dir_.size(); i++) {
    if ((i & ((1 << (deepth)) - 1)) != pre_hash_sign && (i & ((1 << (deepth - 1)) - 1)) == pre_hash_sign) {
      dir_[i] = new_bucket;
    }
  }
}
template <typename K, typename V>
void ExtendibleHashTable<K, V>::Insert(const K &key, const V &value) {
  // UNREACHABLE("not implemented");
  std::scoped_lock<std::mutex> lock(latch_);
  while (true) {
    size_t index = IndexOf(key);
    if (dir_[index]->Insert(key, value)) {
      break;
    }
    // 判断当前bucket的depth和全局depth 如果相同就进行哈希表的扩容操作，如果不相同就进行桶的重新分配
    // 这样做的一个好处就是时间复杂度是线性的，不会遇到桶满的时候就负载激增，导致效率下降。
    if (GetLocalDepthInternal(index) != GetGlobalDepthInternal()) {
      RedistributeBucket(dir_[index]);
    } else {
      global_depth_++;
      size_t size = dir_.size();
       // 这里就是在扩展后将指针指向对应的老桶就是扩展前的桶。
      for (size_t i = 0; i < size; i++) {
        dir_.emplace_back(dir_[i]);
      }
    }
  }
}
```



##  Task #2 LRU-K
### LRU-K Replacer Design

LRU-K Replacer 用于存储 buffer pool 中 page 被引用的记录，并根据引用记录来选出在 buffer pool 满时需要被驱逐的 page。

LRU 应该都比较熟悉了，LRU-K 则是一个小小的变种,出现的目的是为了在一定程度上解决**缓存污染**的问题，有兴趣的可以读一下有关论文，有很详细的介绍。
[cs.cmu.edu/\~natassa/courses/15-721/papers/p297-o\_neil.pdf](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p297-o_neil.pdf)

在普通的 LRU 中，我们仅需记录 page 最近一次被引用的时间，在驱逐时，选择最近一次引用时间最早的 page。

在 LRU-K 中，我们需要记录 page 最近 K 次被引用的时间。假如 list 中所有 page 都被引用了大于等于 K 次，则比较最近第 K 次被引用的时间，驱逐最早的。假如 list 中存在引用次数少于 K 次的 page，则将这些 page 挑选出来，用普通的 LRU 来比较这些 page 第一次被引用的时间，驱逐最早的。

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/image-20230322231114-gk3tsn9.png)
所有的page大概能够分为两类：
1. 达到k次的，那么则选择distance最大的进行淘汰
2. 没有达到k次的，就根据普通的LRU进行淘汰

具体实现不是很难，按照给的注释一步步写就好了
刚开始我建议先去[力扣（LeetCode）官网 - 全球极客挚爱的技术成长平台](https://leetcode.cn/problems/lru-cache/description/)写一下这个LRU，实现的思路不能说差不多也能说是差不多。

对于数据结构的组织我推荐2023fall封装好的这个，我自己做的是2022fall没有这个封装，需要自己设计，给出了一个推荐的方案，对于一个项目的起步还是非常友好的。


```c++
class LRUKNode {
private:
std::list<size_t> history_;
size_t k_;
frame_id_t fid_;
bool is_evictable_{false};
};
std::unordered_map<frame_id_t, LRUKNode> node_store_;
```


>只有一个需要注意的点：在`SetEvictable` 这个函数中只有发生改变才修改curr_size_（可驱逐的个数）这个变量。

## Task# 3-Buffer Pool Manager 

这个我认为最主要的就是要厘清各个函数之间的关系

```c++
/** Array of buffer pool pages. */
Page *pages_;
/** Page table for keeping track of buffer pool pages. */
 ExtendibleHashTable<page_id_t, frame_id_t> *page_table_;
/** Replacer to find unpinned pages for replacement. */
LRUKReplacer *replacer_;
/** List of free frames that don't have any pages on them. */
std::list<frame_id_t> free_list_;

```

上述的定义要和初始化函数结合在一起看
```c++
// we allocate a consecutive memory space for the buffer pool
  pages_ = new Page[pool_size_];
  page_table_ = new ExtendibleHashTable<page_id_t, frame_id_t>(bucket_size_);
  replacer_ = new LRUKReplacer(pool_size, replacer_k);
  // Initially, every page is in the free list.

  for (size_t i = 0; i < pool_size_; ++i) {
    free_list_.emplace_back(static_cast<int>(i));
  }
 
```

结合注释不难发现他们之间的关系
- `pages_:`Array of buffer pool pages 。buffer pool的一个数组。他的下标就是frame_id。
- `page_table_`:记录page和frame之间的映射关系。
- `replacer_`: 记录访问历史，在free_list中没有空闲frame可以承载page的时候进行驱逐。
- `free_list`:表示有无空闲的frame可供使用


![image.png|550](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231206222714.png)
>上图**引用自**：[BusTub Lab1 Buffer Pool Manager | Fischer.space](https://itfischer.space/2023/03/20/bustub/Bustub-Lab1/)

这个图很好的解释了三者之间的关系以及各部分所负责的东西。

接下来梳理一下task3中比较重要的部分
###  NewPgImp()和FetchPgImp()

两个函数有很大一部分都是相同的
重点也是这相同的一部分

先来看**NewPgImp()** 的要求：
>Create a new page in the buffer pool. Set page_id to the new page's id, or nullptr if all frames are currently in use and not evictable (in another word, pinned)

就是新建一个页，但是新建页的过程非常的有讲究：
1. 首先需要找到一个空闲的frame_承载这个page，优先从free_list中找frame
2. 如果free_list中不存在的话，就需要在replacer_中用驱逐函数将一个frame上的page数据清空，清空在table里的映射关系。用来承载新建的page。
3. 在驱逐的时候如果原来的页是脏的要先吧脏页写回磁盘，防止数据的丢失

之后就是页的初始化操作，注意要顾及到**每一个**变量。
将新的映射关系写入table，新的访问历史放入replacer

**FetchPgImp()** 的流程大致相同：

只是第一步先去table里找有无这个page存在，如果没有就类似于**NewPgImp()** 先找一个frame，然后通过disk_manager_将磁盘的数据读入新页。之后建立新的关系。


### UnpinPgImp（page_id_t page_id, bool is_dirty）

当上层有一个进程在读取这个页的时候这个页都要被pin住不能被淘汰，在使用结束的时候会对该页进行一次unpin的操作，这会使得pin_count_--，当这个数值为0的时候该页就可被`SetEvictable()`函数将其在replacer中标记为可淘汰。

随着page_id一同传进来的还有一个bool变量`is_dirty`这个变量用来表示使用这个页的进程对这个页干了什么，如果仅仅只是读的话`is_dirty`的值就为false，但是进行的改动的话`is_dirty`的值就为true。
需要注意的一定，只有当原来这个页是**干净的时候**才能进行状态的修改。


## Summary && 踩坑记录

1. 在task1的时候没有搞懂IndexOf（）函数的作用是什么，看到进行位操作本能的进行回避，导致被卡了好长时间，毫无头绪，在写的时候还是要搞懂细节，多看官方的注释。
2. SetEvictable（）这个函数中明白curr_size_什么时候能改，什么时候不能改。
3. 在FlushAllPgsImp()中复用了FlushPgImp()函数，导致在线评测虽然样例都过了但是一直超时，最后还是在[BusTub Lab1 Buffer Pool Manager | Fischer.space](https://itfischer.space/2023/03/20/bustub/Bustub-Lab1/)这篇文章中得到启发。总的来说还是多线程编程学的不是很好。

以上三个都是卡我时间比较长而且处理起来毫无头绪的。不过幸好项目用cmake构建，debug起来算是比较方便，帮我节约了很多的时间。

我认为比较有挑战的就是task1，大的框架建好之后这种修修补补的编程很有意思。相比于有些课程的全面铺开完全从0开始实现，这种直入主题，学习核心思想的过程让我受益匪浅。



最后附上通过截图，rank比较靠后就不放了，因为全局一把大锁，能跑下来我已经很开心了。


![image.png](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/20231205150555.png)