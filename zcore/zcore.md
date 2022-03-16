---

marp: true
size: 4:3
piginate: true
---
<!-- paginate: true -->

# zCore 简介

// TODO： reporter
2022.3.16

---

# 提纲

* 整体介绍

* 主要技术：用户态 OS，异步的 zCore

* Fuchsia 与 Zircon 内核对象简介

---

# zCore: A Next Gen Rust OS!

Rust 语言编写的“混合”操作系统内核

* 同时支持 Linux 和 Zircon 系统调用

* 同时支持 LibOS 和 裸机 OS 形式

  可以完全在用户态开发、测试、运行

* 符合 Rust 风格，项目模块化

* 使用 Rust async 机制，内核协程化

* 充分发挥异步特征，重新思考 syscall 以及调度

---

## 整体架构

![h:500](structure.svg)

---

## 项目规模

// TODO: 重新计算？

![h:500](zcore-loc.svg)

---

## 完成度：Zircon 官方系统调用测试集
![](tests.png)

---

## Benchmark

基于 Ubuntu 20.04，QEMU-KVM 1 CPU 测试

![h:250](benchmark-channel.png)
![h:250](benchmark-vmo.png)

---

## 软件工程：自动测试

* `#![deny(warnings)]`：警告报错

* cargo fmt && clippy：检查代码格式和风格
* cargo build：保证所有平台编译通过
* cargo test：用户态单元测试，报告测试覆盖率
* core-test：内核态集成测试，维护通过测例列表
* （TODO）cargo bench：性能测试

上述测试全部通过才允许合入 master

---

## 模块化：rCore OS 生态

拆成小型 no_std crate，每个专注一件事：
  * `trapframe-rs`：用户-内核态切换
  * `rcore-console`：在 Framebuffer 上显示终端
  * `naive-timer`：简单计时器
  * `executor`：单线程 Future executor
  * ……

完成功能后，继续完善文档、测试、examples

基本稳定后，发布到 crates.io，遵守语义化版本号更新

---

# HAL 硬件抽象层的设计实现

![h:250](zcore-hal.svg)

---

## 需求：内核对象单元测试

* 测试对象：线程 `Thread`，内存映射 `VMO`，`VMAR`

* 但 `cargo test` 只能在开发环境用户态运行

* 思考：能否在用户态**模拟**页表和内核线程？

---

## 方案：用户态模拟内核机制

* 内核线程：等价于用户线程 `std::thread`

* 内存映射：Unix `mmap` 系统调用

    * 用一个文件代表全部物理内存
    * 用 `mmap` 将文件的不同部分映射到用户地址空间

### 潜在问题

* 用户线程难以细粒度调度
* “页表”共享同一个地址空间

---

## 进一步思考：用户态 OS

既然每个内核对象都能在用户态完成其功能，

那么整个 OS 可不可以完全跑在用户态呢？

### 潜在好处

充分利用用户态丰富的开发工具，降低开发难度：
IDE + gdb + cargo + perf ...

### 现有解决方案

Library OS，User-mode Linux

---

## 最后的技术难点：用户-内核态切换 🧙‍♀️

用户程序和内核都运行在用户态……

* 控制流转移：系统调用 -> 函数调用
    * `int 80` / `syscall` -> `call`
    * `iret` / `sysret` -> `ret`
    * 需要修改用户程序代码段！

* 上下文恢复：寻找 "scratch" 寄存器
    * 用户程序如何找到内核入口点？内核栈？
    * 利用线程局部存储 TLS，线程指针 fsbase
    * macOS 无法设置 fsbase 怎么办？

---

## HAL API 举例

* 内核线程：`hal_thread_spawn`

* 物理内存：`hal_pmem_{read,write}`
* 虚拟内存：`hal_pt_{map,unmap}`
* 用户态：`hal_context_run`
* 定时器：`hal_timer_{set,tick}`
* 输入输出：`hal_serial_{read,write}`
* 设备管理：`hal_irq_{enable,handle}`

---

# async Rust 原理和设计模式

![h:200](are-we-async-yet.png)

