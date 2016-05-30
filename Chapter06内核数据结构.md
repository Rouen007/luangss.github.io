#内核的数据结构
链表、队列、映射、二叉树
##链表
**链表**是linux内核中最简单，同时也是应用最广泛的数据结构。

内核中定义的是双向环形链表。

内核中关于链表定义的代码位于： include/linux/list.h
其实刚开始只要先了解一个常用的链表操作（追加，删除，遍历）的实现方法，
其他方法基本都是基于这些常用操作的。

链表代码的注意点
在阅读list.h文件之前，有一点必须注意：linux内核中的链表使用方法和一般数据结构中定义的链表是有所不同的。
一般的双向链表一般是如下的结构，
有个单独的头结点(head)
每个节点(node)除了包含必要的数据之外，还有2个指针(pre,next)
pre指针指向前一个节点(node)，next指针指向后一个节点(node)
头结点(head)的pre指针指向链表的最后一个节点
最后一个节点的next指针指向头结点(head)
具体见下图： 
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/6.1.PNG)

传统的链表有个最大的缺点就是不好共通化，因为每个node中的data1，data2等等都是不确定的(无论是个数还是类型)。
linux中的链表巧妙的解决了这个问题，linux的链表不是将用户数据保存在链表节点中，而是将链表节点保存在用户数据中。
linux的链表节点只有2个指针(pre和next)，这样的话，链表的节点将独立于用户数据之外，便于实现链表的共同操作。
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/6.2.PNG)
linux链表中的最大问题是怎样通过链表的节点来取得用户数据？

和传统的链表不同，linux的链表节点(node)中没有包含用户的用户data1，data2等。

 

整个list.h文件中，我觉得最复杂的代码就是获取用户数据的宏定义

\#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
这个宏没什么特别的，主要是container_of这个宏

\#define container_of(ptr, type, member) ({          \
    const typeof(((type *)0)->member)*__mptr = (ptr);    \
             (type *)((char *)__mptr - offsetof(type, member)); })
这里面的type一般是个结构体，也就是包含用户数据和链表节点的结构体。

ptr是指向type中链表节点的指针
member则是type中定义链表节点是用的名字

struct student
{
    int id;
    char* name;
    struct list_head list;
};
type是struct student
ptr是指向stuct list的指针，也就是指向member类型的指针
member就是 list
下面分析一下container_of宏:

// 步骤1：将数字0强制转型为type*，然后取得其中的member元素
((type *)0)->member  // 相当于((struct student *)0)->list

// 步骤2：定义一个临时变量__mptr，并将其也指向ptr所指向的链表节点
const typeof(((type *)0)->member)*__mptr = (ptr);

// 步骤3：计算member字段距离type中第一个字段的距离，也就是type地址和member地址之间的差
// offset(type, member)也是一个宏，定义如下：
\#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

// 步骤4：将__mptr的地址 - type地址和member地址之间的差
// 其实也就是获取type的地址

步骤1，2，4比较容易理解，下面的图以sturct student为例进行说明步骤3：

首先需要知道 ((TYPE *)0) 表示将地址0转换为 TYPE 类型的地址

由于TYPE的地址是0，所以((TYPE *)0)->MEMBER 也就是 MEMBER的地址和TYPE地址的差，如下图所示：
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/6.3.PNG)

##队列
内核中的队列是以字节形式保存数据的，所以获取数据的时候，需要知道数据的大小。
如果从队列中取得数据时指定的大小不对的话，取得数据会不完整或过大。
内核中关于队列定义的头文件位于：<linux/kfifo.h> include/linux/kfifo.h
头文件中定义的函数的实现位于：kernel/kfifo.c
内核队列编程需要注意的是：
队列的size在初始化时，始终设定为2的n次方
使用队列之前将队列结构体中的锁(spinlock)释放

##映射

映射类似于其他语言(C#或者python)中的字典类型，每个唯一的id对应一个自定义的数据结构。
Linux中的映射并不是一种通用的映射，它的目标是映射一个唯一的标识数（UID）到一个指针。
映射的使用需要注意的是，给自定义的数据结构申请一个id的时候，不能直接申请id，先要分配id(函数idr_pre_get)，分配成功后，在获取一个id(函数idr_get_new)。

##二叉树
红黑树由于节点颜色的特性，保证其是一种自平衡的二叉搜索树。
红黑树的一系列规则虽然实现起来比较复杂，但是遵循起来却比较简单，而且红黑树的插入，删除性能也还不错。
所以红黑树在内核中的应用非常广泛，掌握好红黑树，即有利于阅读内核源码，也可以在自己的代码中借鉴这种数据结构。

红黑树必须满足的规则：
1. 所有节点都有颜色，要么红色，要么黑色
2. 根节点是黑色，所有叶子节点也是黑色
3. 叶子节点中不包含数据
4. 非叶子节点都有2个子节点
5. 如果一个节点是红色，那么它的父节点和子节点都是黑色的
6. 从任何一个节点开始，到其下叶子节点的路径中都包含相同数目的黑节点
红黑树中最长的路径就是红黑交替的路径，最短的路径是全黑节点的路径，再加上根节点和叶子节点都是黑色，
从而可以保证红黑树中最长路径的长度不会超过最短路径的2倍。

内核中关于红黑树定义的头文件位于：<linux/rbtree.h> include/linux/rbtree.h
头文件中定义的函数的实现位于：lib/rbtree.c

红黑树代码的注意点
内核中红黑树的使用和链表(list)有些类似，是将红黑树的节点放入自定义的数据结构中来使用的。

首先需要注意的一点是红黑树节点的定义：

struct rb_node
{
    unsigned long  rb_parent_color;
\#define    RB_RED        0
\#define    RB_BLACK    1
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
作者巧妙的利用内存对齐来将2个内容存入到一个字段中（不服不行啊^_^!）。
字段 rb_parent_color 中保存了2个信息：父节点的地址+本节点的颜色
这2个信息是如何存入一个字段的呢？主要在于 __attribute__((aligned(sizeof(long))));
这行代码的意思就是 struct rb_node 在内存中的地址需要按照4 bytes或者8 bytes对齐。
注：sizeof(long) 在32bit系统中是4 bytes，在64bit系统中是8 bytes
4 的二进制为 100 ，所以申请分配的 struct rb_node 的地址的最后2位始终是零，（父节点的地址的最后两位始终为0）
struct rb_node 的字段 rb_parent_color 就是利用最后一位来保存节点的颜色信息的。




