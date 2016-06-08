#内存管理

内核的内存使用不像用户空间那样随意，内核的内存出现错误时也只有靠自己来解决（用户空间的内存错误可以抛给内核来解决）。

所有内核的内存管理必须要简洁而且高效。

主要内容：

内存的管理单元
获取内存的方法
获取高端内存
内核内存的分配方式
总结

##内存的管理单元

内存最基本的管理单元是页，同时按照内存地址的大小，大致分为3个区。

###页
页的大小与体系结构有关，在 x86 结构中一般是 4KB或者8KB。

可以通过 getconf 命令来查看系统的page的大小：

$ getconf -a | grep -i 'page'

PAGESIZE                           4096
PAGE_SIZE                          4096
_AVPHYS_PAGES                      637406
_PHYS_PAGES                        2012863

页的结构体头文件是： <linux/mm_types.h> 位置：include/linux/mm_types.h

/\*
 \* 页中包含的成员非常多，还包含了一些联合体
 \* 其中有些字段我暂时还不清楚含义，以后再补上。。。
 \*/
struct page {
    unsigned long flags;    /\* 存放页的状态，各种状态参见<linux/page-flags.h> \*/
    atomic_t _count;        /\* 页的引用计数 \*/
    union {
        atomic_t _mapcount;    /\* 已经映射到mms的pte的个数 \*/
        struct {        /\* 用于slab层 \*/
            u16 inuse;
            u16 objects;
        };
    };
    union {
        struct {
        unsigned long private;        /\* 此page作为私有数据时，指向私有数据 \*/
        struct address_space *mapping;    /\* 此page作为页缓存时，指向关联的address_space \*/
        };
#if USE_SPLIT_PTLOCKS
        spinlock_t ptl;
#endif
        struct kmem_cache *slab;    /\* 指向slab层 \*/
        struct page *first_page;    /\* 尾部复合页中的第一个页 \*/
    };
    union {
        pgoff_t index;        /\* Our offset within mapping. \*/
        void *freelist;        /\* SLUB: freelist req. slab lock \*/
    };
    struct list_head lru;    /\* 将页关联起来的链表项 \*/
#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;            /\* 页的虚拟地址 \*/
#endif /\* WANT_PAGE_VIRTUAL \*/
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
    unsigned long debug_flags;    /\* Use atomic bitops on this \*/
#endif

#ifdef CONFIG_KMEMCHECK
    /\*
     \* kmemcheck wants to track the status of each byte in a page; this
     \* is a pointer to such a status block. NULL if not tracked.
     \*/
    void *shadow;
#endif
};

物理内存的每个页都有一个对应的 page 结构，看似会在管理上浪费很多内存，其实细细算来并没有多少。

比如上面的page结构体，每个字段都算4个字节的话，总共40多个字节。(union结构只算一个字段)

那么对于一个页大小 4KB 的 4G内存来说，一个有 4*1024*1024 / 4 = 1048576 个page，

一个page 算40个字节，在管理内存上共消耗内存 40MB左右。

如果页的大小是 8KB 的话，消耗的内存只有 20MB 左右。相对于 4GB 来说并不算很多。

page结构与物理页相关，而并非与虚拟页相关。内核需要知道一个页是否空闲（是否被分配）。如果页已经被分配，内核还需要知道谁拥有这个页。

###区
页是内存管理的最小单元，但是并不是所有的页对于内核都一样。

内核将内存按地址的顺序分成了不同的区，有的硬件只能访问有专门的区。

内核中分的区定义在头文件 <linux/mmzone.h> 位置：include/linux/mmzone.h

内存区的种类参见 enum zone_type 中的定义。

内存区的结构体定义也在 <linux/mmzone.h> 中。

具体参考其中 struct zone 的定义。

其实一般主要关注的区只有3个：

区           描述                     物理内存

ZONE_DMA     DMA使用的页				<16MB
ZONE_NORMAL	 正常可寻址的页			  16～896MB
ZONE_HIGHMEM 动态映射的页				>896MB
 
