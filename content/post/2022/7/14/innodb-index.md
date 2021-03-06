---
title: "Innodb Index"
date: 2022-07-14T11:54:52+08:00
draft: false
---
之前的 [简单索引方案](/post/2022/7/13/index) 假设所有的目录项都可以在物理存储器上连续存储，这样会有几个问题：
 * InnoDB 是使用页来作为管理存储空间的基本单位，也就是最多能保证 16KB 的连续存储空间，而随着表中记
   录数量的增多，需要非常大的连续的存储空间才能把所有的目录项都放下，这对记录数量非常多的表是不现
   实的。
 * 我们时常会对记录进行增删，假设我们把 _页28_ 中的记录都删除了，_页28_ 也就没有存在的必要了，那意味
 着 _目录项2_ 也就没有存在的必要了，这就需要把 _目录项2_ 后的目录项都向前移动一下，这种牵一发而动全身
  

### InnoDB 中的索引方案

![innodb](https://pic.rmb.bdstatic.com/bjh/5824062564d8413a913d41707c7a11c8.png)

 * _目录项记录_ 的 _record_type_ 值是1， 而普通用户记录的 _record_type_ 值是0
 * _目录项记录_ 只有主键值和页的编号两个列， 而普通的用户记录的列是用户自己定义的，可能包含多很多列，另外还有 _InnoDB_ 自己添加的隐藏列
 * 只有在存储 目录项记录 的页中的主键值最小的 目录项记录 的 min_rec_mask 值为 1 ，其他别的记录的 min_rec_mask 值都是 0

虽然说 _目录项记录_ 中只存储主键值和对应的页号， 比 _用户记录_ 需要的存储空间小多了，但是不论怎么说一个页的大小只有 _16KB_ ， 能存放的
_目录项记录_ 也是有限的。，那如果表中的数据太多，以至于一个数据页不足以存放所有的 _目录项记录_ 会新分配一个 _目录项记录_ 的页。

![b+tree](https://pic.rmb.bdstatic.com/bjh/d677d1f254fa7b12aac660a539dfbe6b.png)

如图，我们生成了一个存储更高级目录项的 _页33_ ，这个页中的两条记录分别代表 _页30_ 和 _页32_ ，如果 _用户记录_
的主键值在 _(1, 320)_ 之间，则到 _页30_ 中查找更详细的 _目录项记录_ ，如果主键值不小于 _320_ 的话，就到 _页32_ 中查找更详细的 _目录项记录_ 。


不论是存放用户记录的数据页，还是存放目录项记录的数据页，我们都把它们存放到 [_B+树_](https://blog.csdn.net/weixin_35794878/article/details/122609218) 
数据结构中，所以我们也称这些数据页为 _节点_ 。从图中可以看出来，我们的实际用户记录其实都存放在B+树的最底层的节点上，这些节点也被称为 _叶子节点_ 或 _叶节点_ ，
其余用来存放 _目录项_ 的节点称为 _非叶子节点_ 或者 _内节点_ ，其中 B+ 树 最上边的节点也称为 _根节点_ 。

一个 B+Tree 的节点可以分成很多层， InnoDB规定存放 _用户记录_ 的那层为 _第0层_ ,之后依次往上加。


#### 聚簇索引 Clustered Index
B+Tree 本身就是一个目录，或者说本身就是一个索引。其特点为：

 * 使用 _记录_ 主键值的大小进行记录和页的排序
    1. 页内的记录是按照主键值的大小排成一个单项链表。
    2. 各个存放 _用户记录_ 的页也是根据页中用户记录的主键值大小顺序排成一个 [双向链表](https://blog.csdn.net/weixin_48524215/article/details/119103566)
    3. 存放目录项记录的页份分为不同的层次，在 _同一层次中_ 的页也是根据页中目录项记录的主键大小顺序排成一个双向链表。
 * B+Tree 的叶子节点存储的是完整的用户记录。（存储了用户记录的所有的列的值，包含隐藏列）
 
 
#### 二级索引 Secondary Index
聚簇索引只能在搜索条件是主键值时才能发挥作用，因为 B+Tree 中的数据都是按照主键进行排序的。那如果我们想以别的列作为搜索条件该咋办呢？难道应该从头到尾
沿着链表依次遍历记录吗？

no~,只需要多建几棵 B+Tree , 不同的 B+Tree 中的数据采用不同的排序规则。例如我们将 _用户记录_ c2 列（int）的值大小作为数据页、页中记录的排序规则建立一颗 B+Tree (c1 为 primary key):
![secondary_index](https://pic.rmb.bdstatic.com/bjh/222732204099b27931ff83337075e4c6.png)

这个 B+Tree 与上边介绍的聚簇索引有基础不同：
 * 根据记录 c2 列的大小进行据记录和页的排序
   1. 页内的记录是按照 c2 列的大小顺序排成一个单向链表
   2. 各个存放用户记录的页也是根据页中记录的 c2 列大小顺序排成一个双向链表
   3. 存放 _目录项记录_ 的页分为不用的层次，在同一层次中的页中 _目录项_ 记录的 c2 列大小排成一个双向链表
 * B+Tree 的叶子节点存储的并不是完整的用户记录，而是存储了 c2 列 & 主键列 这两个列的值
 * _目录项记录_ 中不再是 主键+页号 的搭配，而变成了 c2列+页号 的搭配
 
所以如果我们现在想通过 c2 列的值查找某些记录的话就可以使用我们刚建立的 B+Tree 了。

但是这个 B+Tree 的叶子节点中的记录值存储了 _c2_ 和 _primary key_ 两个列,
>所以如果 _查询的列不在叶子节点中_ 我们必须在根据主键值去 _聚簇索引_ 中再查找一遍获取 _完整的用户记录_ （[回表](https://wenku.baidu.com/view/bd311b7974232f60ddccda38376baf1ffc4fe3d0.html)）。

##### 联合索引

我们页可以同时使用多个列的大小作为排序规则，也就是同时为多个列建立索引。
联合索引其本质上也是一个 _二级索引_ ,且只会建立 1 棵 B+Tree


#### InnoDB B+Tree 索引注意事项
    一个 B+Tree 索引的根节点自诞生之时起，便不会再移动。
* 每当为一个表创建一个 B+Tree 索引（非聚簇索引，聚簇索引是默认有的）时，都会为这个索引创建一个 _根节点_ 页面。
    最开始表中没有数据的时候，每个 B+Tree 索引对应的 _根节点_ 中既没有 _用户记录_ ，也没有 _目录项记录_ 。
* 随后向表中插入用户记录时，先把用户记录存储到这个 _根节点_ 中。
* 当 _根结点_ 中的可用空间用完时继续插入记录，此时会将 _根结点_ 中的所有记录复制到一个新分配的页，如 _页a_ 中，然后对这个
    新页进行 [页分裂](https://www.cnblogs.com/ZhuChangwu/p/14041410.html) 操作，得到一个新页 如 _页b_ 。
    这时新插入的记录根据键值 的大小就会被分配到 _页a_ 或者 _页b_ 中,而 _根节点_ 就被升级为存储目录项记录的页。

####
    内节点中目录项记录的唯一性
为了让新插入记录能找到自己在哪个页中，我们需要保证在 B+Tree 的同一层内节点的目录项记录除 _页号_ 这个字段意外是唯一的。
所以对于二级索引的内节点的目录项记录的内容实际上是由三个部分构成的：
* 索引列的值
* 主键值
* 页号

也就是我们把 _主键值_ 也添加到二级索引内节点中的目录项记录了，这样就能保证 B+Tree 每一层节点中各条目录项记录除
 _页号_ 这个字段外是唯一的。

####
    一个页面最少存储2条记录
    



