# Control path

Control path这个概念是我由kernel control path启发而创建的，它表示OS中所有可能的活动/执行流程，之所以创建这个概念，是因为它可以方便我们来统一地、概括地描述一些问题（一个抽象过程）。

OS中有如下control path：

- kernel control path
- kernel thread
- task（process/thread，现代OS需要支持[multitasking](https://en.wikipedia.org/wiki/Computer_multitasking)）

在本书的有些章节会使用“execution context”、“execution flow”等词语，其实它们和本文所定义的control path表示的是相同的意思。

Control path的典型特征是：它执行都可能会被interrupted：

- 一旦发生了hardware interrupt，OS kernel会立即去响应，从而interrupt（suspend）当前执行的kernel control path，转去执行新的kernel control path。即原kernel control path会被interrupted。

- task是现代OS为支持[multitasking](https://en.wikipedia.org/wiki/Computer_multitasking)而创建的，它由[scheduler](https://en.wikipedia.org/wiki/Scheduling_(computing))进行调度执行的，目前linux采取的调度策略是[Preemptive multitasking](https://en.wikipedia.org/wiki/Preemption_(computing))，这种策略的本质是：

  > It is normally carried out by a [privileged](https://en.wikipedia.org/wiki/Protection_ring) task or part of the system known as a preemptive [scheduler](https://en.wikipedia.org/wiki/Scheduling_(computing)), which has the power to **preempt**, or interrupt, and later resume, other tasks in the system.

  即它可能会interrupt（suspend）正在执行的task，然后转去执行另外一个task。



显然这是OS为了高效，让多个control path interleave（交错运行），为了实现[Reentrancy](https://en.wikipedia.org/wiki/Reentrancy_(computing)) ，每个control path都要有自己private的context、address space（这其实是一个separation机制），它能够保证一个control path在被suspend后，过后能够被resume。

显然context包括每个control path的private数据，如下：

- call stack



## Context switch

需要注意的是，本节所述的context switch是广义的，而不是[Computer multitasking](https://en.wikipedia.org/wiki/Computer_multitasking)中专指task（process/thread）的[context switch](https://en.wikipedia.org/wiki/Context_switch)。



发生context switch的场景：

### Scheduler触发Process Switch

3.3. Process Switch

kernel substitutes one process for another process

### Interrupt Signals触发Switch

4.1. The Role of Interrupt Signals

> the code executed by an interrupt or by an exception handler is not a process. Rather, it is a kernel control path that runs at the expense of the same process that was running when the interrupt occurred



> As a kernel control path, the interrupt handler is lighter than a process (it has less context and requires less time to set up or tear down).

4.3. Nested Execution of Exception and Interrupt Handlers



思考：发生context switch的时候，要把context置于何处呢？



## 总结

通过control path模型我们可以看到，OS在运行和控制它们的时候会面临中类似的问题。