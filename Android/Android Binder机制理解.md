Android Binder是用来做进程通信的，Android的各个应用以及系统服务都运行在独立的进程中，它们的通信都依赖于Binder。
为什么选用Binder，在讨论这个问题之前，我们知道Android也是基于Linux内核，Linux现有的进程通信手段有以下几种：

1.管道：在创建时分配一个page大小的内存，缓存区大小比较有限；

2.消息队列：信息复制两次，额外的CPU消耗；不适合频繁或信息量大的通信；

3.共享内存：无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；

4.套接字：作为更通用的接口，传输效率低，主要用于不通机器或跨网络的通信；

5.信号量：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。

6.信号: 不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；

既然有现有的IPC方式，为什么重新设计一套Binder机制呢。主要是出于以下三个方面的考量：
#### 高性能
从数据拷贝次数来看Binder只需要进行一次内存拷贝，而管道、消息队列、Socket都需要两次，共享内存不需要拷贝，Binder的性能仅次于共享内存。
#### 稳定性
上面说到共享内存的性能优于Binder，那为什么不使用共享内存呢，因为共享内存需要处理并发同步问题，控制负责，容易出现死锁和资源竞争，稳定性较差。而Binder基于C/S架构，客户端与服务端彼此独立，稳定性较好。
#### 安全性
我们知道Android为每个应用分配了UID，用来作为鉴别进程的重要标志，Android内部也依赖这个UID进行权限管理，包括6.0以前的固定权限和6.0以后的动态权限，传统IPC只能由用户在数据包里填入UID/PID，这个标记完全 是在用户空间控制的，没有放在内核空间，因此有被恶意篡改的可能，因此Binder的安全性更高。


### 谈谈对 binder 的理解
binder 是 Android 中主要的跨进程通信方式，binder 驱动和 service manager 分别相当于网络协议中的路由器和 DNS，并基于 mmap 实现了 IPC 传输数据时只需一次拷贝。
binder 包括 BinderProxy、BpBinder 等各种 Binder 实体，以及对 binder 驱动操作的 ProcessState、IPCThreadState 封装，再加上 binder 驱动内部的结构体、命令处理，整体贯穿 Java、Native 层，涉及用户态、内核态，往上可以说到 Service、AIDL 等，往下可以说到 mmap、binder 驱动设备，是相当庞大、繁琐的一个机制。
### 基于 mmap 是如何实现一次拷贝的？
Client 与 Server 处于不同进程有着不同的虚拟地址，所以无法直接通信。而一个页框可以映射给多个页，那么就可以将一块物理内存分别与 Client 和 Server 的虚拟内存块进行映射。
Client 就只需 copy_from_user 进行一次数据拷贝，Server 进程就能读取到数据了。另外映射的虚拟内存块大小将近 1M (1M-8K)，所以 IPC 通信传输的数据量也被限制为此值。
页框是指一块实际的物理内存，页是指程序的一块内存数据单元。内存数据一定是存储在实际的物理内存上，即页必然对应于一个页框，页数据实际是存储在页框上的。
页框和页一样大，都是内核对内存的分块单位。一个页框可以映射给多个页，也就是说一块实际的物理存储空间可以映射给多个进程的多个虚拟内存空间，这也是 mmap 机制依赖的基础规则。

Client 通过 ServiceManager 或 AMS 获取到的远程 binder 实体，一般会用 Proxy 做一层封装，比如 ServiceManagerProxy、 AIDL 生成的 Proxy 类。而被封装的远程 binder 实体是一个 BinderProxy。
BpBinder 和 BinderProxy 其实是一个东西：远程 binder 实体，只不过一个 Native 层、一个 Java 层，BpBinder 内部持有了一个 binder 句柄值 handle。
ProcessState 是进程单例，负责打开 Binder 驱动设备及 mmap；IPCThreadState 为线程单例，负责与 binder 驱动进行具体的命令通信。
由 Proxy 发起 transact() 调用，会将数据打包到 Parcel 中，层层向下调用到 BpBinder ，在 BpBinder 中调用 IPCThreadState 的 transact() 方法并传入 handle 句柄值，IPCThreadState 再去执行具体的 binder 命令。
由 binder 驱动到 Server 的大概流程就是：Server 通过 IPCThreadState 接收到 Client 的请求后，层层向上，最后回调到 Stub 的 onTransact() 方法。
当然这不代表所有的 IPC 流程，比如 Service Manager 作为一个 Server 时，便没有上层的封装，也没有借助 IPCThreadState，而是初始化后通过 binder_loop() 方法直接与 binder 驱动通信的。

#### 聊聊AIDL接口的oneway关键字
AIDL接口中 oneway 关键字主要有两个特性：异步调用和串行化处理。
异步调用是指应用向 binder 驱动发送数据后不需要挂起线程等待 binder 驱动的回复，而是直接结束。像一些系统服务调用应用进程的时候就会使用 oneway，比如 AMS 调用应用进程启动 Activity，这样就算应用进程中做了耗时的任务，也不会阻塞系统服务的运行。
串行化处理是指对于一个服务端的 AIDL 接口而言，所有的 oneway 方法不会同时执行，binder 驱动会将他们串行化处理，排队一个一个调用。

