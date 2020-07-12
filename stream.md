## Stream

stream 本身开发难度较低，本文可作为实现一个相对简单 syscall 的例子。

以下步骤是个人经验，仅供参考。

### Step1 阅读文档，明确开发目标

阅读 fuchsia/docs/reference/kernel_objects/stream.md, fuchsia/docs/reference/syscalls/stream_*，了解自己要开发的大致是怎样的一个 kernel object，有哪些 syscall。

### Step2 仿写 syscall 接口，走流程简单实现

按照 kernel/lib/syscalls/stream.cc 给出的接口，翻译为 zCore 接口。

```c
zx_status_t sys_stream_readv(
    zx_handle_t handle, 
    uint32_t options, 
    user_out_ptr<zx_iovec_t> vector,
    size_t vector_count, 
    user_out_ptr<size_t> out_actual)
```

创建 zircon-syscalls/src/stream.rs 文件

```rust
impl Syscall<'_> {
    pub fn sys_stream_readv(
        &self,
        handle_value: HandleValue,
        options: u32,
        vector: UserInPtr<IoVecOut>,
        vector_size: usize,
        mut actual_count_ptr: UserOutPtr<usize>,
    ) -> ZxResult;
}
```

类型一一对应即可，返回值统一为 ZxResult

接下来可以走流程完成 syscall 的基本工作：

```rust
pub fn sys_stream_readv(
    &self,
    handle_value: HandleValue,
    options: u32,
    vector: UserInPtr<IoVecOut>,
    vector_size: usize,
    mut actual_count_ptr: UserOutPtr<usize>,
) -> ZxResult {
    // info 输出信息，便于调试
    info!(
        "stream.read: stream={:#x?}, options={:#x?}, vector=({:#x?}; {:#x?})",
        handle_value, options, vector, vector_size,
    );
    // 检查输入参数是否错误
    if options != 0 {
        return Err(ZxError::INVALID_ARGS);
    }
    // 通过 handle 获得 kernel object
    let proc = self.thread.proc();
    let stream = proc.get_object_with_rights::<Stream>(handle_value, Rights::READ)?;
    // 读写 user_ptr,调用 kernel_object 函数
    let mut data = vector.read_iovecs(vector_size)?;
    // ...
    actual_count_ptr.write_if_not_null(actual_count)?;
    Ok(())
}
```

完成之后，可以在 zCore zircon-syscall/src/lib.rs 中增加新内容：

```rust
mod stream;
impl Syscall<'_> {
    pub async fn syscall(&mut self, num: u32, args: [usize; 8]) -> isize {
    // ...
    Sys::STREAM_CREATE => self.sys_stream_create(a0 as _, a1 as _, a2 as _, a3.into()),
    Sys::STREAM_WRITEV => {
    self.sys_stream_writev(a0 as _, a1 as _, a2.into(), a3 as _, a4.into())
    }
    Sys::STREAM_WRITEV_AT => {
    self.sys_stream_writev_at(a0 as _, a1 as _, a2 as _, a3.into(), a4 as _, a5.into())
    }
    // ...
    }
}
```

### Step 3 参考 C 代码，设计 kernel object 及其接口

参考 fuchsia kernel/object/stream_dispatcher.cc 及其 .h 文件，设计 kernel object 结构体：

创建 zircon-object/vm/src/stream.rs 文件

```rust
pub struct Stream {
    base: KObjectBase,
    options: u32,
    vmo: Arc<VmObject>,
    inner: Mutex<StreamInner>
}

pub struct StreamInner {
    seek: usize,
}
```

其中，可变部分放到 inner 中，这里由于仅有一个可变量，简化为：

```rust
pub struct Stream {
    base: KObjectBase,
    options: u32,
    vmo: Arc<VmObject>,
    seek: Mutex<usize>,
}
```

这里比较推荐大家参考 .h 文件，但是可以做比较大的简化，利用 rust 中方便的数据结构代替 zircon 中复杂的实现。

接下来设计函数接口，可以按照自己的习惯设计，主要为对应 syscall 的 pub 接口和内部辅助函数。

```rust
impl Stream {
    /// Create a stream from a VMO
    pub fn create(vmo: Arc<VmObject>, seek: usize, options: u32) -> Arc<Self> {}

    /// Read data from the stream at the current seek offset
    pub fn read(&self, data: &mut [u8]) -> ZxResult<usize> {}

    /// Read data from the stream at a given offset
    pub fn read_at(&self, data: &mut [u8], offset: usize) -> ZxResult<usize> {}

    //...
}
```

