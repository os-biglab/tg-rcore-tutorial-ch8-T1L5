# ch8-T1L5 Exercise 实现报告（enable_deadlock_detect）

本文对应 `exercise.md`，说明当前 `tg-rcore-tutorial-ch8-T1L5` 中死锁检测功能的实现思路与代码落点。

## 1. 练习目标

chapter8 练习要求新增：

- `enable_deadlock_detect(is_enable)`（ID 469）

并在开启后使以下资源获取在检测到潜在死锁时拒绝：

- `mutex_lock`
- `semaphore_down`

拒绝时返回 `-0xDEAD`。

---

## 2. 架构前提（为何在 ch8 可做）

ch8 已具备：

- 线程级调度模型（能区分等待线程与持有线程）；
- 进程级共享同步对象（mutex/semaphore 列表）；
- 进程内资源关系记录结构（owner/waiters/allocation/need）。

因此可以在 syscall 层做“请求前检测”，并由调度层执行阻塞或拒绝策略。

---

## 3. 代码落点

主要位于 `src/main.rs` 的 `impls` 模块：

- 死锁开关状态：`Process.deadlock_detect_enabled`；
- 互斥锁检测：
  - `has_path_mutex(...)`
  - `would_deadlock_mutex(...)`
- 信号量检测：
  - `would_deadlock_sem(...)`
  - 使用 `Available/Allocation/Need` 风格安全性判定；
- 错误码常量：`DEADLOCK_ERR = -0xdead`。

辅助状态字段在 `src/process.rs`：

- `mutex_owner`
- `mutex_waiters`
- `sem_allocation`
- `sem_need`
- `sem_total`

---

## 4. enable_deadlock_detect 语义

系统调用签名：

```rust
fn enable_deadlock_detect(&self, _caller: Caller, is_enable: i32) -> isize
```

行为：

- `is_enable == 1`：开启死锁检测；
- `is_enable == 0`：关闭死锁检测；
- 其他参数返回 `-1`；
- 成功返回 `0`。

该开关是“进程级策略”，影响当前进程内线程的同步资源请求行为。

---

## 5. mutex 死锁检测

## 5.1 思路

将“线程等待某 mutex”视为从 waiter 指向 owner 的依赖边。若新边引入后形成环，则可能死锁。

## 5.2 实现

- `would_deadlock_mutex(waiter, owner)`：
  - 若 `waiter == owner` 直接判定死锁；
  - 否则 DFS/BFS 追踪 owner 的等待链，检查是否可回到 waiter。

## 5.3 结果策略

- 开关关闭：沿用原阻塞行为；
- 开关开启且检测命中：`mutex_lock` 直接返回 `-0xDEAD`，不进入阻塞。

---

## 6. semaphore 死锁检测

## 6.1 思路

参考题目给出的安全性检测算法：

- 构造 `Available`；
- 从记录中提取 `Allocation` 与 `Need`；
- 模拟完成序列（`Work/Finish`）判断系统是否安全。

## 6.2 实现

`would_deadlock_sem(waiter, sem_id)` 关键步骤：

1. 收集参与线程集合；
2. 组装二维 `allocation`/`need`；
3. 注入当前 `down` 请求（增加对应 `need`）；
4. 运行安全性循环：可满足则释放资源推进 `work`；
5. 若无法让全部线程 `finish=true`，判定不安全。

## 6.3 结果策略

- 检测命中返回 `-0xDEAD`；
- 未命中则按原逻辑尝试获取或阻塞。

---

## 7. 调度层协同

在主调度循环中：

- `SEMAPHORE_DOWN` / `MUTEX_LOCK` / `CONDVAR_WAIT` 返回 `-1` 时，线程进入 blocked；
- 若返回 `-0xDEAD`，表示请求被拒绝而非等待，线程可继续由用户态处理错误路径。

这样可区分“资源暂不可用”和“检测到死锁风险”两类失败语义。

---

## 8. 验证方式

在 `tg-rcore-tutorial-ch8-T1L5` 目录运行：

```bash
cargo run --features exercise
```

随后输入：

```text
tg-rcore-tutorial-ch8_usertest
```

或直接：

```bash
./test.sh exercise
```

重点验证：

- 开关参数合法性与返回值；
- 开关关闭时行为与原同步逻辑一致；
- 开关开启时潜在死锁请求返回 `-0xDEAD`；
- 非死锁场景不被误拒绝。

---

## 9. 已知边界

当前实现按实验要求简化：

- mutex 与 semaphore 分开检测，不处理混合资源图；
- 面向教程可读性，优先语义正确；
- 更复杂性能优化与跨进程死锁分析未纳入。

但已满足 chapter8 对“可切换死锁检测机制”的核心要求。