# 这里没有线程

原文地址：[https://blog.stephencleary.com/2013/11/there-is-no-thread.html](https://blog.stephencleary.com/2013/11/there-is-no-thread.html)

## 前沿

我是在看 C#8.0 新特性异步流时在评论里看到这篇文章的，阅读之后发现这篇文章干货满满，作者解释的非常清晰，里面的本质分析内容在《CLR via C#》一书中也有讲到。更加加深了我的印象。遂在这里翻译过来，以便加深自己的理解

## 正文

一个本质的事实就是纯粹的异步是不会产生线程的

反对这个事实的人有很多。“不”，他们喊道：“如果我正在等待一个操作，那么这个线程就必须在执行等待！它可能是线程池线程。或者是一个操作系统（OS）线程，又或是其他设备驱动程序...”

无需理会他们。如果一个异步操作是纯粹的（pure），那么就不会有线程的。

那些持怀疑态度的人。那我们就迎合他们罢。

我们将跟踪一个异步操作一直到硬件，特别是 .net 部分和硬件部分。我们必须通过省略一些中间的细节来简化描述，但是我们不能偏离事实真相。

通常写一个异步操作（文件，网络流，USB 接口等等）。代码如下：

```c#
private async void Button_Click(object sender, RoutedEventArgs s)
{
    byte[] data = ...
    await myDevice.WriteAsync(data, 0, data.Length);
}
```

我们已经知道在 `await` 的时候 UI 线程是不会阻塞的。那么问题来了：这里有没有是其他线程在阻塞期间牺牲自己以至于让 UI 线程存活呢？

那让我们继续往下深究

第一步：库（比如进入到 BCL 库源代码）。我们假使 `WriteAsync` 是通过 [http://msdn.microsoft.com/en-us/library/system.threading.overlapped.aspx](P/Invoke 在 .net 异步的标准实现) 来实现的，它是基于overllapped I/O 的。所有它在一个设备驱动程序的句柄上开始一个 Win32 overlapped 操作。

> OVERLAPPED 是一个包含了用于异步输入输出的信息的结构体；详细解释移步 [https://en.wikipedia.org/wiki/Overlapped_I/O](https://en.wikipedia.org/wiki/Overlapped_I/O)

操作系统然后就会转向设备驱动程序并开始请求一个写操作。它首先会构造一个表示写请求的对象；它被称为 I/O Request Packet(IRP)。

设备驱动程序接受到 IRP 之后并向设备提交一个写出数据的命令。如果这个设备支持直接内存访问（DMA 全称是 Direct Memory Access），这能够像写到缓冲区到寄存器一样简单。这就是设备驱动程序所做的一切；它把 IRP 标记为 "pending" (挂起) 并返回给操作系统。

![](../../images/Os1.png)

本质核心就在这里：设备驱动程序正在处理 IRP 时是不会允许阻塞的。也就是说**如果 IRP 不能马上完成，那么它必须要异步处理**。甚至是同步 API 也是如此！在设备驱动程序级别，所有的请求（重要的）都是异步的。

> 这里引用了 [Tomes](http://www.amazon.com/gp/product/0735648735/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=0735648735&linkCode=as2&tag=stepheclearys-20) 的[知识](http://www.amazon.com/gp/product/0735665877/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=0735665877&linkCode=as2&tag=stepheclearys-20)，“无论I/O请求的类型如何，在内部，代表应用程序向驱动程序发出的I/O操作都是异步执行的”

在 IRP 挂起的时候，OS 返回库，库返回了一个未完成的任务给按钮点击事件，并暂停了 async 方法， UI 线程继续执行。

我们跟着请求继续往下走，现在到达了设备的物理层

现在写操作正在进行。那么有多少线程正处理它呢？

没有。

这里没有设备驱动程序线程、OS 线程、库（BCL）线程或者是线程池线程操作写操作。**这里没有线程**。

现在我们来跟着从来自内核的相应回到最初的世界。

在开始写请求之后的一段时间，设备完成了写操作。它会以中断的方式来通知 CPU。

设备驱动程序的中断服务程序（ISR(Interrupt Service Routine) ）响应中断。这个中断是 CPU 级别的事件，无论哪个线程正在运行都会临时的抢占 CPU 的控制权。你可以认为 ISR 是在“借”当前正在运行的线程，但是我更倾向于 ISR 运行时的级别非常低，以至于不存在“线程”的概念。可以这么说，它们在所有线程之下进来的。

不管怎样，ISR 正确写完了，完了它会通知设备程序 “谢谢你的中断” 并且进入 DPC（Deffered Procedure Call） 队列（延迟过程调用）

当 CPU 被中断干扰时，它将会到达 DPCs。DPCs 也会执行在一个很低的级别以至于说它是一个线程是不正确的；就像 ISRs，DPCs 直接在 CPU 上运行，在线程系统之下。

PDC 接受代表写请求的 IRP 并且标记为 “已完成”。然而，这个“完成”状态只存在于 OS 级别；进程有它自己的内存空间，它必须被通知。所以 OS 会入队列一个特殊内核模式异步过程调用（APC）到拥有自己句柄的线程。

由于 BCL 库使用了标准的 P/Invoke  overlapped I/O 系统，它已经在 I/O Completion Port（IOCP）注册句柄，它是线程池的一部分。所以借用 I/O 线程池线程来执行 APC，它会通知这些任务已经完成了。

这个任务已经捕捉了 UI 上下文，所以它不会直接在线程池线程上恢复异步方法。而是它将该方法的延续排队到 UI 上下文中，并且 UI 线程将恢复执行那个方法。

所以我们看到，正当一个请求处理时这里是没有线程的。当请求完成时，一些线程被借过去或者是被短暂的排队。这项工作通常在 1 毫秒左右（例如 APC 运行在线程池线程）或 1 微妙左右（例如 ISR）。但是这里没有线程是阻塞的，仅仅只是等待请求完成。

![](../../images/Os2.png)

现在，我们遵循的路径是标准路径，这是如此清晰简单。这里有无数的变量，但是核心是不变的。

所以说 “这里必须有一个线程是在处理异步操作” 是不正确的。

释怀吧，不要尝试找到异步线程——这是不可能的。而是你应该去了解真相：

**没有线程**

> 该篇文章的评论也是很精彩的，特别是讨论将异步操作当作消息处理那部分讨论，建议也花时间看下

