# 增加 async feature

目标：在支持协程的框架下，能够运行宏内核

主要的困难：如何在 async 的环境下，执行同步的代码，并且能够保证任务切换和任务调度能够正常工作

可能的实现方式：

1. 将使用的接口都改成 async 的形式，但这种情况的工作量无法估计
2. 在 async 的环境下，保留已有的接口不变，但在需要进行任务切换时，按照线程、协程共存的方式进行切换

## 方案 1 可行性分析

由于 async 接口具有传递性（只能在 async 的环境下使用），因此需要分析清楚其他模块中使用到的 axprocess 向外暴露的接口

根据 Cargo.lock 以及手工分析得出一下关系：

1. axfeat：只是 axprocess 模块中向外暴露的接口的中转站
2. axnet：使用了 axprocess::signal::current_have_signals 接口，无需修改
3. axruntime：使用了 axprocess::init_kernel_process 接口，无需修改
4. linux_syscall_api：使用了以下接口
   - [x] wait_pid, yield_now_task, exit_current_task, sleep_now_task
   - [ ] Process, PID2PC, TID2TASK
   - [ ] current_process, current_task, get_task_ref
   - [ ] time_stat_from_kernel_to_user, time_stat_from_user_to_kernel, time_stat_output
   - [ ] handle_page_fault
   - [ ] link::{create_link, FilePath, get_link_count, AT_FDCWD, real_path, remove_link, deal_with_path, raw_ptr_to_ref_str}
   - [ ] futex::{clear_wait, get_futex_key, FutexKey, FutexRobustList, FUTEX_WAIT_TASK, WAIT_FOR_FUTEX}
   - [ ] handle_signals, signal_return, send_signal_to_process, send_signal_to_thread
   - [ ] flags::{CloneFlags, WaitStatus}
   - [ ] set_child_tid, SignalModule,

这个结果可能分析不准确，同理根据 Cargo.lock 文件分析，依赖 axtask 的模块有：

1. arceos_api
2. arceos_posix_api
3. axfeat
4. axnet
5. axprocess
6. axruntime
7. axsync
8. axtrap
9. linux_syscall_api

而即使是将这些依赖关系都解决后，在 axstd 模块中存在着大量的接口都依赖着同步的接口（例如 axstd 中的 io/stdio.rs 和 sync/mutex.rs 文件中依赖着同步的接口），因此工作量非常巨大，只能集众人之力来实现这个目标，或者花更多的时间。

## 方案 2 的可行性

### 试图在同步的接口中直接使用异步函数

future 驱动的方式除了使用 await 之外，还可以在同步的接口中使用 future 的 poll 方法，以下列代码为例。yield_now 是一个 async 形式的函数，在同步的代码中，这是一个 future，无法执行。

```rust {.line-numbers}
impl Read for Stdin {
    // Block until at least one byte is read.
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let read_len = self.inner.lock().read(buf)?;
        if buf.is_empty() || read_len > 0 {
            return Ok(read_len);
        }
        // try again until we got something
        loop {
            let read_len = self.inner.lock().read(buf)?;
            if read_len > 0 {
                return Ok(read_len);
            }
            crate::thread::yield_now();
        }
    }
}
```

按照如下修改，在第 17 行调用 poll 函数会执行 future，进而执行 `yield_now`，在 `yield_now` 返回 `pending` 后，执行流会回到第 18 行，从而返回到上层调用 read 的执行流中。

但这样不能让出 CPU，是没有意义的。

```rust {.line-numbers}
impl Read for Stdin {
    // Block until at least one byte is read.
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let read_len = self.inner.lock().read(buf)?;
        if buf.is_empty() || read_len > 0 {
            return Ok(read_len);
        }
        // try again until we got something
        loop {
            let read_len = self.inner.lock().read(buf)?;
            if read_len > 0 {
                return Ok(read_len);
            }
            let mut future = Box::pin(crate::thread::yield_now());
            let waker = core::task::Waker::noop();
            let mut cx = core::task::Context::from_waker(&waker);
            future.as_mut().poll(&mut cx);
        }
    }
}
```

经过一番探索之后，似乎没有原生的贴近 rust 协程的方法。

因此，在考虑使用 async feature 下，yield 等与调度相关的接口需要按照原本的线程的实现，即使是主动让权，但也需要占用栈，因为没有执行到 await 的点（同步的接口中没有 await），上下文没有保存在堆中。为此，增加了 yield_switch_entry 接口，只保存 callee 寄存器，并切换到新的栈上执行其余的任务。

## U trap

由于第一次从 S 态进入到 U 态是预先在栈上存放 trapframe 上下文，但由于对 TaskInner 数据结构进行了修改，没有了 kstack（内核栈），需要找一个地方来存放 trapframe。首先尝试了在用户栈上保存 trapframe 上下文（需要将用户栈的段的权限修改为在 S 态和 U 态均可访问），可以跳转到 U 态执行用户代码，但执行系统调用回到 S 态时，由于一些寄存器（gp、tp）的保存和恢复与 trap 没有对应，导致了无法正常处理系统调用。==继续在这条路上前进似乎需要的修改很大，但在理想的情况下，可能用户进程执行只需要一个用户态的栈即可。==

为了尽可能的减少工作量，目前的方案是保留了 kstack，只在同时使能了 monolithic、async feature 时使用，将 trapframe 的信息仍然保留在 kstack 上，而任务的上下文切换则仍然使用线程、协程共存的方式。

## 上下文类型

由于增加了 yield 主动让权的接口，因此 task_context 的类型增加了，需要根据不同的类型进行保存和恢复操作：

- 0：S 态 trap 上下文，需要保存/恢复除了 gp、tp 之外的所有通用寄存器以及 sepc、sstatus
- 1：U 态 trap 上下文，保存/恢复所有通用寄存器以及 sepc、sstatus
- 2：S 态线程上下文，保存/恢复 ra、sp、s0 ~ s11 这些寄存器

三种类型上下文的恢复都在 taskctx 模块中的 load_next_ctx 函数中实现。







