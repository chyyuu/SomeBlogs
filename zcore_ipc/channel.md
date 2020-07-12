## Channel

channel 是最终要的 ipc 组件，可以传递字节数据和 handle。注意，虽然 handle 本身也是一个整数，但是仅仅传递这个整数是没有意义的，必须将对应的 kernel object 从原 process object 列表中删除并加入到目标 process 的列表中，这也是 handle 转移的本质。

理论上大小无限。

channel 还可以以同步的方式等待通信的完成。

### 接口

* 写

  在写操作上，channel 的一个特点是不会被部分写入，即：要么全读写入，要么一个都没有写入。

  ```rust
  pub fn sys_channel_write(
      &self,
      handle_value: HandleValue,
      options: u32,
      user_bytes: UserInPtr<u8>,
      num_bytes: u32,
      user_handles: UserInPtr<HandleValue>,
      num_handles: u32,
  )
  ```

  与通用模板相比，多了 handle 相关的指针，没有实际写入数量的反馈。

* 同步写

  channel 最有特色的机制在于它的同步机制。可以通过调用 channel_call() 进行一次写，同时阻塞直到对方作出回应，并得到返回的消息。channel 的同步机制通过 zCore async 机制进行实现。

  ```rust
  pub async fn sys_channel_call_noretry(
      &self,
      handle_value: HandleValue,
      options: u32,
      deadline: Deadline,	  // deadline of blocking wait
      user_args: UserInPtr<ChannelCallArgs>, // has both read array and write array
      mut actual_bytes: UserOutPtr<u32>,
      mut actual_handles: UserOutPtr<u32>,
  )
  ```

  ```rust
  pub struct ChannelCallArgs {
      wr_bytes: UserInPtr<u8>,
      wr_handles: UserInPtr<HandleValue>,
      rd_bytes: UserOutPtr<u8>,
      rd_handles: UserOutPtr<HandleValue>,
      wr_num_bytes: u32,
      wr_num_handles: u32,
      rd_num_bytes: u32,
      rd_num_handles: u32,
  }
  ```

  主要的同步代码为:

  ```rust
  fn sys_channel_call_noretry() {
  	// ...
      let future = channel.call(wr_msg);
      let rd_msg: MessagePacket = self
          .thread
          .blocking_run(future, ThreadState::BlockedChannel, deadline.into())
          .await?;
      // ...
  }
  ```

  ```rust
  pub async fn call(self: &Arc<Self>, mut msg: T) -> ZxResult<T> {
      // ...
      let (sender, receiver) = oneshot::channel();
      self.call_reply.lock().insert(txid, sender);
      receiver.await.unwrap()
  }
  
  pub fn write(&self, msg: T) -> ZxResult {
      // ...
      let txid = TxID::from_ne_bytes(msg.data[..4].try_into().unwrap());
      if let Some(sender) = peer.call_reply.lock().remove(&txid) {
          let _ = sender.send(Ok(msg));
          return Ok(());
      }
      // ...
  }
  ```

  在 call 中会阻塞直到 receiver 返回消息，而每次写入都会检测当前写操作是否对应一个对一个的 call 操作，如果是，则通过 peer.sender 马上返回 Ok，这样被阻塞进程就可以继续执行。

  

  