👍 推荐阅读：[《使用 Rust 编写操作系统》#12 Async/Await](https://github.com/rustcc/writing-an-os-in-rust/blob/master/12-async-await.md)

---

## `async-await`：用同步风格编写异步代码

* 本质：无栈协程，协作式调度

* 适用于高并发 IO 场景

应用情况：

* 需要编译器的特殊支持：函数 => 状态机对象

* 主流编程语言均已支持：C#，JavaScript，Python，C++

* 几乎没有在 bare-metal 中应用

---

### Sync
```rust
fn handler(mut stream: TcpStream) -> Result<()> {
    let mut buf = [0; 1024];
    let len = stream.read(&mut buf)?; // may block
    stream.write_all(&buf[0..len]))?; // may block
}
```

### Async

```rust
// fn handler(...) -> impl Future<Output = Result<()>>
async fn handler(mut stream: TcpStream) -> Result<()> {
    let mut buf = [0; 1024];
    let len = stream.read(&mut buf).await?;
    stream.write_all(&buf[0..len])).await?;
}
```

---

![](async.png)

---


## 底层：手动构造 Future

```rust
// 例：在内核对象上等待信号
fn wait_signal(&self, signal: Signal) -> WaitSignal {
    // 定义一个状态机结构
    struct WaitSignal {...}
    // 实现 Future trait 的 poll 函数
    impl Future for WaitSignal {
        type Output = Signal;
        fn poll(self: Pin<&mut Self>, cx: &mut Context)
            -> Poll<Self::Output>
        {
            // 若目标事件已发生，直接返回 Ready
            if self.signal().contains(signal) {
                return Poll::Ready(signal);
            }
            // 尚未发生：注册回调函数，当事件发生时唤醒自己
            let waker = cx.waker().clone();
            self.add_signal_callback(move || waker.wake());
            Poll::Pending
        }
    }
    // 返回状态机对象
    WaitSignal {...}
}
```

---

## 中层：用 async-await 组合 Future

```rust
async fn sys_object_wait_signal(...) -> Result {
    ...
    let signal = kobject.wait_signal(signal).await;
    ...
}
```

高级用法：用 `select` 组合子实现 超时处理 和 异步取消

```rust
async fn sys_object_wait_signal(..., timeout) -> Result {
    ...
    let signal = select! {
        s = kobject.wait_signal(signal) => s,
        _ = delay_for(timeout) => return Err(Timeout),
        _ = cancel_token => return Err(Cancelled),
    };
    ...
}
```


---

## 上层：Executor 运行 Future

* libos：`tokio` / `async-std`，支持多线程，可以模拟多核
* bare：`rcore-os/executor`，简易单核
* 未来：期望嵌入式社区的 `async-nostd`？

### 进出用户态问题

async 要求保持内核上下文（即内核栈）

* 传统 OS：User call Kernel 风格，平时内核栈清空
* zCore：Kernel call User 风格，保留内核上下文

---

# Fuchsia 与 Zircon 内核对象

---

![](fuchsia-layer.jpeg)

---

## Zircon 内核特点

* 实用主义微内核

* 使用 C++ 实现，支持 x86_64 和 ARM64
* 面向对象：将功能划分到内核对象
* 默认隔离：使用 Capability 进行权限管理
* 安全考量：强制地址随机化，使用 vDSO 隔离系统调用

---

![h:600](user-env.jpg)

---

# Zircon 内核对象
## Everything can be KernelObject

* 任务：Job, Process, Thread, Exception

* 内存：VMAR, VMO, Pager, Stream
* IPC：Channel, FIFO, Socket
* 信号：Event, Timer, Port, Futex
* 驱动：Resource, Interrupt, PCI ...

---

### Object

* Object：内核对象
* Rights：对象访问权限
* Handle = Object + Rights：对象句柄（类似 fd）

![h:350](zircon-object.png)

---

## IPC

* Channel：进程间通信基础设施，可以传递数据和 handle
* FIFO：报文数据传输
* Socket：流数据传输

![](zircon-channel.png)

---

## Tasks

* Job：作业，负责控制权限（类似容器）
* Process：进程，负责管理资源
* Thread：线程，负责调度执行

![h:350](zircon-task.png)

---

## Memory and address space

* VMO: Virtual Memory Object
  * Paged：分页物理内存，支持写时复制
  * Physical：连续物理内存
    <!-- * 一段连续的虚拟内存，可以用于在进程之间、内核和用户空间之间共享内存
    * 在内核中被维护为类似线段树的数据结构，支持从一个VMO中创建新的VMO -->

* VMAR: Virtual Memory Address Region
    * 代表一个进程的虚拟地址空间
    * 树状结构
    <!-- * VMO可以被映射到VMAR中，默认为内核进行位置随机，或用户态指定位置但可能失败。 -->

* Pager：用户态分页机制

---

## Signaling and Waiting

每个 Object 有 32 个信号位，用户程序可以阻塞等待。

* Event (Pair)：事件源/对
* Timer：计时器
* Futex：用户态同步互斥机制
* Port：事件分发机制（类似 epoll）

![h:300](zircon-port.png)

---
# 异步 syscall 与内核协程调度
## io_uring 与异步 syscall
![img](io_uring.png)

---

## 异步 syscall 简单测试

![img](syscall.png)

---

## 异步 syscall 的感知

* 如何感知异步 syscall ?

  |              | CPU占用 | Latency | Locality |     备注     |
  | :----------: | :-----: | :-----: | :------: | :----------: |
  |   syscall    |    👍    |    👍    |    😰     |              |
  |   polling    |    😰    |    👍    |    👍     | 适合高速设备 |
  | free-polling |    👍    |    😰    |    👍     |              |
  |    uintr     |    👍    |    👍    |    👍     | 需要硬件支持 |

  zCore: free-polling + uintr

---

## 可抢占协程

* 异步执行 latency 的累加
  * 用户提交 => 内核感知, 内核提交 => 用户感知 ：uintr  or  polling
  * interrupt => 内核响应：优先级，可抢占协程
* 可退化的协程 = 可抢占的协程
  * 在中断 handler 里新建一个线程 
  * 协程的线程间迁移

---

## 协程的调度

* 协程的类型
  * 内核协程：处理异步 syscall 的协程，polling 协程等
  * 用户协程：用户线程封装而成
* 核专用化
  * 内核核：内核协程，切换次数多，切换开销小
  * 用户核：用户协程，切换考校大，切换次数少
  * 如果更进一步：磁盘 IO 核，网络核
* 问题：load balance


---
## 协程切换开销

* thread switch  =>  function return and call
* lmbench lat-ctx

```shell
Context switching - times in microseconds - smaller is better
-------------------------------------------------------------------------
Host                 OS  2p/0K 2p/16K 2p/64K 8p/16K 8p/64K 16p/16K 16p/64K
--------- ------------- ------ ------ ------ ------ ------ ------- -------
proc switch             0.3900 0.8800 0.8900 1.9200 2.2700 2.11000 2.25000
coroutine switch        0.0330 0.0390 0.0390 0.0600 0.0600 0.06800 0.06800
```

* 协程的内存开销 $\approx$ Max{ 同时存活的状态之和 }

---

# 谢谢

