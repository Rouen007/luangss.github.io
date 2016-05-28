##操作系统
操作系统是指在整个系统中负责完成最基本功能和系统管理的那些部分。（内核、设备驱动程序、启动引导程序、命令行Shell、用户界面、基本的文件管理工具以及系统工具）。
##内核
内核是操作系统的核心，系统的其它部分必须依靠内核这部分软件提供的服务：管理硬件设备，分配系统资源。内核由负责响应中断的中断服务程序，负责管理多个进程从而分享处理器时间的调度程序，负责管理地址空间的内存管理程序和网络，进程间通信等系统服务程序共同组成。
内核属于系统态拥有保护内存空间以及访问硬件设备的所有权限而应用程序属于用户态，用户态只对部分资源可见。
应用程序通过系统调用与内核通信。应用程序通过调用库函数再由库函数通过系统调用界面，让内核代表其完成不同任务。

![image](https://github.com/Rouen007/luangss.github.io/image-lib/1.1.png)
##处理器的活动：
1. 运行于用户空间，执行用户进程；
2. 运行于内核空间，处于进程上下文，代表某个特定进程的执行；
3. 运行于内核空间，处于中断上下文，与任何进程无关，处理某个特定的中断。

