## IPC 综述

ipc 模块的主要功能是按要求完成不同格式和要求的进程间数据传输，包含 3 个 kernel object : channel, fifo, socket。

对于具体 kernel object 的功能和接口可以参考 zircon 文档或者相关代码，这里主要记录在 zCore 实现中的一些宏观的细节。

### 主要接口格式和实现方式

 ipc 模块的系统调用有三个最主要接口：创建、读、写，在 kernel object 接口上也一样。每个接口都可以通过传入 optioins 参数实现不同不能的读写。例如 socket 的 write 函数，在不同的 options 参数(其实还要根据创建时的 options 参数区分)下通过调用 write_control, write_data, write_datagram 完成三个不同功能的读写。

一下以 socket 为例说明接口基本情况。

* 创建

  和其他的 kernel object 不同，ipc 模块的 kernel object 都是成对存在的，因此创建时都会创建两个，互为 peer，并返回一对 handle。进程需要通过 channel 将一个 handle 传递给通信对象，最初的 channel 在新进程创建时传递。

  signal 是所有的 ipc kernel object 的重要处理对象，所以在创建时必须依据参数进行正确的 signal 初始化。

  创建的接口大都形如：

  ```rust
  pub fn sys_socket_create(
      &self,
      options: u32,
      mut out0: UserOutPtr<HandleValue>,
      mut out1: UserOutPtr<HandleValue>,
  )
  ```

* 写

  写入接口大都形如:

  ```rust
  pub fn sys_socket_write(
      &self,
      handle_value: HandleValue,				// handle value of target object
      options: u32,							// options to control read
      user_bytes: UserInPtr<u8>,				// user buffer pointer
      count: usize,							// the size of user buffer
      mut actual_count_ptr: UserOutPtr<usize>,// the number of bytes actually write
  )
  ```

  其中，由于各种原因，写可能被截断，所以实际写入的数据长度可能小于 buffer size，这一数据通过最后一个参数传递。

  读的总体实现方式如下:

  * 进行简单参数检查，将 user_bytes 处理为 Vec\<T\>, 将 handle_value 处理为实际 Arc\<Socket\>。这一步通过调用 kernel_hal::user::UserPtr 和 Process 的相关函数实现。
  * 调用 socket.write() 完成写入。
    * 根据 options 分发到不同的内部函数。
    * 完成数据的实际写入。
    * 处理 signal。如：是否仍旧可读，peer 是否可写等。
  * 写入实际读入数量，返回正确。

  写操作本质上是写入 peer 的数据区域，所以读操作要求 peer 必须存在，否则操作会直接失败。

* 读

  读与写互为镜像操作，在接口和实现方式上基本一致。

  ```rust
  pub fn sys_socket_read(
      &self,
      handle_value: HandleValue, 			 
      options: u32,						 
      mut user_bytes: UserOutPtr<u8>,      
      size: usize,						 
      mut actual_size_ptr: UserOutPtr<usize>, 
  )
  ```

  与写的不同之处为：user_bytes : UserOutPtr 可写但不可读，与写操作中的 UserInPtr 相反。

  此外，读不要求 peer 一定存在，peer 可以已经关闭。读操作除了 signal 的设置和清楚不需要 peer 参与。

### ipc 中的内存拷贝

每一次 ipc 涉及三个内存区域：sender process 的用户空间（Ｓ空间），内核空间（Ｋ空间），receiver process 的用户空间（R 空间）。

读的本质是将数据从Ｋ空间转移到 Ｒ空间，写则是从 S 空间转移到 K 空间。理论上讲，每次读或者写都只需要一个内存拷贝，一次 ipc 需要两次内存拷贝（异步通信最少两次）。

但在 zCore 实际实现中，受制于 UserPtr 接口形式，每次读写实际上都发生了两次内存拷贝。以写为例，有如下主要步骤：

```rust
data = user_bytes.read_array(size)?; // the first memory copy
socket.data.extend(data);            // the second memory copy
```

也就是从 S 到 kernel object 实际存储区域（位于K）的过程中，会有一个同样属于 K 的中转的临时存储。 read_array (即将数据从 user_bytes中取出的过程) 会自动产生这个中转存储。

在 stream 的实现中，我感觉通过 slice 可以消除这多出来的一次内存拷贝，且不会带来很大的内存安全问题。