stream 并无内部函数，但是其他复杂的 kernel object 可能有不少，这是，最外层函数的工作往往是根据 options 调用不同的函数。

完成这一步后，到对应的 mod.rs 中添加内容。

```rust
mod stream;
pub use self::{stream::*};
```

### Step4 开发

参考 fuchsia kernel/object/stream_dispatcher.cc 及其它相关文件，进行开发。

```rust
pub fn sys_stream_readv(
    &self,
    handle_value: HandleValue,
    options: u32,
    vector: UserInPtr<IoVecOut>,
    vector_size: usize,
    mut actual_count_ptr: UserOutPtr<usize>,
) -> ZxResult {
    // ...
    let mut actual_count = 0usize;
    for io_vec in data.iter_mut() {
        // 调用设计好的 pub 接口
        actual_count += stream.read(io_vec.as_mut_slice()?)?;　
    }
    actual_count_ptr.write_if_not_null(actual_count)?;
    Ok(())
}

/// Read data from the stream at the current seek offset
pub fn read(&self, data: &mut [u8]) -> ZxResult<usize> {
    let mut seek = self.seek.lock();　　　　　 // 获取锁，进行数据操作
    let length = self.read_at(data, *seek)?; // 尽量复用代码，保持代码简洁
    *seek += length;
    Ok(length)
}

/// Read data from the stream at a given offset
pub fn read_at(&self, data: &mut [u8], offset: usize) -> ZxResult<usize> {
    let count = data.len();
    // 如果碰到其他部分没有实现的接口，需要想办法补全
    let content_size = self.vmo.content_size();
    // 如果 offset 指向无意义区域，直接返回。
    if offset >= content_size {
        return Ok(0);
    }
    let length = count.min(content_size - offset);
    // stream 更多的是一层封装，实际工作由 vmo 完成
    self.vmo.read(offset, &mut data[..length])?;
    Ok(length)
}
```

### Step5 对照测试消除 bug，优化代码

运行 runtests -n core-stream 

对照 fuchsia system/utest/core/stream/stream.cc 进行debug，善用 {:#x?} 等进行信息输出。

建议在 debug 和 release 两个模式下都进行测试。

debug 完成后，进行代码优化，尽量消除冗余代码，多利用 rust 语言特性。

例如利用 bitflags 进行 options 处理：

```rust
pub fn sys_stream_create() -> ZxResult {
    bitflags! {
        struct CreateOptions: u32 {
            #[allow(clippy::identity_op)]
            const MODE_READ     = 1 << 0;
            const MODE_WRITE    = 1 << 1;
        }
    }
    let options = CreateOptions::from_bits(options).ok_or(ZxError::INVALID_ARGS)?;
    let mut rights = Rights::DEFAULT_STREAM;
    let mut vmo_rights = Rights::empty();
    if options.contains(CreateOptions::MODE_READ) {
        rights |= Rights::READ;
        vmo_rights |= Rights::READ;
    }
    if options.contains(CreateOptions::MODE_WRITE) {
        rights |= Rights::WRITE;
        vmo_rights |= Rights::WRITE;
    }
    // ...
}
```

利用 numeric_enum 处理 enum

```rust
numeric_enum! {
    #[repr(usize)]
    #[derive(Debug)]
    pub enum SeekOrigin {
        Start = 0,
        Current = 1,
        End = 2,
    }
}

let whence = SeekOrigin::try_from(whence).map_err(|_| ZxError::INVALID_ARGS)?;
```

## 开发日志

在实际开发过程中，stream 的开发并非一帆风顺，主要困难在于

* vmo 功能不全
* release mode 空指针检查失效的问题(slice is not FFI-safe)

因此还需要对 vmo 进行功能补全并设计 iovec 模块（由 wrj 完成实际开发）。

vmo 部分可以参考 vmo_dispatcher.cc 等文件，需要阅读 vmo 相关 zCore 代码。

空指针检查失效问题比较隐蔽，好在测试比较全面，借助读写非法 buffer的测试可以比较精确定位 bug，使用自定义类型（iovec）限制内存布局后解决了这个问题。