# ch8-T1L5 软件架构总览

本文描述 `tg-rcore-tutorial-ch8-T1L5` 作为独立 crate 的实现结构、执行路径与模块分工。

## 1. 系统定位

`ch8-T1L5` 在 ch7 基础上引入“线程 + 同步原语 + 阻塞/唤醒”并发模型。核心转变是把“执行实体”从进程拆分为线程。

本 crate 的关键能力：

- 进程与线程解耦：
  - `Process` 管共享资源（地址空间、fd 表、信号、同步对象）；
  - `Thread` 管执行上下文（TID + 寄存器状态）。
- 双层调度管理：`PThreadManager` 同时维护进程与线程集合；
- 同步原语 syscall：`mutex/semaphore/condvar`；
- 线程阻塞态管理：资源不可用时将线程从就绪队列移除，等待唤醒；
- 可选死锁检测开关（exercise 关注点）。

与 ch3/ch4/ch5/ch6 的差异：前几章主要是“任务/进程/虚存/文件系统”，ch8 的中心是“并发执行语义与同步正确性”。

---

## 2. 目录与模块职责

```text
tg-rcore-tutorial-ch8-T1L5/
├── .cargo/config.toml         # 目标平台与 QEMU 运行参数
├── build.rs                   # 构建脚本（含用户程序和 fs 镜像准备）
├── Cargo.toml                 # crate 元信息、feature、依赖
├── README.md                  # 章节文档
├── exercise.md                # 练习要求（死锁检测开关）
└── src/
    ├── main.rs                # 启动、调度主循环、trap/syscall 分发
    ├── process.rs             # Process/Thread 定义与 from_elf/fork/exec
    ├── processor.rs           # PThreadManager 封装与线程/进程管理器
    ├── fs.rs                  # 文件系统封装与统一 Fd 枚举
    └── virtio_block.rs        # VirtIO 块设备驱动
```

---

## 3. 分层架构

```text
用户态线程（ecall / 同步原语）
      │
      ▼
syscall 语义层（main.rs::impls）
      │
      ▼
并发管理层（processor.rs + process.rs）
      │
      ├── Process：共享资源容器
      └── Thread：执行上下文
      ▼
资源层（fs + signal + sync + vm）
      │
      ▼
RISC-V Sv39 + VirtIO + QEMU
```

### 3.1 `main.rs`：并发调度总控

`rust_main` 负责：

1. 初始化内核空间与传送门；
2. 初始化 syscall 子系统（含 `thread` 与 `sync_mutex`）；
3. 从 FS 读取 `initproc`，创建 `(Process, Thread)`；
4. 启动主循环，按线程粒度调度执行；
5. 在 trap 返回后处理：
   - 常规 syscall；
   - 信号处理；
   - 同步原语失败时的阻塞态迁移。

### 3.2 `process.rs`：资源与执行分离

- `Thread`：`tid + ForeignContext`；
- `Process`：
  - 地址空间、fd 表、信号；
  - `semaphore_list/mutex_list/condvar_list`；
  - 死锁检测状态与资源图数据结构。

关键方法：

- `from_elf` 返回 `(Process, Thread)` 而不是单对象；
- `fork` 复制资源容器并生成子线程；
- `exec` 替换进程映像并更新主线程上下文。

### 3.3 `processor.rs`：双层管理器

- `ProcessorInner = PThreadManager<...>`；
- `ThreadManager` 维护线程实体与就绪队列；
- `ProcManager` 维护进程实体；
- 调度粒度从进程切换为线程。

### 3.4 `fs.rs` / `virtio_block.rs`：共享资源层

- `fs.rs` 提供统一 `Fd` 枚举（文件/管道/空描述符）；
- 同一进程多线程共享 `fd_table`；
- `virtio_block.rs` 作为稳定块设备后端，为 easy-fs 提供读写。

---

## 4. 运行时关键路径

### 4.1 启动路径

1. 建立内核映射（含 MMIO）；
2. 初始化 syscall；
3. 载入 `initproc` 并创建进程 + 主线程；
4. 注入双层管理器并进入调度。

### 4.2 syscall 与状态迁移

- 普通 syscall：返回后线程挂起（轮转）；
- `EXIT`：当前线程/进程按管理器语义退出；
- `MUTEX_LOCK` / `SEMAPHORE_DOWN` / `CONDVAR_WAIT`：
  - 成功 -> 挂起；
  - 失败（资源不可用）-> 标记阻塞，等待唤醒。

### 4.3 死锁检测切入点

- 通过 `enable_deadlock_detect` 控制进程级检测开关；
- 对 mutex 和 semaphore 分别检测；
- 检测到死锁风险时拒绝请求并返回特定错误码。

---

## 5. 设计要点（ch8 特有）

- 资源归属进程，执行归属线程；
- 同步原语对象在进程内共享，避免跨进程污染；
- 阻塞与唤醒由调度层统一处理，syscall 只表达语义结果；
- 死锁检测使用资源分配/需求关系图，保持实现可教学追踪。

---

## 6. 配置与依赖

关键依赖：

- `tg-task-manage`（`thread` feature）：线程/进程双层管理；
- `tg-sync`：mutex/semaphore/condvar 原语；
- `tg-signal` + `tg-signal-impl`：信号机制；
- `tg-easy-fs` + `virtio-drivers`：文件系统与块设备；
- `tg-kernel-vm`：地址空间与页表。

---

## 7. 当前实现边界

为保持教学最小闭环，当前实现仍简化：

- 调度与唤醒策略偏可读性，不追求高吞吐；
- 死锁检测按实验要求分开处理 mutex 与 semaphore；
- 高级并发诊断与性能分析能力未展开。

但已完整覆盖“线程并发 + 同步互斥 + 阻塞唤醒 + 死锁检测开关”的核心路径。