某些硬件只能直接访问内存地址，不支持内存映射，对于这些硬件内核会分配 ZONE_DMA 区的内存。

某些硬件的内存寻址范围很广，比虚拟寻址范围还要大的多，那么就会用到 ZONE_HIGHMEM 区的内存，

对于 ZONE_HIGHMEM 区的内存，后面还会讨论。

对于大部分的内存申请，只要用 ZONE_NORMAL 区的内存即可。
 
##获取内存的方法

内核中提供了多种获取内存的方法，了解各种方法的特点，可以恰当的将其用于合适的场景。

###按页获取 - 最原始的方法，用于底层获取内存的方式
以下分配内存的方法参见：<linux/gfp.h>

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.1.PNG)

alloc** 方法和 get** 方法的区别在于，一个返回的是内存的物理地址，一个返回内存物理地址映射后的逻辑地址。

如果无须直接操作物理页结构体的话，一般使用 get** 方法。

相应的释放内存的函数如下：也是在 <linux/gfp.h> 中定义的

extern void __free_pages(struct page *page, unsigned int order);
extern void free_pages(unsigned long addr, unsigned int order);
extern void free_hot_page(struct page *page);

在请求内存时，参数中有个 gfp_mask 标志，这个标志是控制分配内存时必须遵守的一些规则。

gfp_mask 标志有3类：(所有的 GFP 标志都在 <linux/gfp.h> 中定义)

1. 行为标志 ：控制分配内存时，分配器的一些行为
2. 区标志   ：控制内存分配在那个区(ZONE_DMA, ZONE_NORMAL, ZONE_HIGHMEM 之类)
3. 类型标志 ：由上面2种标志组合而成的一些常用的场景

行为标志主要有以下几种：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.2.PNG)

区标志主要以下3种：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.3.PNG)

**注1**：ZONE_DMA32 和 ZONE_DMA 类似，该区包含的页也可以进行DMA操作。 
         唯一不同的地方在于，ZONE_DMA32 区的页只能被32位设备访问。 
**注2**：优先从 ZONE_HIGHMEM 分配，如果 ZONE_HIGHMEM 没有多余的页则从 ZONE_NORMAL 分配。

类型标志是编程中最常用的，在使用标志时，应首先看看类型标志中是否有合适的，如果没有，再去自己组合 行为标志和区标志。

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.4.PNG)

以上各种类型标志的使用场景总结：
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.5.PNG)

###按字节获取 - 用的最多的获取方法

这种内存分配方法是平时使用比较多的，主要有2种分配方法：kmalloc()和vmalloc()

kmalloc的定义在 <linux/slab_def.h> 中

/\*
 \* @size  - 申请分配的字节数
 \* @flags - 上面讨论的各种 gfp_mask
 \*/
static __always_inline void *kmalloc(size_t size, gfp_t flags)
#end_src

vmalloc的定义在 mm/vmalloc.c 中
#+begin_src C
/\*
 \* @size - 申请分配的字节数
 \*/
void *vmalloc(unsigned long size)

**kmalloc 和 vmalloc 区别在于**：

1. kmalloc 分配的内存物理地址是连续的，虚拟地址也是连续的
2. vmalloc 分配的内存物理地址是不连续的，虚拟地址是连续的

因此在使用中，用的较多的还是 kmalloc，因为kmalloc 的性能较好。

因为kmalloc的物理地址和虚拟地址之间的映射比较简单，只需要将物理地址的第一页和虚拟地址的第一页关联起来即可。

而vmalloc由于物理地址是不连续的，所以要将物理地址的每一页都和虚拟地址关联起来才行。

kmalloc 和 vmalloc 所对应的释放内存的方法分别为：

void kfree(const void *)
void vfree(const void *)

##slab层实现原理
linux中的高速缓存是用所谓 slab 层来实现的，slab层即内核中管理高速缓存的机制。

整个slab层的原理如下：

