## 切片不是 FFI 安全的

### 前言

在开发 zCore kernel object stream 的时候，我碰到了一个很奇怪的 bug ，主要表现为空指针 is_null() 为 false，该 bug 在 debug 模式下不会出现，但是在 release 模式下会导致空指针检查失效，最终访问非法内存。

学长提醒我这可能是由于切片类型不是 FFI 安全的，我就这一问题进行了简单的测试。

### 前置经历

```rust
let s = String::from("hello world");
let world = &s[6..11];
```

<img src="https://kaisery.github.io/trpl-zh-cn/img/trpl04-06.svg" alt="图片替换文本" width="750" height="468" align="bottom" />

在[Rust 程序设计语言](https://kaisery.github.io/trpl-zh-cn/ch04-03-slices.html)中这张图示给了我暗示，我想当然的认为 slice 的存在如下的内存布局。

```rust
struct slice<T> {
    *const T,
    usize len,
}
```

### 测试

我们来测试一下上述猜想是否正确。

#### rust 内部测试

将 `(usize, usize)` 强制转化为 `&[u8]` 

```rust
fn main() {
    let test = (0usize, 1usize);
    let slice = make_slice(test);
    println!("{} {}", slice.as_ptr() as usize, slice.as_ptr().is_null());
}

fn make_slice(pair: (usize, usize)) -> &'static mut [u8] {
    unsafe { core::mem::transmute(pair) }
}
```

在 debug 和 release 模式下都能够正确运行。

#### rust 与 C FFI测试

测试代码

```rust
extern crate libc;

extern "C" {
    fn make_slice() -> &'static [u8];
    fn make_null_slice() -> &'static [u8];
}

fn main() {
let slice = unsafe { make_slice() };
let null = unsafe { make_null_slice() };
println!("{:x?}", slice);
println!(
    "{:x?} {} {}",
    null.as_ptr(),
    null.len(),
    null.as_ptr().is_null()
    );
}
```

```c
typedef unsigned long long usize;

// I assume that 'zx_iovec' has the same memory layout as '&[u8]'
typedef struct zx_iovec {
    void* buffer;
    usize capacity;
} zx_iovec_t;

zx_iovec_t null = {
    .buffer = (void*)0,
    .capacity = 3,
};

char buffer[17] = "0123456789ABCDEF";
zx_iovec_t vec = {
    .buffer = buffer,
    .capacity = sizeof(buffer),
};

zx_iovec_t make_null_slice() {
    return null;
}

zx_iovec_t make_slice() {
    return vec;
}
```

输出:

* 编译输出：

    ```c
    warning: `extern` block uses type `[u8]`, which is not FFI-safe
     --> src/main.rs:4:24
      |
    4 |     fn make_slice() -> &'static [u8];
      |                        ^^^^^^^^^^^^^ not FFI-safe
      |
      = note: `#[warn(improper_ctypes)]` on by default
      = help: consider using a raw pointer instead
      = note: slices have no C equivalent
    ```

    在 zCore 的编译中，类似的 warn 由于 UserPtr 的封装而未显示。

* debug 输出:

    ```c
    [30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 41, 42, 43, 44, 45, 46, 0]
    0x0000000000000000 3 true
    ```

    输出完全正确。

* release 输出:

    ```c
    [30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 41, 42, 43, 44, 45, 46, 0]
    0x0000000000000000 3 false
    ```

    只有最后的 is_null() 判断出错。

该结果完全复现了我在 zCore 中的遭遇：在大部分情况下能够正常使用，且 as_ptr() 获取的指针输出后确实为 0 ，但在 release 模式下却无法正确检测是否为空。另外，即便传递的是指针也无济于事。

### 分析

// TODO

这应该就是 warn 中 not FFI-safe 的具体体现之一。具体机制可能需要查阅 FFI 和 slice 相关文档才能解答，我还没来得及去做。

有兴趣的同学欢迎复现和解释背后机制。