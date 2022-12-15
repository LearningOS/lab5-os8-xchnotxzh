# 功能总结

- 实现功能 -- 基于“改良的银行家算法”的mutex 和 semaphore的死锁检测
- 实现策略
  - 死锁判定状态 -- 在TCB中加入锁和信号量的分配矩阵和需求矩阵
  - 死锁判定状态更新 -- 在创建锁和开关锁、创建信号量和执行信号量P/V操作时 做相应状态变化。
  - 死锁判定 -- 在上锁、信号量P操作时进行死锁判定



# 问答题

## 1

### 问题

在我们的多线程实现中，当主线程 (即 0 号线程) 退出时，视为整个进程退出， 此时需要结束该进程管理的所有线程并回收其资源。 - 需要回收的资源有哪些？ - 其他线程的 TaskControlBlock 可能在哪些位置被引用，分别是否需要回收，为什么？

### 解答

整个进程退出需回收

- 所有线程的私有资源 -- 内核栈、用户栈、TRAMPOLINE
- 私有线程的公共资源 -- 进程内的地址空间和物理页框、PID等。

其他线程的TCB可能

- 在进程的tasks中被引用，是weak引用，不需要回收。
- 在TaskManager队列、同步机制的等待队列中被引用，需要同步清理，因为进程的公共资源已经被回收，如果这些线程被重新调度，可能会产生非法访问。

## 2

### 问题

对比以下两种 Mutex.unlock 的实现，二者有什么区别？这些区别可能会导致什么问题？

```rust
impl Mutex for Mutex1 {
    fn unlock(&self) {
        let mut mutex_inner = self.inner.exclusive_access();
        assert!(mutex_inner.locked);
        mutex_inner.locked = false;
        if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
            add_task(waking_task);
        }
    }
}

impl Mutex for Mutex2 {
    fn unlock(&self) {
        let mut mutex_inner = self.inner.exclusive_access();
        assert!(mutex_inner.locked);
        if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
            add_task(waking_task);
        } else {
            mutex_inner.locked = false;
        }
    }
}
```

### 解答

Mutex1 先解除互斥锁再唤醒等待线程，可导致互斥锁解除后被其他线程占用从而互斥锁失效。

Mutex2 先唤醒等候线程再解除互斥锁，可导致等候线程被唤醒后互斥锁仍未解除。



# 建议

无