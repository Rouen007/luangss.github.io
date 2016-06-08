#定时器与时间管理

系统中有很多与时间相关的程序（比如定期执行的任务，某一时间执行的任务，推迟一段时间执行的任务），因此，时间的管理对于linux来说非常重要。

1. 系统时间
2. 定时器
3. 定时器相关概念
4. 定时器执行流程
5. 实现程序延迟的方法
6. 定时器和延迟的例子

##系统时间

系统中管理的时间有2种：实际时间和定时器。

实际时间：实际时间就是现实中钟表上显示的时间，其实内核中并不常用这个时间，主要是用户空间的程序有时需要获取当前时间，所以内核中也管理着这个时间。

实际时间的获取是在开机后，内核初始化时从RTC读取的。
内核读取这个时间后就将其放入内核中的 xtime 变量中，并且在系统的运行中不断更新这个值。
注：RTC就是实时时钟的缩写，它是用来存放系统时间的设备。一般和BIOS一样，由主板上的电池供电的，所以即使关机也可将时间保存。
实际时间存放的变量 xtime 在文件 kernel/time/timekeeping.c中。

实际时间存放的变量 xtime 在文件 kernel/time/timekeeping.c中。
系统读写 xtime 时用的就是顺序锁。
上述场景中，**写锁必须要优先于读锁(因为 xtime** 必须及时更新)，而且写锁的使用者很少(一般只有系统定期更新xtime的线程需要持有这个锁)。
这正是顺序锁的应用场景。

##定时器
定时器是内核中主要使用的时间管理方法，通过定时器，可以有效的调度程序的执行。

内核中的定时器有2种，静态定时器和动态定时器。

静态定时器一般执行了一些周期性的固定工作：

1. 更新系统运行时间
2. 更新实际时间
3. 在SMP系统上，平衡各个处理器上的运行队列
4. 检查当前进程是否用尽了自己的时间片，如果用尽，需要重新调度。
5. 更新资源消耗和处理器时间统计值
 
动态定时器顾名思义，是在需要时（一般是推迟程序执行）动态创建的定时器，使用后销毁（一般都是只用一次）。
一般我们在内核代码中使用的定时器基本都是动态定时器，下面重点讨论动态定时器相关的概念和使用方法。

##定时器相关概念

定时器的使用中，下面3个概念非常重要：

1. HZ
2. jiffies
3. 时间中断处理程序

**HZ**
节拍率(HZ)是时钟中断的频率，表示的一秒内时钟中断的次数。

比如 HZ=100 表示一秒内触发100次时钟中断程序。

HZ的值一般与体系结构有关，x86 体系结构一般定义为 100，参考文件 include/asm-generic/param.h
HZ值的大小的设置过程其实就是平衡 精度和性能 的过程，并不是HZ值越高越好。

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.1.PNG)

此外，有一点需要注意，内核中使用的HZ可能和用户空间中定义的HZ值不一致，为了避免用户空间取得错误的时间，

内核中也定义了 USER_HZ，即用户空间使用的HZ值。

一般来说，USER_HZ 和 HZ 都是相差整数倍，内核中通过函数 jiffies_to_clock_t 来将内核来将内核中的 jiffies转为 用户空间 jiffies

**jiffies**
jiffies用来记录自系统启动以来产生的总节拍数。比如系统启动了 N 秒，那么 jiffies就为 N×HZ

jiffies的相关定义参考头文件 <linux/jiffies.h>  include/linux/jiffies.h

使用定时器时一般都是以jiffies为单位来延迟程序执行的，比如延迟5个节拍后执行的话，执行时间就是 jiffies+5

32位的jiffies的最大值为 2^32-1，在使用时有可能会出现回绕的问题。

正常情况下，上面的代码没有问题。当jiffies接近最大值的时候，就会出现回绕问题。

由于是unsinged long类型，所以jiffies达到最大值后会变成0然后再逐渐变大，如下图所示：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.2.PNG)

所以在上述的循环代码中，会出现如下情况：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.3.PNG)

1. 循环中第一次比较时，jiffies = J1，没有超时
2. 循环中第二次比较时，jiffies = J2，实际已经超时了，但是由于jiffies超过的最大值后又从0开始，所以J2远远小于timeout
3. while循环会执行很长时间(> 2^32-1 个节拍)不会结束，几乎相当于死循环了

为了回避回扰的问题，可以使用<linux/jiffies.h>头文件中提供的 time_after，time_before等宏

原理其实就是将 unsigned long 类型转换为 long 类型来避免回扰带来的错误，

long 类型超过最大值时变化趋势如下：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.4.PNG)

long 型的数据的回绕会出现在 2^31-1 变为 -2^32 的时候，如下图所示：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.5.PNG)

