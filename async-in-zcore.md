## 异步的 zCore

### 1. 概述

随着处理器性能的提升，程序局部性对于性能的重要性进一步提升，因此如何进一步提升程序局部性愈发成为一个值得研究的问题。zCore 设计核心出发点：利用异步特性增加 OS 局部性最终提升 OS IO 吞吐量。

zCore 主要特征体现在:

* 类 io_uring 异步系统调用。类似 io_uring，用户程序可以通过一个 syscall 获得一块与内核共享的内存，称为 async-syscall-channel （下称 AS-channel）, 可以通过该 channel 来提交 syscall 请求（下成异步系统调用，简称 syscall），而不用真的触发 syscall 异常陷入内核。AS-channel 格式亦类似 io_uring，包含一些元数据以及一个 submission queue，一个 completion queue，SQ 中一项对应一个 async syscall 请求，CQ 中一项对应一个 async syscall 的结果。用户程序通过 SQ 提交请求，通过 CQ 得到返回值，通过内存 fence 指令与内核同步，最终实现无异常系统调用。
* 内核模块及其接口的高度异步化。zCore 充分利用 rust async/await 语法特性，使得内部模块设计高度协程化，以协程的形式执行内核任务而不是线程。以文件系统为例，zCore 引入了异步磁盘驱动，异步文件系统，异步的内核文件模块以及异步系统调用接口，这里异步的含义是：这些模块的接口函数都是 async 函数，可以以协程的形式被调用。zCore 异步的最终基石是中断，磁盘中断支持了异步磁盘驱动，进而支持了异步的文件系统和文件模块，最终封装得到异步系统调用接口。实际文件读写过程中，对应协程会在发出磁盘请求后 Pending，并在中断处理例程中被唤醒，这一过程类似基于内核线程的 linux 文件读写，不同之处在于 zCore 内部这一过程以协程的形式进行，进而获得更小的切换开销和更小的内存占用。
* 提供保证机制，避免异步延迟累积导致关键事件内核相应延迟过大。zCore 在充分利用协程机制的同时，并没有严重损害内核的相应延迟。首先：zCore 结合线程机制实现了可抢占协程：当内核中出现需要立即相应的高优先级事件时，zCore 会将当前执行的协程退化为线程并退出，而后创建并立即执行一个对应高优先级事件的协程，这一过程可以嵌套进行。这一机制保证了内核可感知事件处理的延迟。此外，利用 x86_64 用户态中断机制，可以将本身不可感知的异步系统调用的提交（写共享内存）转化为可感知的中断事件。

### 2. 具体实现

本届分别就异步系统调用和异步化内核模块作出解释，延迟保证机制会分散在这两部分中并在下一节中作出总结。本届最后，会介绍与内核调度有关模块。

#### 2.1 类 io_uring 异步系统调用

