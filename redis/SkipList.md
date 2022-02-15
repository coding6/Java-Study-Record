# SkipList
## overview
跳跃表（skiplist）是一种随机化的数据结构，由 William Pugh 在论文《Skip lists: a probabilistic alternative to balanced trees》中提出，是一种可以与平衡树媲美的层次化链表结构——查找、删除、添加等操作都可以在对数期望时间下完成

有序列表 zset 的数据结构，它类似于 Java 中的 SortedSet 和 HashMap 的结合体，一方面它是一个 set 保证了内部 value 的唯一性，另一方面又可以给每个 value 赋予一个排序的权重值 score，来达到 排序 的目的。

它的内部实现就依赖了一种叫做 「跳跃列表」 的数据结构。

## zset为什么使用跳表
首先，因为 zset 要支持随机的插入和删除，所以它 不宜使用数组来实现，关于排序问题，我们也很容易就想到 红黑树/ 平衡树/HashMap 这样的树形结构，为什么 Redis 不使用这样一些结构呢？

性能考虑： 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部，且空间复杂度上skiplist更胜一筹
实现考虑： 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；
key-value无序，不支持范围查找
基于以上的一些考虑，Redis 基于 William Pugh 的论文做出一些改进后采用了 跳跃表 这样的结构。

![](/asset/普通链表.jpg)
本质上，解决的就是查找问题



我们需要这个链表按照 score 值进行排序，这也就意味着，当我们需要添加新的元素时，我们需要定位到插入点，这样才可以继续保证链表是有序的，通常我们会使用 二分查找法，但二分查找是有序数组的，链表没办法进行位置定位，我们除了遍历整个找到第一个比给定数据大的节点为止（时间复杂度 O(n)) 似乎没有更好的办法。

但假如我们每相邻两个节点之间就增加一个指针，让指针指向下一个节点，如下图：
![](/asset/双层跳表.png)


这样所有新增的指针连成了一个新的链表，但它包含的数据却只有原来的一半 （图中的为 3，11）。

现在假设我们想要查找数据时，可以根据这条新的链表查找，如果碰到比待查找数据大的节点时，再回到原来的链表中进行查找，比如，我们想要查找 7，查找的路径则是沿着下图中标注出的红色指针所指向的方向进行的：
![](/asset/双层跳表搜索步骤.png)


这是一个略微极端的例子，但我们仍然可以看到，通过新增加的指针查找，我们不再需要与链表上的每一个节点逐一进行比较，这样改进之后需要比较的节点数大概只有原来的一半。

利用同样的方式，我们可以在新产生的链表上，继续为每两个相邻的节点增加一个指针，从而产生第三层链表：
![](/asset/三层跳表.png)


在这个新的三层链表结构中，我们试着 查找 13，那么沿着最上层链表首先比较的是 11，发现 11 比 13 小，于是我们就知道只需要到 11 后面继续查找，从而一下子跳过了 11 前面的所有节点。

可以想象，当链表足够长，这样的多层链表结构可以帮助我们跳过很多下层节点，从而加快查找的效率。

## 升级跳表
跳跃表 skiplist 就是受到这种多层链表结构的启发而设计出来的。按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到 O(logn)。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点 （也包括新插入的节点） 重新进行调整，这会让时间复杂度重新蜕化成 O(n)。删除数据也有同样的问题。

skiplist 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 为每个节点随机出一个层数(level)。比如，一个节点随机出的层数是 3，那么就把它链入到第 1 层到第 3 层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个 skiplist 的过程：
![](/asset/跳表添加节点过程.png)


对于每一个新插入的节点，都需要调用一个随机算法给它分配一个合理的层数，源码在 t_zset.c/zslRandomLevel(void) 中被定义：

```c
randomLevel()
    level := 1
    // random()返回一个[0...1)的随机数
    while random() < p and level < MaxLevel do
        level := level + 1
    return level
```
zset中p取1/4，MaxLevel取32

空间复杂度分析：

节点层数至少为1。而大于1的节点层数，满足一个概率分布
* 节点层数恰好等于1的概率为1-p
* 节点层数大于等于2的概率为p，而节点层数恰好等于2的概率为p(1-p)
* 节点层数大于等于3的概率为p2，而节点层数恰好等于3的概率为p2(1-p)
* 节点层数大于等于4的概率为p3，而节点层数恰好等于4的概率为p3(1-p)

平均节点数目：1/(1-p)，空间复杂度上优于平衡树。时间复杂度粗略约为logn级别

## 跳表的实现
### 节点实现
```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;   // 存储字符串类型数据
    double score;   // 存储排序的分值
    struct zskiplistNode *backward;   //后退指针，指向当前节点最底层的前一个节点，头节点指向NULL，从后向前遍历跳跃表使用
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 指向当前节点的下一个节点，最后一个节点指向NULL 
        unsigned long span;  //forward 指向的节点与当前节点之间的元素个数。  值越大，跳过的节点个数越多。
    } level[]; // 柔性数组。每个节点的数组长度不同，生成跳跃表节点时，随机生成1~64 的值，值越大出现的概率越低
} zskiplistNode;
```
### 插入一个节点
```c
// 逐步降级寻找目标节点，得到 "搜索路径"
for (i = zsl->level-1; i >= 0; i--) {
    /* store rank that is crossed to reach the insert position */
    rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
    // 如果 score 相等，还需要比较 value 值
    while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) < 0)))
    {
        rank[i] += x->level[i].span;
        x = x->level[i].forward;
    }
    // 记录 "搜索路径"
    update[i] = x;

//创建新的节点
/* we assume the element is not already inside, since we allow duplicated
 * scores, reinserting the same element should never happen since the
 * caller of zslInsert() should test in the hash table if the element is
 * already inside or not. */
level = zslRandomLevel();
// 如果随机生成的 level 超过了当前最大 level 需要更新跳跃表的信息
if (level > zsl->level) {
    for (i = zsl->level; i < level; i++) {
        rank[i] = 0;
        update[i] = zsl->header;
        update[i]->level[i].span = zsl->length;
    }
    zsl->level = level;
}
// 创建新节点
x = zslCreateNode(level,score,ele);

```