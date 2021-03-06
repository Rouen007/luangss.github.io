##操作系统
操作系统是指在整个系统中负责完成最基本功能和系统管理的那些部分。（内核、设备驱动程序、启动引导程序、命令行Shell、用户界面、基本的文件管理工具以及系统工具）。
##内核
内核是操作系统的核心，系统的其它部分必须依靠内核这部分软件提供的服务：管理硬件设备，分配系统资源。内核由负责响应中断的中断服务程序，负责管理多个进程从而分享处理器时间的调度程序，负责管理地址空间的内存管理程序和网络，进程间通信等系统服务程序共同组成。
内核属于系统态拥有保护内存空间以及访问硬件设备的所有权限而应用程序属于用户态，用户态只对部分资源可见。
应用程序通过系统调用与内核通信。应用程序通过调用库函数再由库函数通过系统调用界面，让内核代表其完成不同任务。

![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/1.1.PNG)
##处理器的活动：
1. 运行于用户空间，执行用户进程；
2. 运行于内核空间，处于进程上下文，代表某个特定进程的执行；
3. 运行于内核空间，处于中断上下文，与任何进程无关，处理某个特定的中断。

操作系统内核分为单内核、微内核（、外内核）：
单内核中的所有内核服务都运行在一个大的内核地址空间上，可以直接调控函数，而微内核则是将功能划分成多个独立的过程，需要采用进程间通信IPC机制互通消息，互换服务。
Linux属于单内核，运行在单独的内核地址空间上。但是也借鉴了微内核的设计：模块化设计、抢占式内核、支持内核线程以及动态装载内核模块。和单内核的保持性能的设计一致，Linux让所有的事情都运行在内核态，直接调用函数，无需消息传递。
![image](https://github.com/Rouen007/luangss.github.io/blob/master/image-lib/1.2.png)

Linux 支持：
支持动态加载内核模块
支持对称多处理（SMP）
内核可以抢占（preemptive），允许内核运行的任务有优先执行的能力
不区分线程和进程

##内核版本号
内核的版本号主要有四个数组组成。比如版本号：2.6.26.1  其中，
2  - 主版本号
6  - 从版本号或副版本号
26 - 修订版本号
1  - 稳定版本号
副版本号表示这个版本是稳定版（偶数）还是开发版（奇数），上面例子中的版本号是稳定版。
稳定的版本可用于企业级环境。
修订版本号的升级包括BUG修正，新的驱动以及新的特性的追加。
稳定版本号主要是一些关键性BUG的修改。
