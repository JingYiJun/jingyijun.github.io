+++
title = "Rust 异步运行时实战"
date = "2024-02-13T20:07:00+08:00"
lastmod = "2024-02-13T20:07:00+08:00"
math = false
hidden = false
comments = true
draft = false
categories = ['随笔']
tags = ['Rust', 'async']
+++

# Rust 异步运行时实战

项目仓库：[JingYiJun/async-runtime (github.com)](https://github.com/JingYiJun/async-runtime)

## 同步与异步

首先我们来聊一下**异步**的编程思想。

异步的思想实际是与外部资源访问、读写是息息相关的。在执行指令的 CPU 之外，一台计算机也有着磁盘、网络设备等外围 IO 设备，一部分 IO 设备具有独立的控制器，可以访问磁盘上的文件并复制到特定的内存区域；在程序执行之下也有操作系统的支撑。

当一个程序对外围 IO 设备进行访问，例如使用 `open`，`read`，`write` 等 Linux 系统调用访问文件，程序会陷入（trap）操作系统，操作系统指挥外围 IO 设备将数据复制到特定的内存区域，复制完成之后，再返回程序继续执行处理逻辑。这个过程可能大量的时间是花在了读写磁盘，程序在这一段时间内即使有其他的事情可以做，也没发做，浪费了很多的程序时间。这样的访问就是**同步**访问。

另一种情况，如果程序通知 IO 设备读取数据，可以立刻返回做其他事情，等到无事可做的时候等待 IO 设备的响应，IO 设备主动通知程序可以处理数据了，程序继续处理数据，这样的访问就是**异步**访问。

因此，**同步**、**异步**，实际上是程序**读取设备数据**的步骤不同。同步发出访问指令后，等待直到访问数据返回；异步发出指令后，不等待，而是数据完成后通知程序继续执行。

另一个概念是**阻塞**、**非阻塞**，这是描述一个操作是否会暂停当前执行流。阻塞的访问指令发出后，会等到访问数据返回再解除阻塞状态；非阻塞的访问指令发出后，不会等待数据返回，而是直接返回程序。

因此同步与异步、阻塞与非阻塞虽然是两个维度的概念，但是也息息相关。Rust 中的异步协程其实就是一种异步非阻塞的程序流。

## 协程：有栈与无栈

协程是为了实现异步思想提出的一种模型，可以理解为是一段代码片段，或者说一种特殊的函数。程序本身可以在调度器的调度下，从一段协程代码/函数的开头开始执行；执行到中间的某个地方，由于发起了异步非阻塞的读取数据请求，暂停当前程序并由调度器执行其他代码片段。当读取数据的请求就绪的时候，调度器再调度这个协程，从之前暂停的地方继续执行。这样的暂停被称为挂起（`suspend` 或者 `yield`）。

要实现协程的调度和切换，需要知道如何保存并恢复协程的执行状态。

要保存协程的执行状态，最直观的方法就是保存程序当前运行的 **CPU 状态和栈状态**。如果程序执行到某处，把当前所有寄存器、栈指针、程序计数器存下来，再换入另一套寄存器、栈指针、程序计数器，就完成一次协程的保存和恢复。这样的保存要求每个协程都有独立的虚拟栈空间（实际是在堆上分配的），也要有独立的结构存放寄存器、栈指针、程序计数器等。这样的协程被称为**有栈协程**，例如 Go 语言、Linux 操作系统内核用的就是这样的方式管理协程。

另一种保存协程状态的方法是将协程分解成多个片段，每个片段是一个独立的函数，协程就好比一个**状态机**，根据当前执行状态切换不同的函数，协程只需要把跨不同独立函数的变量存储起来就可以了。程序的执行就是在顺序执行一个一个独立的函数，等到最后一个函数执行完毕，协程也就结束了。这样的协程没有独立的运行栈，而是使用主线程栈运行，这样的协程称为**无栈协程**，Rust、C#、C++20、Kotlin 就是这样的无栈协程。

这两种方案各有优劣。

- **有栈协程**：在实现上需要借助汇编语言、或者 libc 中的一些特殊函数存储寄存器、栈信息，但是每个协程结构所占空间是固定的，方便手动实现协程的管理结构；有栈协程无论协程内部的函数调用有多少层，无论协程执行到哪一条指令，挂起协程的流程和开销是统一的，因此针对一个长时间占据 CPU 时间的协程，当有新的 IO 请求到来，可以由操作系统发出抢占的信号，在协程执行到任意一条指令时挂起协程并切换到另一个协程；但是当函数调用栈过深的时候，可能出现给定的虚拟栈空间不足的情况，需要做栈扩容；当函数调用栈比较浅，也容易出现内碎片的问题；每次函数调用都要检查是否发生栈溢出，这样的开销是不可忽视的。
- **无栈协程**：实现最好有编译器的辅助，计算一个协程保存信息的结构的大小；提取所有跨挂起点使用的变量，这样的操作实际是很繁琐的机械化的操作，编译器通过语法树生成对应的状态机结构是很合适的。由于没有独立的栈，每次挂起时协程函数都要返回到最上层；异步函数栈深度越深，挂起点挂起时要返回到最上层的成本也越高。无栈协程没法实现有栈协程类似的基于信号的强制抢占机制，不能在函数的任意位置挂起协程，只能在协程特定的挂起点挂起协程，这要求协程的编写者需要估计挂起点之间的执行时间，如果挂起点之间有过长的处理时间，可能会导致新的IO访问需求无法及时处理。

总结来看，无栈协程不需要独立的栈空间，编译时即可计算出状态空间大小，更节省内存资源；但是对于 IO 密集性的应用，无栈协程无法在任意位置挂起程序执行流，可能导致 CPU 和 IO 资源分配效率降低。有栈协程需要独立栈空间、动态检查栈溢出等特点，导致协程的运行性能相比无栈协程低，但是如果调度得当，可以比无栈协程有更高的 CPU / IO 分配效率，这对于云原生、微服务时代的事件驱动服务提高响应速度、降低延迟至关重要。 

## Rust 协程

**Rust 协程**是一种**无栈协程**，Rust 编译器将 `async fn` 结构的函数，`async`闭包或者 `async` 块转为无栈协程的结构，即自动实现了 `Future` trait。以下的协程仅指 Rust 协程。

Rust 标准库中只定义了协程最基础的数据结构 `Poll` 和 `Future`，封装唤醒任务的 `Waker` 结构，协程的状态上下文 `Context` 结构，而协程在何时挂起、由谁来唤醒、如何唤醒、由谁来调度执行、如何调度执行，这些具体实现在标准库中并没有实现，而是交给了不同的第三方库实现，例如 `tokio`、`async-std`、`smol` 等。

### Poll 和 Future

Rust 中用 `Poll` enum 包装的协程对外部调用者展示的状态：

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

当为 Ready 时，表示协程已经完成，并且返回具体的值；当为 Pending 时，协程从中间挂起，交由调度器调度。

Rust 协程重点是 `Future` trait：

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

在 `async fn` 中对一个 `Future` 执行 `await` 时，实际上执行的就是 `fn poll` 函数。因此如果要自己实现一个异步运行时，需要自己定义一系列的结构，实现 `Future` trait，在 `poll` 函数中定义如何唤醒，最后在 `async fn` 中获取结构并调用。

### Pin

什么是 Pin？字面意思，把一个结构钉在一个地方，不允许移动。

那为什么 future 要 Pin 住，不能移动呢？

举个例子：

```rust
async fn test() {
    let a = 1;
    let ra = &a;
    delay(100).await;
    println!("{}", ra);
}
```

这样的一个异步函数，如果生成一个状态机结构（不关心生命周期），将会是：

```rust
struct TestFuture {
    state: usize,
    a: i32,
    ra: &i32,
}

impl TestFuture {
    fn state1(self: &mut Self) {
        self.a = 1;
        self.ra = &a;
    }
    
    fn state2(self: &mut Self) {
        println!("{}", self.ra);
    }
}
```

注意看，在一个结构中产生了**自引用结构**，即 `ra` 引用了 `a`，获取到了 `a` 的地址。当这个结构 move 的时候，`ra` 将指向原先结构的地址，导致悬垂指针错误。因此对于实现了 `Future` 的结构，需要用 `Pin<impl Future>` 保障内存安全。

### Context 和 Waker

`Future` trait 中还有一个重要的结构是 `Context`，这个结构目前只存储了 `Waker` 结构，以获取协程任务唤醒时数据结构。

一个 `Waker` 结构最重要的函数有两个：

```rust
pub fn wake(self)
pub fn wake_by_ref(&self)
```

这两个函数分别表示消费所有权和不消费所有权状态下唤醒一个协程的方法。常规的运行时实现中，协程在运行时定义的执行器 `Executor` 中管理，wake 系列函数即通知执行器可以继续 `poll` 这个协程。而当协程需要挂起之前，协程会把这个 `Waker` 结构交给反应器 `Reactor` 管理，`Reactor` 则会监听 IO 情况和其他事件，在必要的时候调用 `waker` 的 `wake` 或者 `wake_by_ref` 方法。

持续更新。。。

参考：

[https://night-cruise.github.io/async-rust](https://night-cruise.github.io/async-rust)