# Linux OS implementation of process model

在本书的chapter 1.1. Linux Versus Other Unix-Like Kernels对linux OS中process model的实现思路进行了概括：

> Most modern operating systems have some kind of support for multithreaded applications that is, user programs that are designed in terms of many relatively independent execution flows that share a large portion of the application data structures. A multithreaded user application could be composed of many **lightweight processes** (LWP), which are **processes** that can operate on a common address space, common physical memory pages, common opened files, and so on. Linux defines its own version of **lightweight processes**, which is different from the types used on other systems such as SVR4 and Solaris. While all the commercial Unix variants of LWP are based on **kernel threads**, Linux regards **lightweight processes** as the basic **execution context** and handles them via the nonstandard   [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html)  system call.

Process和Thread的概念在前面的章节已经描述了，上面这段话引入了一个新的概念：**lightweight processes** (LWP)，**lightweight processes** 是一个实现层面的概念，而Process和Thread是标准定义的概念。

结合前面章节关于process model的描述和上面这段关于linux OS中process model实现概述，可以总结：

- Linux OS的kernel scheduling entity是**lightweight processes**

- linux OS通过它的**lightweight processes**来实现process model的；linux OS中，light weight process对应的是标准的thread，linux OS中，一个process由n（n>=1）个light weight process组成（显然，当n为1时，一个process只有一个lightweight process，这就是我们通常所说的single-thread process）。

  在本书chapter 3.1. Processes, Lightweight Processes, and Threads中定义了*thread group*的概念，*thread group*相当于process。

  在本书chapter 3.1. Processes, Lightweight Processes, and Threads中提出：对于面向process的system call，thread group要“act as a whole”即表示为一个整体，这些system call包括：[getpid](http://man7.org/linux/man-pages/man2/getpid.2.html)、[kill](http://man7.org/linux/man-pages/man2/kill.2.html)。





---

需要注意的是，在本书中有时候会将lightweight process简称为process，比如上面这段话中的这句：

> A multithreaded user application could be composed of many **lightweight processes** (LWP), which are **processes** that can operate on a common address space, common physical memory pages, common opened files, and so on. 

所以在本书中，process不一定指的是标准的process，有的时候指的是lightweight process；在本书中，使用**thread group**来表示标准的process。记住这一点，否则本书中的很多地方都会搞混淆。

关于lightweight process，参见：

- [Light-weight process](https://en.wikipedia.org/wiki/Light-weight_process)

---



## 更加深入的分析

上面这段话告诉了我们，lightweight process是由nonstandard   [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html)  system call创建。标准给出的创建process的api是[fork(2)](http://man7.org/linux/man-pages/man2/fork.2.html)，POSIX标准所定义的创建thread的api是[pthread_create(3)](http://man7.org/linux/man-pages/man3/pthread_create.3.html)，下面通过linux的man来探索它们的实现细节：

在[PTHREADS(7)](http://man7.org/linux/man-pages/man7/pthreads.7.html)的Linux implementations of POSIX threads章节给出了linux中POSIX threads的实现方式的详细信息

> Over time, two threading implementations have been provided by the GNU C library on Linux:
>
> LinuxThreads
>
> ​      This is the original Pthreads implementation.  Since glibc 2.4, this implementation is no longer supported.
>
> NPTL (Native POSIX Threads Library)
>
> ​      This is the modern Pthreads implementation.  By comparison  with LinuxThreads, NPTL provides closer conformance to the requirements of the POSIX.1 specification and better performance when creating large numbers of threads.  NPTL is available since glibc 2.3.2, and requires features that are present in the Linux 2.6 kernel.
>
> Both of these are so-called 1:1 implementations, meaning that each thread maps to a kernel scheduling entity.  Both threading implementations employ the Linux [clone(2](http://man7.org/linux/man-pages/man2/clone.2.html)) system call.  In NPTL, thread synchronization primitives (mutexes, thread joining, and so on) are implemented using the Linux [futex(2)](http://man7.org/linux/man-pages/man2/futex.2.html) system call.

可以看到无论采用哪种方式，最终都是依赖 [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) 。

在[fork(2)](http://man7.org/linux/man-pages/man2/fork.2.html)的[NOTES](http://man7.org/linux/man-pages/man2/fork.2.html#NOTES)章节描述了`fork`的实现细节：

> Since version 2.3.3, rather than invoking the kernel's fork() system call, the glibc fork() wrapper that is provided as part of the NPTL threading implementation invokes [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) with flags that provide the
> same effect as the traditional system call.  (A call to fork() is equivalent to a call to [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) specifying flags as just SIGCHLD.) The glibc wrapper invokes any **fork handlers** that have been established using [pthread_atfork(3)](http://man7.org/linux/man-pages/man3/pthread_atfork.3.html).

显然，[fork(2)](http://man7.org/linux/man-pages/man2/fork.2.html)的实现也是依赖 [clone(2)](http://man7.org/linux/man-pages/man2/clone.2.html) 。



## linux kernel如何实现process与thread

参见3.1. Processes, Lightweight Processes, and Threads

我觉得要想解释好这个问题，需要梳理一下linux的fork，clone之间的关系。在[Fork (system call)](https://en.wikipedia.org/wiki/Fork_(system_call))这篇文章中梳理地非常好。在3.4. Creating Processes中也有相关的描述。



### per-process kernel data structures

在3.4. Creating Processes中提出了这个说法，它让我想起了两件事情：

- process作为system resource分配单位，它有哪些resource呢？显然，它的所有的resource都需要使用一个 kernel data structures来进行描述。有必要总结per-process的resource以及对应的kernel data structures。与此相关的一个问题就是，这些resource哪些是child process可以继承的，哪些是无法继承的。

- 显然，多个lightweight process是可以共享per-process kernel data structure的（这是标准规定的），这种共享，我觉得实现上应该也是非常简单的，无非就是传入一个指针。



#### Address space

这个问题是由前面的关于process的resource的思考衍生出来的。Address space是一个process非常重要的resource，可以认为它是process进行活动的空间。目前的OS都是采用的virtual address，即process运行的时候，所使用的是virtual memory，所以也可以将Address space称为Virtual address space。关于process的Virtual address space，我有如下疑问：

Question:

process使用virtual memory，并且使用基于page的memory management，那它是如何实现的基于page的virtual memory呢？是分割为一个一个的page？

经过简单的思考，我觉得应该是编译器在给生成代码的时候其实是不需要考虑这个问题的，因为是OS在运行program的时候按照page进行memory management，无论编译器生成的program是怎样的，是OS负责将这些program装入到memory中，这一切对compiler而言都是透明的。

但是这个问题可以延伸一下：我们知道，编译器生成的代码肯定是需要遵循alignment的，那这就涉及到alignment和page size之间的关系；应该只要符合alignment，那么应该就不会存在一个数据存储跨越了多个page的情况了。

Question:

进程的virtual address space都是相同的，那virtual address是如何映射到physical memory address的呢？

既然使用的是demand page，也就是在process运行的时候需要访问该virtual memory的时候，才allocate physical memory或者swap-in，才将virtual address映射到physical memory并将这些信息保存到该process的page table中。

其实通过这个思考才发现virtual memory的重要价值所在，它是实现demand page的基础，它是实现扩充memory的基础，它是实现copy on write的基础。



Question:

如1.6.8.4. Process virtual address space handling节所叙述的

> The kernel usually stores a process virtual address space as a list of memory area descriptors .

即我们通常将virtual address space分割为多块，那是在什么地方将virtual address space分割为如上所述的a **list** of *memory area descriptors* ？

operating system采用的是demand paging，并且stack的增长方向和heap的增长方向相反，那这些又是如何实现的呢？

要想完全理解这个问题，阅读calling convention。我觉得process在运行过程中，对call stack的维护是一个非常重要的活动，每次new一个栈帧都需要分配新的内存空间重要才能够保证process运行下去。

另外一个问题是，为什么需要申请memory？

其实如果这个系统中只有一个程序的话，那么它想怎么样使用memory就怎么样使用memory，但是问题是，我们的系统是需要支持多任务的，那它就需要做好不同的process之间的隔离，A process不能够使用B process的东西。所以，所有的process都必须要先想OS申请memory，然后才能够使用，OS会记住memory的所属，这样就能够保证不冲突了。其次是process的运行是需要一定的memory space来存放它的相关的数据的，比如在发生context switch的时候，就需要将它的context相关的数据都保存到它的memory space中来。另外一个就是process的call stack，这是非常重要的一个需要memory space的场所。



Question:

如前所述，栈也是virtual address space的成分之一，每个thread都有各自**独立**的call stack，而所有的thread理论上都是共享process的virtual address space的，那这又是如何实现的呢？

其实最最简单的方式是查看`task_descriptor`的成员变量



## LWP VS thread VS kernel thread?

关于本段，有疑问：LWP VS thread VS kernel thread?

上一段中所描述的：Linux **kernel threads** do not represent the basic **execution context** abstraction.

本段中所描述的：Linux regards **lightweight processes** as the basic **execution context** and handles them via the nonstandard  `clone( )` system call.

显然，kernel thread不是linux的lightweight process。

显然linux的lightweight process是需要由linux的scheduler来进行调度的，那kernel thread是由谁来进行调度呢？下面是一些有价值的内容：

- [Are kernel threads processes and daemons?](https://unix.stackexchange.com/questions/266434/are-kernel-threads-processes-and-daemons)
- [Difference between user-level and kernel-supported threads?](https://stackoverflow.com/questions/15983872/difference-between-user-level-and-kernel-supported-threads)
- [Kernel threads made easy](https://lwn.net/Articles/65178/)



## Linux OS lightweigh thread所共享的

- 标准process ID

# See also

参见章节：

1.6.2. Process Implementation
1.6.4. Process Address Space