1. 第一次比较时，jiffies = J1，没有超时
2. 第二次比较时，jiffies = J2，一般 J2 是负数 
理论上 (long)timeout - (long)J2 = 正数 - 负数 = 正数（result） 
但是，这个正数（result）一般会大于 2^31 - 1，所以long型的result又发生了一次回绕，变成了负数。 
除非timeout和J2之间的间隔 > 2^32 个节拍，result的值才会为正数(注1)。
注1：result的值为正数时，必须是在result的值 小于 2^31-1 的情况下，大于 2^31-1 会发生回绕。

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.6.PNG)

上图中 X + Y 表示timeout 和 J2之间经过的节拍数。

result 小于 2^31-1 ，也就是 timeout - J2 < 2^31 – 1

timeout 和 -J2 表示的节拍数如上图所示。(因为J2是负数，所有-J2表示上图所示范围的值)

因为 timeout + X + Y - J2 = 2^31-1 + 2^32

所以 timeout - J2 < 2^31 - 1 时， X + Y > 2^32

也就是说，当timeout和J2之间经过至少 2^32 个节拍后，result才可能变为正数。

timeout和J2之间相差这么多节拍是不可能的(不信可以用HZ将这些节拍换算成秒就知道了。。。)

利用time_after宏就可以巧妙的避免回绕带来的超时判断问题，将之前的代码改成如下代码即可：

##时钟中断处理程序
时钟中断处理程序作为系统定时器而注册到内核中，体系结构的不同，可能时钟中断处理程序中处理的内容不同。

但是以下这些基本的工作都会执行：

1. 获得 xtime_lock 锁，以便对访问 jiffies_64 和墙上时间 xtime 进行保护
2. 需要时应答或重新设置系统时钟
3. 周期性的使用墙上时间更新实时时钟
4. 调用 tick_periodic()

static void tick_periodic(int cpu)
{
    if (tick_do_timer_cpu == cpu) {
        write_seqlock(&xtime_lock);

        /* Keep track of the next tick event */
        tick_next_period = ktime_add(tick_next_period, tick_period);

        do_timer(1);
        write_sequnlock(&xtime_lock);
    }

    update_process_times(user_mode(get_irq_regs()));
    profile_tick(CPU_PROFILING);
}
其中最重要的是 do_timer 和 update_process_times 函数。

void do_timer(unsigned long ticks)
{
    /\* jiffies_64 增加指定ticks \*/
    jiffies_64 += ticks;
    /\* 更新实际时间 \*/
    update_wall_time();
    /\* 更新系统的平均负载值 \*/
    calc_global_load();
}

void update_process_times(int user_tick)
{
    struct task_struct *p = current;
    int cpu = smp_processor_id();

    /\* 更新当前进程占用CPU的时间 \*/
    account_process_tick(p, user_tick);
    /\* 同时触发软中断，处理所有到期的定时器 \*/
    run_local_timers();
    rcu_check_callbacks(cpu, user_tick);
    printk_tick();
    /\* 减少当前进程的时间片数 \*/
    scheduler_tick();
    run_posix_cpu_timers(p);
}

##定时器执行流程

这里讨论的定时器执行流程是动态定时器的执行流程。

**定时器**在内核中用一个链表来保存的，链表的每个节点都是一个定时器。

**定时器的生命周期**

一个动态定时器的生命周期中，一般会经过下面的几个步骤：

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/11.7.PNG)

1. 初始化定时器：

struct timer_list my_timer; /\* 定义定时器 \*/
init_timer(&my_timer);      /\* 初始化定时器 \*/
 

2. 填充定时器：

my_timer.expires = jiffies + delay; /\* 定义超时的节拍数 \*/
my_timer.data = 0;                  /\* 给定时器函数传入的参数 \*/
my_timer.function = my_function;    /\* 定时器超时时，执行的自定义函数 \*/

/\* 从定时器结构体中，我们可以看出这个函数的原型应该如下所示： \*/
void my_function(unsigned long data);

3. 激活定时器和修改定时器：

激活定时器之后才会被触发，否则定时器不会执行。

修改定时器主要是修改定时器的延迟时间，修改定时器后，不管原先定时器有没有被激活，都会处于激活状态。

填充定时器结构之后，可以只激活定时器，也可以只修改定时器，也可以激活定时器后再修改定时器。

所以填充定时器结构和触发定时器之间的步骤，也就是虚线框中的步骤是不确定的。

add_timer(&my_timer);  /\* 激活定时器 \*/
mod_timer(&my_timer, jiffies + new_delay);  /\* 修改定时器，设置新的延迟时间 \*/
 
4. 触发定时器：

每次时钟中断处理程序会检查已经激活的定时器是否超时，如果超时就执行定时器结构中的自定义函数。

5. 删除定时器：

激活和未被激活的定时器都可以被删除，已经超时的定时器会自动删除，不用特意去删除。

/\*
 \* 删除激活的定时器时，此函数返回1
 \* 删除未激活的定时器时，此函数返回0
 \*/
del_timer(&my_timer);