![img](https://pic3.zhimg.com/80/v2-549f6af4518938aed8f7065d2c5d1022_720w.png)

用户程序通过 AS-channel 提交系统调用请求、处理系统调用结果的过程很类似 io_uring。用户程序首先通过一个 syscall (1.1.1小节) 建立一块共享内存，并通过返回的信息得知这块内存的布局。用户程序发出系统调用时，需要在 SQ 末尾附加一项，然后将 tail 加一，系统调用的提交就算完成，可以立即返回。当内核发现 SQ head < tail 后，即识别到有尚未执行的 syscall 请求，内核会产生处理 head 对应的 syscall的协程，而后 head 加一，直到 head 与 tail 相等。当处理 syscall 的协程执行完毕后，内核会将返回结果附加在 CQ 末尾，然后 CQ tail 加一。用户程序则通过比较 CQ head 和 tail 来处理 syscall 返回结果。

整个过程中用户程序不会真的陷入到内核，实现了一个类似 IPC 的 syscall 提交和完成的机制。与 io_uring 的区别在于：

* AS-channel 可以接受的 syscall 种类更多，除了 `sys_yield` `sys_sleep` 等进程退出的 syscall，以及 `sys_fork` 等几个特殊 syscall，其他 syscall 都可以通过 AS-channel 提交（当然，这一点 io_uring 也可以实现）。
* io_uring 内部通过线程来处理系统调用请求，而 zCore 内部使用协程处理，这使得在 syscall 提交数量很大的时候，zCore 的内存开销和 context switch 开销相对较小。
* 相比 linux io_uring，AS-channel 任务在内核有更加灵活的调度，不同的任务会被路由到不同的内核核处理，使得总体局部性更优。

##### 2.1.1 相关数据结构与 syscall 接口

建立 AS-channel 系统调用接口如下：

```c
/* 
* Set up an AS-channel
* @para sub_capacity: SQ entry number
* @para comp_capacity: CQ entry number
* @para info: the layout of AS-channel
* @para info_size: the size of `info`
*/
int sys_setup_async_call(int sub_capacity, int comp_capacity, struct async_call_info *info, size_t info_size);
```

用户可以指定 SQ, CQ 的大小，一般而言，SQ 大小为用户程序最多并法的 syscall 数量，而CQ entry 数量应当不大于 SQ。

AS-channel 具体布局如下：

```rust
#[repr(C)]
#[derive(Debug)]
pub(super) struct Ring {
    head: AlignCacheLine<u32>,
    tail: AlignCacheLine<u32>,
}

#[repr(C)]
#[derive(Debug)]
pub(super) struct AsyncCallBufferLayout {
    sub_ring: Ring,
    comp_ring: Ring,
    sub_capacity: u32,
    sub_capacity_mask: u32,
    comp_capacity: u32,
    comp_capacity_mask: u32,
    sub_entries: AlignCacheLine<[RequestRingEntry; 0]>,
    comp_entries: AlignCacheLine<[CompletionRingEntry; 0]>,
}
```

分别包含元数据以及 SQ CQ两个队列，对于可能在不同核之间同步的数据，确保 cache 对其以避免 false sharing。其中 `sub_capacity_mask` 是用来实现环形队列的 index mask。

 SQ、CQ entry 具体格式如下：

```rust
#[repr(C)]
#[derive(Debug)]
pub(super) struct SubmissionRingEntry {
    pub syscall_id: u64,
    pub arg0: u64,
    pub arg1: u64,
    pub arg2: u64,
    pub arg3: u64,
    pub arg4: u64,
    pub arg5: u64,
    pub flags: u32,
    pub user_data: u32,
}

#[repr(C)]
#[derive(Debug, Default)]
pub(super) struct CompletionRingEntry {
    user_data: u32,
    result: i32,
}
```

其中 `user_data` 用来建立 SQ, CQ 之间的一一对应，当内核处理完毕 SQ 的一个请求后，会把 `user_data` 拷贝到 CQ 对应项中。这也意味着，SQ 中请求的完成是乱序的，目前尚无机制保证存在依赖的 syscall 的执行顺序，但后续可以实现。

##### 2.1.2 异步 syscall 识别与返回值提交

目前的实现中，异步 syscall 任务的识别和提交依赖于内核 / 用户的轮询。具体而言，每个用户线程，都会在内核中对应一个始终处于可运行态但优先级最低的 checker 协程，该协程每次执行都会检查 SQ 中 head 与 tail 是否一致，并在不一致时产生新协程处理新提交的 syscall 请求，之后 yield。也就是说内核会在空闲的时候检查用户是否产生了新的 syscall 提交，避免频繁检查导致的无意义 CPU 开销。用户程序同样需要轮询 CQ 来检查 syscall 是否执行结束。

如果用户程序要求低延时的 syscall，有三种解决方案。其一，使用普通的同步 syscall，这与异步 syscall 并不冲突。其二，使用高优先级 syscall，异步 syscall 可以额外增加一个优先级的属性，当内核识别到高优先级 syscall 后，会尽快执行（一般会在 checker 协程结束后马上执行）。其三，第二种方法无法满足要求，且不希望陷入内核破坏局部性，可以使用 x86_64 用户态中断特性，与内核建立用户态中断的连接，在发出高优先级 syscall 之后发出一个用户态中断，这会使得内核停止执行当前协程，并立即执行 checker 协程，进而快速处理高优先级 syscall。

> intel x86_64 处理机新提出的 UINTR 特性是一个可以在用户态发出的、轻量级的 IPI。虽然设计目标是用来在用户态和用户态之间传递中断，但内核可以将其当成一个普通中断处理，这就使得用户态中断可以由一个用户核发送向一个内核核，反方向亦然。

##### 2.1.3 用户态程序的修改

异步系统调用对用户程序最大的改变在于，传统的同步的系统调用需要改为异步的。对于一些天生具有异步特征的 workload，如数据库、网络服务程序，这些程序本身就有自己基于事件的异步机制，可以将 syscall 完成当成一个特殊的事件，改造比较容易。对于传统的同步程序或者基于线程实现异步的程序，可以通过修改线程库，在一个线程发出 syscall 之后将其阻塞，转而运行另一个线程，以此尽力挖掘用户程序并法 syscall 的能力。后者方法已经在前人工作中被证明有效。

#### 2.2 异步内核模块

本节首先简单介绍了 rust async 机制，而后以文件系统为例，从异步驱动开始，自底向上简单介绍完成一次异步 syscall 会经过的步骤。总体而言，除了使用协程替代线程之外，zCore 的运行逻辑与 Linux 等传统内核区别并不大。

##### 2.2.1 Rust async 语法特性简介

`async` `await` 语法是 rust 提供的一种以同步风格写异步代码的方式。一个普通函数被标注为 `async` 后，编译器将会将其转化为一个 generator（一个包含状态的函数，可以被多次调用，本质是一个有限状态自动机，每次调用相当于尝试进行一次状态转移）。调用 `async` 函数会返回一个 `Future` 结构体，也就是一个包含状态转移函数和初始状态的 generator，一个 `Future` 被 await 后

##### 2.2.1 协程内存开销以及切换开销



##### 2.2.2 异步 virtio 驱动

使用 rust async 语法可以很方便的写出协程程序，唯一需要特殊处理的是叶 future，而文件系统中的叶 future 是磁盘驱动相应接口。这些接口不会使用 async 标注而是会直接返回一个 Future 类型。

```rust
/// 读一个块，返回一个 future.
pub fn read_block(
    self: &Arc<Self>,
    block_id: usize,
    buf: &mut [u8],
) -> Pin<Box<BlkFuture<'a>>> {
    let req = BlkReq {
        type_: ReqType::In,			// 表示读磁盘
        sector: block_id as u64,	// 读取的块ID
    };
    let mut inner = self.inner.lock(LockChannel::Normal);	// 或者对应数据结构的锁，防止访问冲突
    let mut future = BlkFuture::new(self);  	// 返回值, 由于 future 注册 waker 需要访问驱动管理模块的数据，这里传入 self 指针
    match inner
    .queue
    .add(req, buf, future.resp)			// add 操作会发出磁盘读请求，注意 futrue.resp 为请求完成后磁盘会存放结果的内存位置
    {
        Ok(n) => future.head = n，	   // 记录中断时返回的 des 编号与 future 的对应关系
        Err(e) => future.err = Some(e),
    }
    future
}

/// future poll 接口实现
impl Future for BlkFuture<'_> {
    type Output = Result;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        let mut driver = self.driver.inner.lock(LockChannel::Normal);
        match self.resp.status {		// 检查 resp 来判断是否磁盘操作已经完成
            RespStatus::Ok => Poll::Ready(Ok(())),		// 已经完成，返回 Ready
            RespStatus::_NotReady => {					// 尚未完成，返回 Pending 并注册 waker，方便在处理磁盘中断时唤醒该 future
                driver.register_waker(self.id, cx.waker);
                Poll::Pending
            }
            _ => Poll::Ready(Err(Error::IoError)),
        }
    }
}

/// 中断处理函数
pub fn handle_irq(self: &Arc<Self>) -> Result {
    let mut inner = self.inner.lock(LockChannel::Interrupt);
    let idx = inner.queue.pop_used()?;		// 获得中断对应的 des id
    inner.wake_future(idx)?;				// 唤醒对应 future
    Ok(())
}
```

一般而言，调用磁盘读写后会立即 poll 或者的 future，第一次 poll 一般没有完成（时间太短，磁盘操作一般还未完成）就是注册 future 并返回 Pending 将该协程置入休眠队列，指导发生磁盘中断，读取到完成的描述符 id，进而找到完成的协程并通过注册的 waker 唤醒。

###### 双相锁

这段代码中还需要注意的是该驱动使用的锁。一般而言，对于中断处理需要访问的数据，必须使用关中断锁来防止死锁或者数据冲突，但是在仔细分析中断和非中断态下使用的数据后，我们发现此二者冲突并不大。具体而言，两种状态仅有两种数据冲突： (1)几个 counter 上存在写写冲突，且都是计数增加，不需要保证顺序 (2) 在几个 index 上存在读写冲突（一者写一者读），而这两种冲突都可以使用原子变量结果。因此，即便在非中断态获得了锁，在中断态中再次获得锁不会导致数据冲突，故而我们设计了一种双相锁，中断态和非中断态分别可以由一个核或者锁，且获得锁不需要关中断。

究其原因，是因为该 virtio 驱动包含的数据有两种状态，OS态和硬件态。处于 OS 态的数据在非中断态使用后变为硬件态，作为硬件的输入输出。而硬件态的数据在硬件完成操作后需要在中断处理中回收为 OS 态。所以非中断态只会使用 OS 态数据，而中断处理只会使用硬件态数据，不多的例外就是一些 counter 以及必要的 index。

###### 其他异步模块

在有了异步驱动之后，文件系统的异步化已经 syscall 接口的异步化就容易多了，大部分情况下，只需要让异步特性自然传播即可，重构工作主要是增加 async 和 await 标记以及更新同步互斥相关的实现（主要是锁），值得探讨的问题并不多。最终我们将得到如下与文件系统相关的 async syscall 接口。

```rust
/// Reads from a specified file using a file descriptor. Before using this call,
/// you must first obtain a file descriptor using the opensyscall. Returns bytes read successfully.
/// - fd – file descriptor
/// - base – pointer to the buffer to fill with read contents
/// - len – number of bytes to read
pub async fn sys_read(&self, fd: FileDesc, mut base: UserOutPtr<u8>, len: usize) -> SysResult {
    let file_like = self.proc.get_file_like(fd)?;
    let len = file_like.read(&mut base.into()).await?;
    Ok(len)
}
```

#### 2.3 内核协程调度

内核调度是保证良好局部性的核心所在，糟糕的调度顺序会使得协程的局部性被破坏，同时，调度对于高优先级任务的及时响应也至关重要。本节首先简介 zCore 内存模型以及切换开销，然后结合 zCore 内部协程的不同种类，介绍 zCore 调度对高优先级任务的支持以及在局部性方面作出努力。 

##### 2.3.0 内存模型与地址空间切换开销

这里简单介绍 zCore 内存模型。为了避免熔断漏洞，zCore 在用户态与内核态切换时会切换页表，内核页表包含内核映射和用户映射，用户页表则只包含用户映射。在处于不同地址空间的协程之间切换时，zCore 需要更换页表，因此需要刷新 tlb。然而 riscv SV32 / SV39 叶表支持 Global 标志，在刷新 tlb 时，带 Global 标记的页表项不会被刷新，zCore 利用这一点，将内核地址全部置为 Global。因此内核协程之间的地址空间切换开销相对较小，而在内核和用户之间切换开销较大。下一节中我们将看到，zCore 中大部分的切换属于内核协程之间的切换。

##### 2.3.1 内核内部的协程类型

zCore 内核中会存在两类协程任务，一类为内核协程，对应处理用户远程系统调用产生的协程以及 checker 协程，另一类为封装成协程的用户线程，称之为用户协程，这类协程被执行就会进入用户态执行用户线程，仅为了处理的统一性将其封装为协程。从上一章我们知道仅涉及内核协程的切换开销较小，而设计用户协程的切换开销较大，因此为了尽量使得打开销较少，zCore 调度器倾向于将内核协程集中在若干个核上（成为内核核），而将用户协程分配在其他核上（成为用户核）。对于内核核，虽然切换频繁，但无论 context switch 切换开销还是地址空间切换开销都较小，对于用户核，由于异步系统调用会尽力挖掘用户程序执行潜能，只会在时间片耗尽以及完全处于空闲状态时才会切换到其他进程，切换次数很少。综合来看，zCore 的切换开销都比较低。

##### 2.3.2 可抢占协程

协程在取得占用内存少，切换开销小优势的同时，其最大的问题在于不可抢占性。在没有线程机制的前提下，发生中断后，一个协程的退出至少需要等到下一个退出点，这期间所需的时间是不确定的，这远远无法满足对 latency 要求较高程序的要求。因此 zCore 结合线程机制实现了可抢占协程，为 zCore 实时性作出一定保障。

理想情况下，zCore 一个核上只有一个线程，其上运行一系列协程，只有协程之间的切换。但 zCore 允许一个协程在碰到中断或者执行特殊函数时（这二者分别对应中断 / 内核事件引起高优先级任务就绪）将正在执行的协程退化为线程，新创建一个线程并立即执行高优先级事件，处理完毕之后回归到之前协程的执行。

### 2. 其他

本节总结 zCore 现有的应对 latency 敏感应用 / syscall 的方法，然后分析 zCore 后续可能的改进方向。 

##### 2.1 Lantency 分析

一般而言，异步模型并不适合 Latency 敏感的应用场景，因为各个异步链路之间的延迟会累加，导致总体延迟不可控。zCore 中异步系统调用的异步链路主要有三个：(1) 用户提交 syscall 到内核识别; (2)中断发生到内核响应; (3)内核提交返回值到用户识别。其中 (1) (3) 的传播延迟可以使用用户态中断消除，代价是引入两次内核线程切换，(2) 的传播延迟可以使用抢占协程消除，代价同样是引入两次内核线程切换。而对于尚不支持用户态中断的处理器，(1) (3) 的延迟只能通过退化为同步 syscall 的方式保证。

##### 2.2 内核专用的其他思考

* 进一步核心专用化

  除了将核区分为内核核和用户核，在核心数量允许和应用场景合适的情况下，zCore 可以进一步发挥核专用化的思想。比如，可以将所有磁盘 IO 有关的任务全部分配给一个核，该核心可以一直持有相关的锁和很热的 cache，以此来获得局部性的进一步提升。这就要求 checker 协程在产生携程任务时进行合适的路由，将对应的 woker 携程置于对应核心所属线程的调度队列中。

* load balance 与调度

  核心专用化在提高局部性的同时，给负载均衡带来了麻烦，一方面时用户核与内核核之间的均衡，另一方面是不同任务的内核核之间的均衡。zCore 需要引入合理的负载度量机制，动态的在两个维度上进行负载均衡的调度。



## 3. 测试

* coroutine

```shell
> sudo perf stat -e dTLB-load-misses,iTLB-load-misses,cs,cache-misses,sched:sched_switch ./coroutine_switch
 
 TIMES 10000000 delta1 0.000000079 seconds delta2 0.000000079 seconds

 Performance counter stats for './target/release/coroutine_switch':

             8,456      dTLB-load-misses                                            
             3,509      iTLB-load-misses                                            
                43      cs                                                          
           188,592      cache-misses                                                

       0.796615966 seconds time elapsed

       0.791792000 seconds user
       0.007997000 seconds sys
```

* proc

```shell
> sudo perf stat -e dTLB-load-misses,iTLB-load-misses,cs,cache-misses,sched:sched_switch ./proc_switch

 F TIMES 10000000 delta 0.000001677 seconds
 C TIMES 10000000 delta 0.000001677 seconds

 Performance counter stats for './target/release/proc_switch':

        20,008,657      dTLB-load-misses                                            
        21,090,368      iTLB-load-misses                                            
        20,000,006      cs                                                          
         3,549,111      cache-misses                                                

      16.774689969 seconds time elapsed

       3.130511000 seconds user
       5.271587000 seconds sys
```

* thread

```shell
> sudo perf stat -e dTLB-load-misses,iTLB-load-misses,cs,cache-misses,sched:sched_switch ./thread_switch

 Sched Policy: SCHED_FIFO
 Run in cpu #8
 TIMES 10000000 delta1 0.000001498 seconds delta2 0.000001498 seconds

 Performance counter stats for './target/release/thread_switch':

        20,011,649      dTLB-load-misses                                            
        22,897,374      iTLB-load-misses                                            
        20,000,007      cs                                                          
         1,620,228      cache-misses                                                

      14.992523193 seconds time elapsed

       6.383470000 seconds user
       8.607285000 seconds sys

```



|  Insize   |    bs     | entry | time |
| :-------: | :-------: | :---: | :--: |
| 0x1000000 |  0x1000   |   -   | 1109 |
| 0x1000000 |  0x1000   | 1024  | 1112 |
| 0x1000000 |  0x1000   |  256  | 1108 |
| 0x1000000 |  0x1000   |  64   | 1133 |
| 0x1000000 |  0x1000   |  16   | 1139 |
| 0x1000000 |  0x1000   |   4   | 1202 |
| 0x1000000 |  0x1000   |   2   | 1261 |
| 0x1000000 |  0x1000   |   1   | 1369 |
| 0x1000000 | 0x1000000 |   -   | 1088 |
|           |           |       |      |
|   65536   |    32     |   1   | 104  |
|   65536   |    32     |   4   |  64  |
|   65536   |    32     |  16   |  58  |
|   65536   |    32     |  64   |  55  |
|   65536   |    32     | 2048  |  56  |
|   65536   |   65536   |   -   |  4   |
|   65536   |    32     |   1   | 130  |
|   65536   |    32     |   4   |  73  |
|   65536   |    32     |  16   |  60  |
|   65536   |    32     |  64   |  59  |
|   65536   |    32     | 2048  |  61  |
|   65536   |   65536   |   -   |  4   |
|           |           |       |      |
|           |           |       |      |
|           |           |       |      |
