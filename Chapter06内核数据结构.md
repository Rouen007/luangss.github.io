#内核的数据结构
链表、队列、映射、二叉树
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

#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
这个宏没什么特别的，主要是container_of这个宏

#define container_of(ptr, type, member) ({          \
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
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

// 步骤4：将__mptr的地址 - type地址和member地址之间的差
// 其实也就是获取type的地址

步骤1，2，4比较容易理解，下面的图以sturct student为例进行说明步骤3：

首先需要知道 ((TYPE *)0) 表示将地址0转换为 TYPE 类型的地址

由于TYPE的地址是0，所以((TYPE *)0)->MEMBER 也就是 MEMBER的地址和TYPE地址的差，如下图所示：
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/6.3.PNG)

 