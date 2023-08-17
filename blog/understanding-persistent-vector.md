---
title: understanding-persistent-vector
date: 2016-10-18 16:39:59
category: 语言
tags: [clojure]
---

本文翻译自以下系列文章：
* [understanding-persistent-vector-pt-1](http://hypirion.com/musings/understanding-persistent-vector-pt-1)
* [understanding-persistent-vector-pt-2](http://hypirion.com/musings/understanding-persistent-vector-pt-2)
* [understanding-persistent-vector-pt-3](http://hypirion.com/musings/understanding-persistent-vector-pt-3)
* [understanding-clojure-transients](http://hypirion.com/musings/understanding-clojure-transients)
* [persistent-vector-performance-summarised](http://hypirion.com/musings/persistent-vector-performance-summarised)
* [persistent-vector-performance](http://hypirion.com/musings/persistent-vector-performance)

作者是leiningen的维护者, [blog](http://hypirion.com/)

我把这几篇文章合并到一起了，如果想要更清楚的描述，请直接阅读以上英文文档。

## Part 1 ##

听过或者没有，clojure的persistent vector就在那里，Rich为clojure开发了这个数据结构（灵感来自Phil Bagwell的论文[Ideal Hash Tree]()），使得append、update、lookup和subvec的时间复杂度为O(1)。因为所有的vector都是持久化的，所有对vector的修改操作都不会改变原来结构，而是生成一个全新的vector。


听起来非常不可思议，但这怎么能做到呢？我会在本文详细介绍它的实现，以及实现过程中的一些技巧。

首先，我们看一下3个基础操作：update、append、popping。这3个操作是Clojure Persistent Vector的内核，稍后，我们再看看一些对常用便捷操作上的优化，比如transient和tail。

### 基本思路 ###

C++ STL的vector和Java的ArrayList底层使用最基础的数组实现，在空间不够的时候，会自动分配内存。如果你想要可变性，这就是完美的，但如果你想要持久性，这就行不通。要实现持久性，你就需要频繁的生成新对象，这会导致修改操作变得极慢，且内存消耗速增。怎么才能在查询和修改时避免这种底层数据的拷贝以获得最佳性能？这就是clojure 的persistent vector实现所完成的工作，它的实现实际上是一颗平衡排序树。

我们先看这样一颗类二叉树。和普通二叉树不同是，树的内部结点最多两个子节点，且不含任何值。叶结点也最多包含两个值。所有值都是排好序的，也就是，最左边的叶节点包含序列里最小的值，最右边的叶节点包含序列里最大的值。我们暂时假设所有叶节点的深度相同（可以认为这是路径选择的一种简化，虽然JVM会对路径选择优化起到一些作用，但这个后文再详细讨论）。请看下面这棵树，包含了0~8 9个整数，0是第一个元素，8是最后一个元素，9是这个vector的大小。

![a vector with 9 elements in it](http://hypirion.com/imgs/pvec/9-annotated.png)

如果要在vector尾部插入一个新的元素，在一个可变性主宰的世界里，结果会是这样，把9插入到最右边的叶子节点：

![the same vector, but with 9 inserted at the end](http://hypirion.com/imgs/pvec/10.png)

但问题是：这改变了原来的数据结构！如果想要基于这种实现获得持久性的修改操作，就需要拷贝整颗树（或者至少树的一部分）。

既要保留持久性，又要最小化底层的拷贝操作，我们想到一种方法：路径拷贝，只拷贝从根节点到需要修改（update，insert，replace）的叶节点的整条路径。多次插入的操作演示如下，有7个元素的vector与有10个元素的vector共享了部分数据结构：

![two vectors, with structural share](http://hypirion.com/imgs/pvec/7to10.png)

粉色的节点由两个vector共享，棕色和蓝色的分别属于各vector自己，如果还有其他操作，新生成的vector可能会和这两个vector有更多的节点共享。

### 更新 ###

最简单的修改操作就是更新/替换值，在clojure中，由assoc或者update-in/update实现。

更新一个元素，先沿着树遍历到需要更新的节点，遍历过程就会拷贝遍历过的路径以保证后续的持久性，遍历到节点后，直接拷贝出一个新的节点，然后把值替换掉，然后返回一个新的vector，这个新的vector包含的是拷贝出来的新的路径。

举个栗子。在vector上执行一个assoc操作：

```clojure
(def brown [0 1 2 3 4 5 6 7 8])
(def blue (assoc brown 5 'beef))
```

操作演示如下，蓝色vector是拷贝出来的新路径：

![two vectors, where we're updated a value](http://hypirion.com/imgs/pvec/update.png)

假设我们已经知道如何获取需要遍历的路径（后文会介绍这个部分），这个操作还算简单。

### 追加 ###

追加(Append，即尾部插入)和更新大体相同，但是有一些边界问题需要处理，可能会要求生成新的叶节点，基本上，分下面3中情况：

1. 最右边的叶节点还有空间
2. 根节点有空间，但最右叶节点没有空间
3. 根节点没有空间

下面逐个看一下。

#### 1: 和Assoc操作一致 ####

只要最右叶节点还有空间，其实和assoc更新操作完全一致：拷贝路径，创建新节点，把新的元素放入新节点内。举个栗子：下面的图演示了操作`(conj [0 1 2 3 4] 5)`，蓝色是新生成的节点，棕色是原数据结构：

![append with enough space in the leaf node](http://hypirion.com/imgs/pvec/conj-1.png)

简单！！！

#### 2: 按需生成新节点 ####

如果最右叶节点没有空间呢？没关系，只要能找到正确的路径到达叶节点就ok了，幸运的是我们总能找到这样一条路径，毕竟它是一颗二叉树。但是...会发现这条路径其实还不存在...当一个节点不存在的时候，我们就生成新的节点，把这个新节点当成是拷贝来的节点，后续的操作就和前面一样了：

![append where we generate new node instead of copy](http://hypirion.com/imgs/pvec/conj-2.png)

上图，粉色节点是新生成的，蓝色是真正拷贝来的。

#### 3: 根节点溢出 ####

最后一种情况是根节点溢出，也就是根节点没有空间了（其实就是所有节点的子节点数都是2），不难理解，我们只能在根节点之上再加新节点，这样老根节点是原数据结构的根节点，是新数据结构根节点的子节点，新加的节点就是新数据结构的根节点，后续的操作也和前面一致了。看似有点绕，但如图所示，其实很简单：

![append where we create a new root](http://hypirion.com/imgs/pvec/conj-3.png)

解决这个case还是相对简单，但当你进行append操作的时候，如何鉴别根节点是否有足够空间呢？方法也很简单，因为我们是二叉树，所有当vector的大小正好是2的次幂的时候，原来的vector根节点就溢出了，更通用的是，当我们是一颗n叉树的时候，当vector的大小是n的次幂的时候根节点就溢出了。

###  弹出(Popping) ###

弹出操作(popping, 删除尾部元素)也比较简单，它和追加操作比较相似：

1. 最右叶节点包含不止个元素
2. 最右叶节点包含一个元素
3. 执行popping操作后，根节点只有一个元素

本质上，这就是append操作3中情况的逆操作，都很简单。

#### 1: dissoc to the rescue ####

只要最右叶节点在执行popping操作后还有至少一个元素，我们就直接拷贝路径和节点，完成操作：

![popping a value from vector with more than 1 element in the rightmost leaf node](http://hypirion.com/imgs/pvec/pop-1.png)

记住，在一个vector上多次执行popping操作不会生成相同的vector，他们包含的元素相同，但他们不共享跟节点，举例：

```clojure
(def brown [0 1 2 3 4 5])
(def blue (pop brown))
(def green (pop brown))
```

![performing pops on the same vector](http://hypirion.com/imgs/pvec/pop-twice.png)

#### 2: 删除空节点  ####

如果执行完popping操作后，最右叶节点为空，那这个节点需要被回收，父节点不再指向这个节点，应该置空（空指针）。

![popping and removing a leaf node](http://hypirion.com/imgs/pvec/pop-2.png)

注意，这次棕色的是pop前的vector，蓝色是pop后新生成的vector。

看起来仍然很简单，但其实有一个问题，假如，把空指针返回给一个节点，而这个节点原本就只有一个节点，也就是这个空指针节点，就必须把这个（父）节点也变成空指针，继续向上返回。所以，把一个节点置空的操作结果需要向上传递，实现起来可能会比较tricky，需要检查这个子节点是否为空，且子节点在父节点中的索引为0，如果是就继续向上返回空。

clojure的递归实现如下：

```clojure
(defn node-pop [idx depth cur-node]
  (let [sub-idx (calculate-subindex idx depth)]
    (if (leaf-node? depth)
      (if (== sub-idx 0)
        nil
        (copy-and-remove cur-node sub-idx))
      ; not leaf node
      (let [child (node-pop idx (- depth 1)
                            (child-at cur-node sub-idx))]
        (if (nil? child)
          (if (== sub-idx 0)
            nil
            (copy-and-remove cur-node sub-idx))
          (copy-and-replace cur-node sub-idx child))))))
```

以上实现就可以保证移除空节点不会带来问题，下面的例子演示了新生成的蓝色的vector实际上删除了两个节点：叶子节点c和它的父节点：

![popping and removing mutiple nodes](http://hypirion.com/imgs/pvec/pop-2-recur.png)

#### 3: Root Killing ####

还剩最后一种情况，依照上面的方法，以下棕色的树在移除8这个节点后会得到蓝色的树，如图：

![popping with bad root handling](http://hypirion.com/imgs/pvec/pop-3-bad.png)

没错，就是根节点只有一个子节点。这种树没有任何意义，因为查找的时候一定会直接找到它的子节点。所以需要把多余的根节点去掉。

这可能是目前为止最简单的操作了，popping完成，检查根节点是否只有一个子节点（检查第2个节点是否为空），如果只有一个子节点，且根节点不是叶子节点，我们就直接用它的子节点替换这个跟节点，相当于两个节点做了合并。结果如下，根节点下降了一层：

![popping with proper root handling](http://hypirion.com/imgs/pvec/pop-3.png)

### O(1) != O(log n) ###

看到这里可能有人会问，这哪是O(1)时间复杂度？事实上，如果每个节点只有连个子节点，时间复杂度是O(log2 n)，远大于O(1)。

技巧就是，前面没有规定我们只能有2个子节点，只是为了解释方便选择了2个子节点作为假设，事实上，clojure的实现有32个子节点，这样，树的深度就会非常小，可以算一下，一个vector中有不到10亿个元素时，树的深度最多6层，需要350亿个元素才能把树的深度增加到8层。在这种情况下，内存消耗可能是一个更亟待考虑的严重问题。

举个实例，一个4叉树，包含14个元素，树的深度只有2，再看上面移除空节点的例子，2个vector各有13和12个元素，在2叉树的情况下，深度已经是4，等于下面这颗树的2倍。

![q 4-way branching vector](http://hypirion.com/imgs/pvec/4-way-branching.png)

在这种精心设计的层数极低的树结构下，我们倾向于认为所有的修改操作都接近于常数时间，虽然理论值等于O(log32 N)。对于理解时间复杂度的人会认为O(log32 N)和O(log N)是一样的，但为了便于市场推广，大家都喜欢说成是常数时间。

### 小节 ###

以上主要描述了clojure的persistent vector内部如何实现，其update、append、popping操作的基本思路，以及如何在常数时间内完成这些操作，有关append和popping的加速优化会在文末介绍，接下来先讨论一些更重要的...

## Part 2 ##

省略承前启后的内容...本小节会讨论查找操作的内部实现，即路径选择，如何找到需要修改的节点。

要理解路径选择的实现，我们先给这个数据结构命名，并且解释为什么会如此命名。听起来有点诡异，通过对数据结构进行命名来解释原理，但这个名字确实能准确描述我们的实现。

### 命名 ###

clojure的persistent vector更正式的名字是(persistent bit-partitioned vector trie)持久化的基于位图分区的向量trie树（）。上一个小节其实介绍的是(persistent digit-partitioned vector trie)持久化的基于数字分区的向量trie树，位图分区只是在数字分区上的优化，基本上没有什么区别，只是性能得到了提升。

估计，大部分人看不懂上面这一段内容，接下来就会深入聊聊。

### 持久化 ###

在part 1，我们已经使用了持久化这个词，说我们希望持久化，但并没有解释持久化本身意味着什么。

持久化的数据结构不会修改它自身：更准确地说，它的内部是可变的，只需要从使用者的角度它没有发生任何变化。对持久化数据结构的实例进行更新、插入和删除，你会得到一个新的实例，旧的数据永远保持不变，无论对它进行多少次操作，它的返回结果都是保持不变的。

当我们讨论一个完全持久化数据结构的时候，这个数据的所有版本都应该是课更新的，意味着，对一个版本的所有操作都可以加之与另外一个版本。在早期的函数式数据结构时代，有一些作弊行为，旧的数据版本内部会发生变化，在这个版本上的操作会越来越慢。但，Rich决定让clojure持久化数据结构的所有版本都保证相同的性能。

### Vector ###

vector是一维可变数组，C++的std::vector和Java的java.util.ArrayList就是可变vector实现的例子，这上面没有技巧，一个vector trie树就是一个表示vector的trie树，它不一定是持久化的，只是在clojure里，它恰好是持久化的。

### Trie ###

Trie是一种特殊类型的树，要解释它，我们可以先看一些比较常见的树结构。

红黑树和其他二叉树，元素或hash表对都保存在树的内部结点中，选择正确的路径通常意味着元素或者key比较，如果元素的值比结点值小，就选左边，否则选右边，叶子节点通常是空指针，不包含任何内容。

![a rb-tree, from wikipedia artical](http://hypirion.com/imgs/pvec/rb-tree.png)
"Example red-black tree" by Cburnett, CC-BY-SA 3.0

上面的红黑树摘自维基百科的文章，我在这里不做详细介绍，只简单解释如何检查22是否包含在这颗树种：

1. 从根节点13开始，与22比较，22>13，选右边
2. 与17比较，22>17，选右边
3. 与25比较，22<25，选左边
4. 这个节点就是22，找到

如果想更深入了解红黑树，推荐Julienne Walker的[红黑树教程](http://www.eternallyconfuzzled.com/tuts/datastructures/jsw_tut_rbtree.aspx)

trie树，完全不同，所有值都存储在叶子节点中（）。选择正确的路径，需要使用部分key去查找，因此，trie树通常不止两个子节点，对clojure来说，会有32个子节点。

![trie](http://hypirion.com/imgs/pvec/trie.png)

一般意义上的trie树如上图，对这个特定的trie树，它工作原理类似于一个map，接收长度为2的字符串，如果代表该字符串数字在这颗树种存在，就返回这个数字。比如，ac的值是7，ba是8。以下是工作原理：

对字符串，我们把字符串分隔成单个字符，从第一个字符开始，如果找到一个边的值等于它，就用第2个字符继续向下查找，如果找不到就停止搜索，整个过程递归进行，当最终结束搜索的时候，就知道是否找到了这个字符串所代表的数字。

举例，我们查找ac。

1. 把ac分隔成[a, c]，从根节点开始
2. 检查这个节点是否有边的值等于a,找到，继续向下查找
3. 检查是否有边的值等于c，找到，继续向下
4. 字符搜索完毕，把当前节点的值返回，也就是7

clojure的persistent vector就是一颗trie树，元素的索引是key，但是，你也许已经猜到，我们必须分裂索引数，索引本身是整数，如何分裂整数，要么就是以数字进行分区，要么就是按位。

### 按数字分区 ###

按数字进行分区就是把key分割成一个个数字，以此为基础构造trie树，举例，把9128分割成[9, 1, 2, 8]，把这个数字代表的元素存入trie树，如果trie树的深度大于list的长度，我们可能要在数字前面补0以对齐。

当然，我们也可以基于非10进制的数字进行分区，比如，9128的7进制数为35420，因此分割成数组[3, 5, 4, 2, 0]，然后用这个数组在trie树种进行查找和插入操作。

![visualization of 35420 lookup](http://hypirion.com/imgs/pvec/35420.png)

这颗trie树（旋转了一下，去掉不需要遍历的路径，方便查看）展示了我们如何遍历一个按数字分区的trie树，先选3，然后5，一直到所有数字都选完，走到最后一个分支，也就是最右边数组的第一个元素，就是我们要查找的元素。

如果知道如何查找数字，实现这个过程并不难，以下是忽略掉所有与查找无关因素的Java代码:

```java
public class DigitTrie {
  public static final int RADIX = 7;

  // Array of objects. Can itself contain an array of objects.
  Object[] root;
  // The maximal size/length of a child node (1 if leaf node)
  int rDepth; // equivalent to RADIX ** (depth - 1)

  public Object lookup(int key) {
    Object[] node = this.root;

    // perform branching on internal nodes here
    for (int size = this.rDepth; size > 1; size /= RADIX) {
      node = (Object[]) node[(key / size) % RADIX];
      // If node may not exist, check if it is null here
    }

    // Last element is the value we want to lookup, return it.
    return node[key % RADIX];
  }  
}
```
rDepth代表根节点下子节点的最大


