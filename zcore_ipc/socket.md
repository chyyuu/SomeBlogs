## Socket

花里胡哨的 ipc。

只能进行字节数据传输，但是支持"数据报"和"数据流"两种传递方式。但是一个 socket 只能选择一个模式，在创建时确定。

* 数据报: 每次读取一整个报文，buffer 空间不足则报文截断，多余部分舍弃。
* 数据流: 每次尽量读取 buffer 长度 byte 的数据。
* 控制数据: 特殊的数据报，但容量为1，必须读取后才能重新写入。

支持 peek 读取：读取内容与正常相同，但不对数据造成影响，即：数据进行复制而不是剪切。

```rust
if options.contains(SocketFlags::SOCKET_PEEK) {
    for (i, x) in inner.control_msg.iter().take(read_size).enumerate() {
        data[i] = *x;
    }
} else {
    for (i, x) in inner.control_msg.drain(..read_size).enumerate() {
        data[i] = x;
    }
    self.base.signal_clear(Signal::SOCKET_CONTROL_READABLE);
    if let Some(peer) = self.peer.upgrade() {
        peer.base.signal_set(Signal::SOCKET_CONTROL_WRITABLE);
    }
}
```

存在一个较大的最大长度（128 * 2048）。

存在 read_threshold, write_threshold 字段，处理特殊 signal: 当 socket 中可读/可写的内容超过 read_threshold/write_threshold 时，SOCKET_READ_THRESHOLD / SOCKET_WRITE_THRESHOLD 信号被激发。

### 数据报的实现：

在 zCore 中，使用一个长数组和一个长度数组模拟数据报。

```rust
data: VecDeque<u8>,
datagram_len: VecDeque<usize>,
```

在 zircon 代码中，使用一个数组的链表实现数据报。

### 关于测试

socket test 中有两个比较特殊，`ReadIntoBadBuffer` 、`WriteFromBadBuffer`

```c
TEST(SocketTest, ReadIntoBadBuffer) {
  zx::socket a, b;
  ASSERT_OK(zx::socket::create(0, &a, &b));
  
  constexpr size_t kSize = 4096;
  zx::vmo vmo;
  ASSERT_OK(zx::vmo::create(kSize, 0, &vmo));

  zx_vaddr_t addr;

  // Note, no options means the buffer is not writable.
  ASSERT_OK(zx::vmar::root_self()->map(0, vmo, 0, kSize, 0, &addr));

  size_t actual = 99;
  void* buffer = reinterpret_cast<void*>(addr);
  ASSERT_NE(nullptr, buffer);

  // Will fail because buffer points at memory that isn't writable.
  EXPECT_EQ(ZX_ERR_INVALID_ARGS, b.read(0, buffer, 1, &actual));
}
```

这两个测试中会创建一个”不可读不可写不可执行“的页面映射，这在目前 zCore 的实现中会导致 vmar map 失败（因为这样的页面是没有任何意义的，仅仅会存在于测试中），这样的判断是合理的，临时变通修改代码后可以通过该测试，目前的实现中保留了这个合理的"错误"。

