# goroutine与线程的区别？
goroutine是可并发执行的最小的Go的实体，是建立在线程之上的轻量级的抽象。相比线程，goroutine的创建和销毁的代价要小得多，并且它的调度是独立于线程的。
优点：
- 内存消耗：goroutine所需要的内存通常只有2KB（如果栈空间不够，会自动扩容，最大1GB），而线程要1MB。
- 创建与销毁：线程的创建和销毁会进入os内核态，因此开销比较大（通常的解决办法是线程池）。但是goroutine的创建和销毁都是go自行管理，它的调度是在用户态完成的，开销更小。
- 切换：线程的调度方式是抢占式的，发生线程抢占时，在线程切换过程中需要保存/恢复寄存器信息。（如通用寄存器、Program Counter、Stack Pointer、段寄存器等）而goroutine的调度是协同式的，在进行切换时，只需要很少量的寄存器的保存和恢复（Program Counter、Stack Pointer和Base Pointer），因此切换效率更高。

# goroutine的生命周期？（存疑）
- Waiting：此时goroutine已停止，可能在等待系统调用或者同步调用。
- Runnable：表示goroutine可以执行。此时这个G可能在某个P的队列里，也可能在全局的队列里。
- Executing：表示G已经在M上执行。