1. 可以在内存中建立各种对象的高速缓存(比如进程描述相关的结构 task_struct 的高速缓存)
2. 除了针对特定对象的高速缓存以外，也有通用对象的高速缓存
3. 每个高速缓存中包含多个 slab，slab用于管理缓存的对象
4. slab中包含多个缓存的对象，物理上由一页或多个连续的页组成
 

高速缓存->slab->缓存对象之间的关系如下图：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/12.6.PNG)

slab层的应用
slab结构体的定义参见：mm/slab.c


struct slab {
    struct list_head list;   /\* 存放缓存对象，这个链表有 满，部分满，空 3种状态  \*/
    unsigned long colouroff; /\* slab 着色的偏移量 \*/
    void *s_mem;             /\* 在 slab 中的第一个对象 \*/
    unsigned int inuse;         /\* slab 中已分配的对象数 \*/
    kmem_bufctl_t free;      /\* 第一个空闲对象(如果有的话) \*/
    unsigned short nodeid;   /\* 应该是在 NUMA 环境下使用 \*/
};

slab层的应用主要有四个方法：

1. 高速缓存的创建
2. 从高速缓存中分配对象
3. 向高速缓存释放对象
4. 高速缓存的销毁

##获取高端内存

高端内存就是之前提到的 ZONE_HIGHMEM 区的内存。

在x86体系结构中，这个区的内存不能映射到内核地址空间上，也就是没有逻辑地址，

为了使用 ZONE_HIGHMEM 区的内存，内核提供了永久映射和临时映射2种手段：

###永久映射
永久映射的函数是可以睡眠的，所以只能用在进程上下文中。

###临时映射
临时映射不会阻塞，也禁止了内核抢占，所以可以用在中断上下文和其他不能重新调度的地方。

##内核内存的分配方式

内核的内存分配和用户空间的内存分配相比有着更多的限制条件，同时也有着更高的性能要求。

下面讨论2个和用户空间不同的内存分配方式。

###内核栈上的静态分配
用户空间中一般不用担心栈上的内存不足，也不用担心内存的管理问题(比如内存越界之类的)，

即使出了异常也有内核来保证系统的正常运行。

而在内核空间则完全不一样，不仅栈空间有限，而且为了管理的效率和尽量减少问题的发生，

内核栈一般都是小而且固定的。

在x86体系结构中，内核栈的大小一般就是1页或2页，即 4KB ~ 8KB

内核栈可以在编译内核时通过配置选项将内核栈配置为1页，

配置为1页的好处是分配时比较简单，只有一页，不存在内存碎片的情况，因为一页是本就是分配的最小单位。

当有中断发生时，如果共享内核栈，中断程序和被中断程序共享一个内核栈会可能导致空间不足，

于是，每个进程除了有个内核栈之外，还有一个中断栈，中断栈一般也就1页大小。

###按CPU分配
与单CPU环境不同，SMP环境下的并行是真正的并行。单CPU环境是宏观并行，微观串行。

真正并行时，会有更多的并发问题。

假定有如下场景：

void* p;

if (p == NULL)
{
/* 对 P 进行相应的操作，最终 P 不是NULL了 */
}
else
{
/* P 不是NULL，继续对 P 进行相应的操作 */
}

在上述场景下，可能会有以下的执行流程：

刚开始 p == NULL
线程A 执行到 [if (p == NULL)] ，刚进入 if 内的代码时被线程B 抢占 
由于线程A 还没有执行 if 内的代码，所以 p 仍然是 NULL
线程B 抢占到CPU后开始执行，执行到 [if (p == NULL)]时， 发现 p 是 NULL，执行 if 内的代码
线程B 执行完后，线程A 重新被调度，继续执行 if 的代码 
其实此时由于线程B 已经执行完，p 已经不是 NULL了，线程A 可能会破坏线程B 已经完成的处理，导致数据不一致

在单CPU环境下，上述情况无需加锁，只需在 if 处理之前禁止内核抢占，在 else 处理之后恢复内核抢占即可。

而在SMP环境下，上述情况必须加锁，因为禁止内核抢占只能禁止当前CPU的抢占，其他的CPU仍然调度线程B 来抢占线程A 的执行

