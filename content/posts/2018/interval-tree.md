---
title: 判断多个区间是否有重叠
description: ""
date: 2018-10-10 18:26:25
tags: ["算法"]
menu: "main" 
---
&nbsp;&nbsp;&nbsp;&nbsp;今天遇到个需求，大致意思是前端给我一堆商品按照阶梯定价的规则，比如订单数为0-200，价格是10块，订单数200-300，价格是8块……简化的数据结构是这样的：  

```
[
  {
    "min_orders": 0,
    "max_orders": 200,
    "price": 10
  },
  {
    "min_orders": 200,
    "max_orders": 300,
    "price": 8
  },
  {
    "min_orders": 400,
    "max_orders": 500,
    "price": 7
  }
]
```
&nbsp;&nbsp;&nbsp;&nbsp;我需要对前端传的数据做校验，其中有一部分就是不同阶的上下界不能有交叉（前端给的阶梯数据不一定有序），比如有0-200的阶梯，如果再有100-300的阶梯那这个数据就有问题，因为没法定价。抽象出来问题其实就是判断多个区间（可能不连续）是否有重叠。  
&nbsp;&nbsp;&nbsp;&nbsp;初一看，这个问题其实很简单，两个for循环，第一层记住遍历过的区间，第二层把当前区间和之前遍历的区间一个个比较看是否交叉就可以了。这么做当然没问题，但是时间复杂度是O(n^2)，怎么看都觉得应该有更好的解决办法，第一遍的遍历是怎么都省不了的，所以优化到极致也不可能比O(n)低，第二遍其实就是个比较的过程，既然是比较的话，那就有思路了，可以联想到很多类似的问题，本质上和排序问题是差不多的。  
&nbsp;&nbsp;&nbsp;&nbsp;在stackoverflow看到一个相同的问题：[search for interval overlap in list of intervals?](https://stackoverflow.com/questions/4446112/search-for-interval-overlap-in-list-of-intervals)。里面提到了一个办法：可以先对区间以左端点进行排序，然后再遍历区间，比较当前区间的左端点和上一个区间的右端点，如果当前的左端点比上一个右端点小，那么一定就有重叠的部分。第一遍排序时间复杂度是O(nlogn)，第二遍遍历时间复杂度是O(n)，所以总体的还是O(nlogn)，比如：  

```
[(4,6), (7,9), (1,5), (6,7)]
```

&nbsp;&nbsp;&nbsp;&nbsp;排序完之后是：  

```
[(1,5), (4,6), (6,7), (7,9)]
```
&nbsp;&nbsp;&nbsp;&nbsp;然后进行遍历比较，遍历到第二个区间的时候，左端点是4，比上个区间的右端点5要小，所以肯定有重叠的部分。

&nbsp;&nbsp;&nbsp;&nbsp;**第二种办法是构建区间树**，上面说的效率最低的那种解决办法，第二层for循环是比较，既然是比较，那么就可以通过树这种结构来提高比较的效率，而不用和前面所有区间进行比较，所以很自然就想到可以构建一个二叉排序树，树的叶节点是每个区间。同时，树的查询效率和树的深度是有关系的，所以构建一个相对平衡的二叉排序树也能进一步提高效率，比如红黑树。算法导论上介绍了这种数据结构，**节点是区间的红黑树——区间树**。  
&nbsp;&nbsp;&nbsp;&nbsp;红黑树实现起来还是很麻烦的，找了一圈发现github上有现成的轮子——[intervaltree](https://github.com/chaimleib/intervaltree)，使用非常简单：  

```
>>> from intervaltree import Interval, IntervalTree

>>> tree = IntervalTree()
>>> iv = Interval(4, 7)
>>> tree.add(iv)

>>> target_iv = Interval(5, 8)
>>> tree.overlaps(target_iv)
True
>>> target_iv_2 = Interval(7, 10)
>>> tree.overlaps(target_iv_2)
False

```

&nbsp;&nbsp;&nbsp;&nbsp; 更多用法可以参考[intervaltree · PyPI](https://pypi.org/project/intervaltree/)