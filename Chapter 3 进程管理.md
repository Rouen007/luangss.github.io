#进程管理

##进程
进程：处于执行期的程序。（进程不限于一段可执行的代码，还包括其他资源：打开文件，挂起信号，内核的内部数据，处理器状态，一个或多个具有内存映射的内存空间，一个或多个执行线程，存放全局变量的数据段等）

（执行）线程：在进程中活动的对象。每个线程有一个独立的程序计数器、进程栈、一组进程寄存器.

内核调度的对象是线程不是进程.

对Linux而言，线程只是一种特殊的进程.

现代操作系统中，进程特工两种虚拟机制：虚拟处理器、虚拟内存。

同一进程中的线程之间可以共享虚拟内存但是都拥有自己的虚拟处理器。（每个进程有独立的虚拟处理器和虚拟内存，每个线程有独立的虚拟处理器，同一个进程内的线程有可能会共享虚拟内存。）

内核把进程的列表存放在叫任务队列（task list）的双向循环列表中

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/3.1.PNG)

进程描述符中包含一个具体进程所需的所有信息（它打开的文件，进程的地址，挂起的信号，进程的状态等）

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/3.2.PNG)

进程标识PID和线程标识TID对于同一个进程或线程来说都是相等的。

##进程状态
运行、可中断、不可中断、被其他进程跟踪、停止

运行（TASK_RUNNING）：进程是可执行的：正在执行、在队列中等待执行。
可中断（TASK_INTERRUPTIBLE）：进程正在睡眠（阻塞），等待某些条件。
不可中断（TASK_UNITERRUPTIBLE）
被其他进程跟踪（__TASK_TRACED）
停止（__TASK_STOPPED）

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/3.3.PNG)


一个程序通过执行系统调用、触发异常陷入内核空间----内核代表进程执行，并处于**进程上下文中**。

每个进程都有父进程（*parent），若干子进程（*children list）