在多核处理器上用 del_timer 函数删除定时器时，可能在删除时正好另一个CPU核上的时钟中断处理程序正在执行这个定时器，于是就形成了竞争条件。

为了避免竞争条件，建议使用 del_timer_sync 函数来删除定时器。

del_timer_sync 函数会等待其他处理器上的定时器处理程序全部结束后，才删除指定的定时器。

/\*
 \* 和del_timer 不同，del_timer_sync 不能在中断上下文中执行
 \*/
del_timer_sync(&my_timer);

##实现程序延迟的方法

内核中有个利用定时器实现延迟的函数 schedule_timeout

这个函数会将当前的任务睡眠到指定时间后唤醒，所以等待时不会占用CPU时间。

/\* 将任务设置为可中断睡眠状态 \*/
set_current_state(TASK_INTERRUPTIBLE);

/\* 小睡一会儿，“s“秒后唤醒 \*/
schedule_timeout(s*HZ);
 
查看 schedule_timeout 函数的实现方法，可以看出是如何使用定时器的。

signed long __sched schedule_timeout(signed long timeout)
{
    /\* 定义一个定时器 \*/
    struct timer_list timer;
    unsigned long expire;

    switch (timeout)
    {
    case MAX_SCHEDULE_TIMEOUT:
        /\*
         \* These two special cases are useful to be comfortable
         \* in the caller. Nothing more. We could take
         \* MAX_SCHEDULE_TIMEOUT from one of the negative value
         \* but I' d like to return a valid offset (>=0) to allow
         \* the caller to do everything it want with the retval.
         \*/
        schedule();
        goto out;
    default:
        /\*
         \* Another bit of PARANOID. Note that the retval will be
         \* 0 since no piece of kernel is supposed to do a check
         \* for a negative retval of schedule_timeout() (since it
         \* should never happens anyway). You just have the printk()
         \* that will tell you if something is gone wrong and where.
         \*/
        if (timeout < 0) {
            printk(KERN_ERR "schedule_timeout: wrong timeout "
                "value %lx\n", timeout);
            dump_stack();
            current->state = TASK_RUNNING;
            goto out;
        }
    }

    /\* 设置超时时间 \*/
    expire = timeout + jiffies;

    /\* 初始化定时器，超时处理函数是 process_timeout，后面再补充说明一下这个函数 \*/
    setup_timer_on_stack(&timer, process_timeout, (unsigned long)current);
    /\* 修改定时器，同时会激活定时器 \*/
    __mod_timer(&timer, expire, false, TIMER_NOT_PINNED);
    /\* 将本任务睡眠，调度其他任务 \*/
    schedule();
    /\* 删除定时器，其实就是 del_timer_sync 的宏
    del_singleshot_timer_sync(&timer);

    /\* Remove the timer from the object tracker \*/
    destroy_timer_on_stack(&timer);

    timeout = expire - jiffies;

 out:
    return timeout < 0 ? 0 : timeout;
}
EXPORT_SYMBOL(schedule_timeout);

/\* 
 \* 超时处理函数 process_timeout 里面只有一步操作，唤醒当前任务。
 \* process_timeout 的参数其实就是 当前任务的地址
 \*/
static void process_timeout(unsigned long __data)
{
    wake_up_process((struct task_struct *)__data);
}

schedule_timeout 一般用于延迟时间较长的程序。

这里的延迟时间较长是对于计算机而言的，其实也就是延迟大于 1 个节拍(jiffies)。

对于某些极其短暂的延迟，比如只有1ms，甚至1us，1ns的延迟，必须使用特殊的延迟方法。

1s = 1000ms = 1000000us = 1000000000ns (1秒=1000毫秒=1000000微秒=1000000000纳秒)

假设 HZ=100，那么 1个节拍的时间间隔是 1/100秒，大概10ms左右。

所以对于那些极其短暂的延迟，schedule_timeout 函数是无法使用的。

好在内核对于这些短暂，精确的延迟要求也提供了相应的宏。


/\* 具体实现参见 include/linux/delay.h
 \* 以及 arch/x86/include/asm/delay.h
 \*/
#define mdelay(n) ...
#define udelay(n) ...
#define ndelay(n) ...

通过这些宏，可以简单的实现延迟，比如延迟 5ns，只需 ndelay(5); 即可。

这些短延迟的实现原理并不复杂，

首先，内核在启动时就计算出了当前处理器1秒能执行多少次循环，即 loops_per_jiffy

(loops_per_jiffy 的计算方法参见 init/main.c 文件中的 calibrate_delay 方法)。

然后算出延迟 5ns 需要循环多少次，执行那么多次空循环即可达到延迟的效果。

loops_per_jiffy 的值可以在启动信息中看到：

[root@vbox ~]# dmesg | grep delay
Calibrating delay loop (skipped), value calculated using timer frequency.. 6387.58 BogoMIPS (lpj=3193792)
我的虚拟机中看到 (lpj=3193792)






