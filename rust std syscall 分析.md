# rust std syscall 分析

### -1 发出 syscall 方式

* libc (abi)

  ```rust
  // sys::unix::fd::write
  pub fn write(&self, buf: &[u8]) -> io::Result<usize> {
      let ret = cvt(unsafe {
          libc::write(self.fd, buf.as_ptr() as *const c_void, cmp::min(buf.len(), READ_LIMIT))
      })?;
      Ok(ret as usize)
  }
  ```

* asm

  ```rust
  // sys::windows::abort_internal()
  asm!("int $$0x29", in("ecx") FAST_FAIL_FATAL_APP_EXIT);
  ```

  ```assembly
  .global get_tls_ptr
  get_tls_ptr:
      mov %gs:tcsls_tls_ptr,%rax
      pop %r11
      lfence
      jmp *%r11
  ```

* syscall!

  ```rust
  syscall! {
      fn copy_file_range(
          fd_in: libc::c_int,
          off_in: *mut libc::loff_t,
          fd_out: libc::c_int,
          off_out: *mut libc::loff_t,
          len: libc::size_t,
          flags: libc::c_uint
      ) -> libc::ssize_t
  }
  ```

  ```rust
  syscall( concat_idents!(SYS_, $name), $($arg_name),*) 
  ```

* extern "C"

  ```rust
  // sys::wasi::fs::open_parent
  extern "C" {
      pub fn __wasilibc_find_relpath(
          path: *const libc::c_char,
          abs_prefix: *mut *const libc::c_char,
          relative_path: *mut *const libc::c_char,
          relative_path_len: libc::size_t,
      ) -> libc::c_int;
  }
  ```

  

### 0. rust 官方 Library

* alloc
* unwind
* core
* panic_abort
* panic_unwind
* proc_macro
* profiler_builtins, 
* ... 

进行系统调用的：

* std
* backtrace
* panic_abort
* panic_unwind
* tests

### 1. std modules
```rust
mod macros;
pub mod prelude;
pub mod f32;
pub mod f64;
pub mod ascii;
pub mod backtrace;
pub mod collections;
pub mod error;
pub mod ffi;
pub mod num;
pub mod fs;
pub mod env;
pub mod io;
pub mod net;
pub mod panic;
pub mod path;
pub mod process;
pub mod sync;
pub mod time;
pub mod lazy;
pub mod task;
pub mod alloc;
mod memchr;
mod panicking;
pub mod rt;
// only abstraction
pub mod os;
mod sys_common
```

```rust
// std::thread
extern "system" fn thread_start(main: *mut c_void) -> c::DWORD {
    unsafe {
        let _handler = stack_overflow::Handler::new();
        Box::from_raw(main as *mut Box<dyn FnOnce()>)();
    }
    0
}
```

```rust
pub mod thread;
mod sys;
```

```rust
pub mod thread;

// thread/mod.rs
use crate::sys::thread as imp;
use crate::sys_common::mutex;
use crate::sys_common::thread;
use crate::sys_common::thread_info;
use crate::sys_common::thread_parker::Parker;
use crate::sys_common::{AsInner, IntoInner};

libc::{sysctl, sysconf} // 获取 cpu 数量

#[cfg(any(
        target_os = "android",
        target_os = "emscripten",
        target_os = "fuchsia",
        target_os = "ios",
        target_os = "linux",
        target_os = "macos",
        target_os = "solaris",
        target_os = "illumos",
    ))] {
        fn available_concurrency_internal() -> io::Result<NonZeroUsize> {
            match unsafe { libc::sysconf(libc::_SC_NPROCESSORS_ONLN) } {
                -1 => Err(io::Error::last_os_error()),
                0 => Err(io::Error::new(io::ErrorKind::NotFound, "...")),
                cpus => Ok(unsafe { NonZeroUsize::new_unchecked(cpus as usize) }),
            }
        }
    }
```


### 2. sys modules

* hermit: [hermit_abi](https://docs.rs/hermit-abi/0.1.8/hermit_abi/)
* sgx : [fortanix_sgx_abi](https://edp.fortanix.com/docs/api/fortanix_sgx_abi/index.html)
* unix: libc
* unsupport: ???
* vxworks: ???
* wasi / wasm: libc + syscall!
* windows: libc + asm


```rust
#[cfg(all(any(
    target_arch = "x86_64",
    target_arch = "aarch64",
    target_arch = "mips64",
    target_arch = "s390x",
    target_arch = "sparc64",
    target_arch = "riscv64"
)))]
pub const MIN_ALIGN: usize = 16;
```


### 3. unix: android / linux / macos

* malloc, free, realloc, memalign, aligned_alloc, posix_memalign, mmap
* ftruncate, pread, pwrite
* pthread_condattr_init, pthread_condattr_setlock, pthread_condattr_destory, pthrad_cond_xxxx
* clock_gettime / gettimeofday
* read / readv/ write / writev/ pread / pwrite / ioctl / fcntl / openat / close / fsync / fdatasync
* readdir / closedir / opendir / linkat / unlinkat / rmdir / rename / chmod
* dup, dup2
* pthread_mutex_[lock/init/unlock/destory/ .....]
* sysctl / getpid / getppid / setgid / setgroups
* yield_now / 
* pthread_set_name ..........
* strlen, memchr

### 使用方式

```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub fn copy<R: ?Sized, W: ?Sized>(reader: &mut R, writer: &mut W) -> Result<u64>
where
    R: Read,
    W: Write,
{
    cfg_if::cfg_if! {
        if #[cfg(any(target_os = "linux", target_os = "android"))] {
            crate::sys::kernel_copy::copy_spec(reader, writer)
        } else {
            generic_copy(reader, writer)
        }
    }
}
```
```rust
{
    syscall! {
        fn copy_file_range(
            fd_in: libc::c_int,
            off_in: *mut libc::loff_t,
            fd_out: libc::c_int,
            off_out: *mut libc::loff_t,
            len: libc::size_t,
            flags: libc::c_uint
        ) -> libc::ssize_t
    }
}
```
```rust
fn stack_buffer_copy<R: Read + ?Sized, W: Write + ?Sized>(
    reader: &mut R,
    writer: &mut W,
) -> Result<u64> {
    let mut buf = MaybeUninit::<[u8; DEFAULT_BUF_SIZE]>::uninit();
    // FIXME: #42788
    //
    //   - This creates a (mut) reference to a slice of
    //     _uninitialized_ integers, which is **undefined behavior**
    //
    //   - Only the standard library gets to soundly "ignore" this,
    //     based on its privileged knowledge of unstable rustc
    //     internals;
    unsafe {
        reader.initializer().initialize(buf.assume_init_mut());
    }

    let mut written = 0;
    loop {
        let len = match reader.read(unsafe { buf.assume_init_mut() }) {
            Ok(0) => return Ok(written),
            Ok(len) => len,
            Err(ref e) if e.kind() == ErrorKind::Interrupted => continue,
            Err(e) => return Err(e),
        };
        writer.write_all(unsafe { &buf.assume_init_ref()[..len] })?;
        written += len as u64;
    }
}
```

```rust
​```rust
// fs.rs
#[stable(feature = "rust1", since = "1.0.0")]
pub fn copy<P: AsRef<Path>, Q: AsRef<Path>>(from: P, to: Q) -> io::Result<u64> {
    fs_imp::copy(from.as_ref(), to.as_ref())
}
```