SMP环境下加锁过多的话，会严重影响并行的效率，如果是自旋锁的话，还会浪费其他CPU的执行时间。

所以内核中才有了按CPU分配数据的接口。

按CPU分配数据之后，每个CPU自己的数据不会被其他CPU访问，虽然浪费了一点内存，但是会使系统更加的简洁高效。

###按CPU分配的优势
按CPU来分配数据主要有2个优点：

最直接的效果就是减少了对数据的锁，提高了系统的性能
由于每个CPU有自己的数据，所以处理器切换时可以大大减少缓存失效的几率 (*注1)

注1：如果一个处理器操作某个数据，而这个数据在另一个处理器的缓存中时，那么存放这个数据的那个

处理器必须清理或刷新自己的缓存。持续的缓存失效成为缓存抖动，对系统性能影响很大。

###编译时分配
可以在编译时就定义分配给每个CPU的变量，其分配的接口参见：<linux/percpu-defs.h>

/* 给每个CPU声明一个类型为 type，名称为 name 的变量 */
DECLARE_PER_CPU(type, name)
/* 给每个CPU定义一个类型为 type，名称为 name 的变量 */
DEFINE_PER_CPU(type, name)
注意上面两个宏，一个是声明，一个是定义。

其实也就是 DECLARE_PER_CPU 中多了个 extern 的关键字

分配好变量后，就可以在代码中使用这个变量 name 了。

DEFINE_PER_CPU(int, name);      /* 为每个CPU定义一个 int 类型的name变量 */
get_cpu_var(name)++;            /* 当前处理器上的name变量 +1 */
put_cpu_var(name);              /* 完成对name的操作后，激活当前处理器的内核抢占 */

通过 get_cpu_var 和 put_cpu_var 的代码，我们可以发现其中有禁止和激活内核抢占的函数。

相关代码在 <linux/percpu.h> 中

#define get_cpu_var(var) (*({                \
    extern int simple_identifier_##var(void);    \
    preempt_disable();/* 这句就是禁止当前处理器上的内核抢占 */    \
    &__get_cpu_var(var); }))
#define put_cpu_var(var) preempt_enable()  /* 这句就是激活当前处理器上的内核抢占 */

###运行时分配
除了像上面那样静态的给每个CPU分配数据，还可以以指针的方式在运行时给每个CPU分配数据。

动态分配参见：<linux/percpu.h>
/* 给每个处理器分配一个 size 字节大小的对象，对象的偏移量是 align */
extern void *__alloc_percpu(size_t size, size_t align);
/* 释放所有处理器上已分配的变量 __pdata */
extern void free_percpu(void *__pdata);

/* 还有一个宏，是按对象类型 type 来给每个CPU分配数据的，
 * 其实本质上还是调用了 __alloc_percpu 函数 */
#define alloc_percpu(type)    (type *)__alloc_percpu(sizeof(type), \
                               __alignof__(type))

在众多的内存分配函数中，如何选择合适的内存分配函数很重要，下面总结了一些选择的原则：

**应用场景** 							**分配函数选择**

如果需要物理上连续的页	        	选择低级页分配器或者 kmalloc 函数
如果kmalloc分配是可以睡眠				指定 GFP_KERNEL 标志
如果kmalloc分配是不能睡眠				指定 GFP_ATOMIC 标志
如果不需要物理上连续的页			vmalloc 函数 (vmalloc 的性能不如 kmalloc)
如果需要高端内存				alloc_pages 函数获取 page 的地址，在用 kmap 之类的函数进行映射
如果频繁撤销/创建教导的数据结构				建立slab高速缓存

GFP_ATOMIC	__GFP_HIGH	这个标志用在中断处理程序，下半部，持有自旋锁以及其他不能睡眠的地方

GFP_KERNEL	(__GFP_WAIT ｜ __GFP_IO ｜ __GFP_FS )	这是常规的分配方式，可能会阻塞。这个标志在睡眠安全时用在进程上下文代码中。 
为了获得调用者所需的内存，内核会尽力而为。这个标志应当为首选标志









