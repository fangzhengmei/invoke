# Invoke 并发与异步执行模式分析报告

## 目录
1. [并发模式与任务协调调度](#1-并发模式与任务协调调度)
2. [异步运行模式的 IO 流管理](#2-异步运行模式的-io-流管理)
3. [pty 初始化阶段的预留路径分析](#3-pty-初始化阶段的预留路径分析)

---

## 1. 并发模式与任务协调调度

### 1.1 Invoke 中的并发模型

**重要说明：Invoke 核心库本身**没有内置的 "Group" 并发类**，但提供了构建并发执行的基础能力。用户提到的 "Group 并发模式" 可能来自：
- **Fabric 2.x**（Invoke 的下游项目）中的 `Group` 类
- 基于 `threading.Thread` 或 `concurrent.futures` 的自定义并发模式

### 1.2 Executor 的串行执行模型

Invoke 核心的 `Executor.execute()` 方法是**串行执行**的：

```python
# executor.py:120-149
def execute(self, *tasks):
    # ...
    for call in calls:
        # 串行执行每个任务
        result = call.task(*args, **call.kwargs)
        results[call.task] = result
    return results
```

**执行流程：
1. 任务规范化（normalize）
2. 展开 pre/post 任务（expand_calls）
3. 去重（dedupe）
4. **串行 for 循环执行**

### 1.3 基于 ExceptionHandlingThread 的并发能力

Invoke 提供了 `ExceptionHandlingThread` 作为并发执行的基础：

```python
# util.py:146-257
class ExceptionHandlingThread(threading.Thread):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.daemon = True  # 守护线程
        self.exc_info = None  # 存储异常信息
    
    def run(self):
        try:
            if hasattr(self, "_run") and callable(self._run):
                self._run()
            else:
                super().run()
        except BaseException:
            self.exc_info = sys.exc_info()  # 捕获异常
    
    def exception(self):
        if self.exc_info is None:
            return None
        return ExceptionWrapper(self.kwargs, *self.exc_info)
    
    @property
    def is_dead(self):
        # 线程已终止且有异常
        return (not self.is_alive()) and self.exc_info is not None
```

### 1.4 StreamWatcher 的线程安全机制

**核心设计：继承 `threading.local`**

```python
# watchers.py:8-36
class StreamWatcher(threading.local):
    """
    继承自 threading.local，确保线程安全。
    
    这意味着：
    - stdout 处理线程和 stderr 处理线程有各自独立的实例属性
    - 每个线程的状态（如 index、tried 等）互不干扰
    """
    
    def submit(self, stream: str) -> Iterable[str]:
        raise NotImplementedError
```

**为什么需要 threading.local 很重要？**

在 Runner 中，stdout 和 stderr 由**两个独立线程**处理：

```
主线程
    ├── handle_stdout 线程 ──→ 调用 watcher.submit(stream)
    │                                   ↓
    │                              watcher.index (线程1 独立)
    │
    ├── handle_stderr 线程 ──→ 调用 watcher.submit(stream)
    │                                   ↓
    │                              watcher.index (线程2 独立)
    │
    └── handle_stdin 线程（可选）
```

**Responder 中的线程隔离验证：**

```python
# watchers.py:62-110
class Responder(StreamWatcher):
    def __init__(self, pattern: str, response: str) -> None:
        self.pattern = pattern
        self.response = response
        self.index = 0  # 每个线程独立的 index
    
    def pattern_matches(self, stream: str, pattern: str, index_attr: str):
        index = getattr(self, index_attr)  # 获取当前线程的 index
        new = stream[index:]
        # ...
        if matches:
            setattr(self, index_attr, index + len(new))  # 更新当前线程的 index
```

**线程安全保证：**

| 场景 | 行为 | 安全性 |
|------|------|--------|
| 单线程使用 | 正常工作 | ✅ 安全 |
| 多线程共享同一 Watcher 实例 | 每个线程有独立的属性副本 | ✅ 安全 |
| FailingResponder 的双索引 | index 和 failure_index 都是线程局部的 | ✅ 安全 |

**潜在的问题：**

如果需要在**多进程（multiprocessing）环境中使用，`threading.local` 不会提供保护，需要额外的同步机制。

### 1.5 自定义并发模式的实现方式

**基于 threading 的并发示例：

```python
from invoke import Context
from invoke.util import ExceptionHandlingThread
from queue import Queue

def run_task(context, command, result_queue):
    result = context.run(command)
    result_queue.put(result)

# 创建并发起
c = Context()
queue = Queue()

threads = [
    ExceptionHandlingThread(target=run_task, args=(c, "echo task1", queue)),
    ExceptionHandlingThread(target=run_task, args=(c, "echo task2", queue)),
]

for t in threads:
    t.start()

for t in threads:
    t.join()
    # 检查异常
    exc = t.exception()
    if exc:
        print(f"Thread error: {exc.value}")

# 收集结果
results = []
while not queue.empty():
    results.append(queue.get())
```

---

## 2. 异步运行模式的 IO 流管理

### 2.1 异步模式的核心标志

```python
# runners.py:122-124
self._asynchronous = False
self._disowned = False
```

### 2.2 标准输入的自动禁用路径

**关键代码路径：**

```python
# runners.py:585-590
in_stream = opts["in_stream"]
if in_stream is None:
    # 异步模式下自动设置为 False，禁用 stdin 读取
    in_stream = False if self._asynchronous else sys.stdin
```

**IO 线程创建条件：**

```python
# runners.py:666-677
if self.streams["in"]:  # 只有 in_stream 为真值时才创建 stdin 线程
    thread_args[self.handle_stdin] = {...}
```

**测试验证：**

```python
# tests/runners.py:1473-1483
def does_not_forward_stdin(self):
    class MockedHandleStdin(_Dummy):
        pass
    MockedHandleStdin.handle_stdin = Mock()
    
    runner = self._runner(klass=MockedHandleStdin)
    runner.run(_, asynchronous=True).join()
    
    # 断言：handle_stdin 从未被调用
    assert not MockedHandleStdin.handle_stdin.called
```

### 2.3 输出流的自动隐藏

**隐藏逻辑：**

```python
# runners.py:574-575
if self._asynchronous:
    opts["hide"] = True  # 异步模式强制隐藏输出
```

**与自定义流的处理：**

```python
# runners.py:1710-1713
# normalize_hide 函数中：
if out_stream is not None and "stdout" in hide:
    hide.remove("stdout")  # 如果用户指定了自定义流，则不隐藏
if err_stream is not None and "stderr" in hide:
    hide.remove("stderr")
```

**测试验证：**

```python
# tests/runners.py:1465-1471
@trap
def hides_output(self):
    self._runner(out="foo", err="bar").run(_, asynchronous=True).join()
    # 默认流没有输出
    assert sys.stdout.getvalue() == ""
    assert sys.stderr.getvalue() == ""

# tests/runners.py:1485-1501
def leaves_overridden_streams_alone(self):
    out, err = StringIO(), StringIO()
    runner.run(
        _,
        asynchronous=True,
        out_stream=out,  # 用户指定的流
        err_stream=err,
    ).join()
    # 自定义流仍然会收到数据
    assert out.getvalue() == "foo"
    assert err.getvalue() == "bar"
```

### 2.4 Promise 类的设计

```python
# runners.py:1618-1686
class Promise(Result, AbstractContextManager):
    """
    异步执行的承诺对象。
    继承自 Result 和上下文管理器。
    """
    
    runner: Runner  # 关联的 Runner 实例
    
    def __init__(self, runner: "Runner") -> None:
        self.runner = runner
        # 复制 Runner 的 result_kwargs
        for key, value in self.runner.result_kwargs.items():
            setattr(self, key, value)
    
    def join(self) -> Result:
        """
        阻塞等待子进程完成。
        行为与同步 run() 的结束阶段相同。
        """
        try:
            return self.runner._finish()
        finally:
            self.runner.stop()
    
    def __enter__(self) -> "Promise":
        return self
    
    def __exit__(self, exc_type, exc_value, exc_tb) -> None:
        self.join()  # 上下文退出时自动 join
```

### 2.5 异步执行的完整流程

```
调用 runner.run(command, asynchronous=True)
         ↓
    _setup() 配置
         ↓
    start() 启动子进程（后台运行）
         ↓
    create_io_threads() 创建 IO 线程
         ↓
    启动 IO 线程
         ↓
    返回 Promise（不等待进程完成）
         ↓
┌───────────────────────────────────────────┐
│  用户可以：                                │
│  - 做其他工作                             │
│  - 使用 Promise 作为上下文管理器           │
│  - 稍后调用 promise.join()                │
└───────────────────────────────────────────┘
         ↓
    promise.join() 被调用
         ↓
    _finish() 执行：
    - wait() 等待进程完成
    - join() IO 线程
    - 收集异常
    - 生成 Result
         ↓
    返回 Result 或抛出异常
```

### 2.6 异步模式的异常处理

**异常收集路径：**

```python
# runners.py:479-537
def _finish(self) -> "Result":
    try:
        while True:
            try:
                self.wait()
                break
            except KeyboardInterrupt as e:
                self.send_interrupt(e)
    finally:
        self.program_finished.set()
        # 收集线程异常
        watcher_errors = []
        thread_exceptions = []
        for target, thread in self.threads.items():
            thread.join(self._thread_join_timeout(target))
            exception = thread.exception()
            if exception is not None:
                real = exception.value
                if isinstance(real, WatcherError):
                    watcher_errors.append(real)
                else:
                    thread_exceptions.append(exception)
    
    # 线程异常抛出
    if thread_exceptions:
        raise ThreadException(thread_exceptions)
    
    # WatcherError 处理
    result = self._collate_result(watcher_errors)
    if watcher_errors:
        raise Failure(result, reason=watcher_errors[0])
    
    # 超时检测
    if timeout is not None and self.timed_out:
        raise CommandTimedOut(result, timeout=timeout)
    
    # 非零退出码
    if not (result or self.opts["warn"]):
        raise UnexpectedExit(result)
    
    return result
```

### 2.7 disown 模式与 asynchronous 模式的对比

| 特性 | asynchronous | disown |
|------|--------------|--------|
| **IO 线程** | ✅ 创建 | ❌ 不创建 |
| **返回值** | Promise | 有限的 Result |
| **join()** | ✅ 可用 | ❌ 不可用 |
| **stdout/stderr 捕获** | ✅ 完整捕获 | ❌ 空字符串 |
| **退出码** | ✅ 可用 | ❌ None |
| **stdin 转发** | ❌ 禁用 | ❌ 禁用 |
| **超时处理** | ✅ 支持 | ❌ 不支持 |
| **Watcher** | ✅ 支持 | ❌ 不支持 |

**disown 模式的代码路径：

```python
# runners.py:451-462
if self._disowned:
    # 直接返回，不创建任何线程
    return self.generate_result(
        **dict(
            self.result_kwargs,
            stdout="",      # 空
            stderr="",        # 空
            exited=None,      # 无退出码
            disowned=True,
        )
    )
```

---

## 3. pty 初始化阶段的预留路径分析

### 3.1 关键 TODO 注释分析

```python
# runners.py:1346-1350
if self.pid == 0:  # 子进程中
    # TODO: both pty.spawn() and pexpect.spawn() do a lot of
    # setup/teardown involving tty.setraw, getrlimit, signal.
    # Ostensibly we'll want some of that eventually, but if
    # possible write tests - integration-level if necessary -
    # before adding it!
```

这个 TODO 指向了三个未实现的领域：
1. **`tty.setraw` - 终端模式设置
2. **`getrlimit`** - 资源限制
3. **`signal`** - 信号处理

### 3.2 tty.setraw 的预期作用

**当前实现（仅设置窗口大小）：

```python
# runners.py:1352-1356
# Set pty window size based on what our own controlling
# terminal's window size appears to be.
winsize = struct.pack("HHHH", rows, cols, 0, 0)
fcntl.ioctl(sys.stdout.fileno(), termios.TIOCSWINSZ, winsize)
```

**pty.spawn() 和 pexpect.spawn() 的做法：**

```python
# Python 标准库 pty.spawn 的典型实现模式：

def _setraw(fd):
    """设置终端为原始模式"""
    mode = termios.tcgetattr(fd)
    mode[0] = mode[0] & ~(termios.IGNBRK | termios.BRKINT | 
                           termios.PARMRK | termios.ISTRIP |
                           termios.INLCR | termios.IGNCR | 
                           termios.ICRNL | termios.IXON)
    mode[1] = mode[1] & ~termios.OPOST
    mode[2] = mode[2] & ~(termios.CSIZE | termios.PARENB)
    mode[2] = mode[2] | termios.CS8
    mode[3] = mode[3] & ~(termios.ECHO | termios.ECHONL | 
                           termios.ICANON | termios.ISIG | 
                           termios.IEXTEN)
    termios.tcsetattr(fd, termios.TCSANOW, mode)
```

**setraw vs cbreak 的区别：**

| 特性 | cbreak (Invoke 当前) | setraw (TODO) |
|------|----------------------|---------------|
| **输入处理** | 字符缓冲关闭，信号仍有效 | 完全原始，无信号处理 |
| **输出处理** | 保持正常 | OPOST 关闭，无输出处理 |
| **信号** | SIGINT/SIGQUIT 有效 | 所有信号禁用 |
| **适用场景** | 交互式程序 | 完全控制终端 |

**潜在影响：**
- **不设置 setraw 的后果：
  - 子进程的终端可能继承父进程的某些设置
  - 某些需要原始模式的程序可能行为异常
  - 特殊字符（如 Ctrl+S/Ctrl+Q 流控制）可能干扰

### 3.3 getrlimit/setrlimit 资源限制

**资源限制的作用：**

```python
import resource

# 典型的 pty.spawn 可能设置的资源限制：

# 1. 核心转储文件大小
resource.setrlimit(resource.RLIMIT_CORE, (0, 0))

# 2. 文件描述符限制
soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)

# 3. 进程数限制
soft, hard = resource.getrlimit(resource.RLIMIT_NPROC)
```

**为什么需要资源限制：**

| 资源类型 | 作用 | 不设置的风险 |
|-----------|------|-------------|
| RLIMIT_CORE | 限制核心转储大小 | 子进程崩溃时产生巨大的核心文件 |
| RLIMIT_NOFILE | 限制打开文件数 | 子进程继承所有父进程 FD |
| RLIMIT_NPROC | 限制子进程数 | 防止 fork bomb |

**当前 Invoke 的现状：**
- 没有设置任何资源限制
- 子进程完全继承父进程的资源限制

### 3.4 信号处理（signal）

**当前已实现的信号处理：**

```python
# runners.py:490-491
except KeyboardInterrupt as e:
    self.send_interrupt(e)

# runners.py:1156-1172
def send_interrupt(self, interrupt: "KeyboardInterrupt") -> None:
    self.write_proc_stdin("\x03")  # 发送 Ctrl+C
```

**TODO 中提到的信号处理：**

```python
# runners.py:492
# TODO: honor other signals sent to our own process and
# transmit them to the subprocess before handling 'normally'.
```

**pty.spawn/pexpect 通常处理的信号：**

| 信号 | 预期处理 |
|------|-----------|
| SIGINT | 转发给子进程（已部分实现） |
| SIGTERM | 终止子进程 |
| SIGWINCH | 窗口大小变更 → 更新 pty 窗口大小 |
| SIGHUP | 挂起信号 |
| SIGCONT | 继续执行 |

**SIGWINCH 信号的重要性：**

```python
# 伪代码：理想的 SIGWINCH 处理

def _handle_sigwinch(self, signum, frame):
    if self.using_pty and self.parent_fd:
        # 获取新的窗口大小
        cols, rows = pty_size()
        # 更新 pty 的窗口大小
        winsize = struct.pack("HHHH", rows, cols, 0, 0)
        fcntl.ioctl(self.parent_fd, termios.TIOCSWINSZ, winsize)
        # 同时向子进程发送 SIGWINCH
        os.kill(self.pid, signal.SIGWINCH)

# 安装信号处理器
signal.signal(signal.SIGWINCH, _handle_sigwinch)
```

**不处理 SIGWINCH 的影响：**

1. 用户调整终端窗口大小时
2. 全屏程序（vim、less、tmux 等）不会收到通知
3. 程序显示可能不会重绘，出现显示错乱

### 3.5 预留路径的架构分析

**当前 pty 初始化流程：**

```
pty.fork()
     ↓
子进程 (pid == 0)
     ↓
设置窗口大小 (TIOCSWINSZ)
     ↓
os.execvpe() 替换进程
```

**理想的完整流程（参考 pty.spawn）：**

```
pty.fork()
     ↓
子进程 (pid == 0)
     ↓
┌─────────────────────────────────┐
│ TODO: 未实现的步骤               │
│ 1. tty.setraw() - 设置原始模式  │
│ 2. 信号处理设置                 │
│ 3. 资源限制调整                 │
└─────────────────────────────────┘
     ↓
设置窗口大小 (TIOCSWINSZ)
     ↓
os.execvpe() 替换进程
```

### 3.6 潜在影响与风险评估

| 未实现功能 | 影响程度 | 触发场景 | 风险等级 |
|-----------|---------|---------|---------|
| tty.setraw | 中 | 需要原始终端模式的程序 | 中 |
| 资源限制 | 低 | 长时间运行/大量 FD 的程序 | 低 |
| SIGWINCH 处理 | 中 | 交互式全屏程序 | 中 |
| 其他信号转发 | 低 | 信号密集场景 | 低 |

**实际案例分析：**

1. **sudo 密码输入场景：**
   - 当前实现：工作正常
   - 需要：FailingResponder 检测密码错误
   - 不依赖 setraw/信号处理

2. **vim 编辑场景：**
   - 当前实现：可能有显示问题
   - 需要：SIGWINCH 处理、正确终端模式
   - 风险：窗口大小变更时显示错乱

3. **长时间运行的后台任务：**
   - 当前实现：工作正常
   - 需要：资源限制（可选）
   - 风险：文件描述符泄漏

### 3.7 实现建议

**分阶段实现路线图：**

**阶段 1：SIGWINCH 处理（高实用）

```python
# 在 Runner 类中添加：

def __init__(self, context):
    # ... 现有初始化 ...
    self._original_winch_handler = None

def _handle_sigwinch(self, signum, frame):
    if self.using_pty and hasattr(self, 'parent_fd'):
        cols, rows = pty_size()
        winsize = struct.pack("HHHH", rows, cols, 0, 0)
        fcntl.ioctl(self.parent_fd, termios.TIOCSWINSZ, winsize)

def start(self, command, shell, env):
    if self.using_pty:
        # 安装信号处理器
        self._original_winch_handler = signal.signal(
            signal.SIGWINCH, self._handle_sigwinch
        )
        # ... 现有 pty 逻辑 ...

def stop(self):
    # 恢复原始信号处理器
    if self._original_winch_handler:
        signal.signal(signal.SIGWINCH, self._original_winch_handler)
    super().stop()
```

**阶段 2：可选的终端模式设置**

提供配置选项让用户选择：

```python
# config.py 中添加：
"run": {
    # ... 现有配置 ...
    "pty_raw": False,  # 是否使用原始模式
    "pty_restore_signals": True,  # 是否恢复信号
}
```

---

## 附录：关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| ExceptionHandlingThread | util.py | 146-257 |
| StreamWatcher (threading.local) | watchers.py | 8-36 |
| 异步模式 stdin 禁用 | runners.py | 585-590 |
| 异步模式输出隐藏 | runners.py | 574-575 |
| Promise 类 | runners.py | 1618-1686 |
| disown 模式 | runners.py | 451-462 |
| pty 初始化 TODO | runners.py | 1346-1350 |
| pty 窗口大小设置 | runners.py | 1352-1356 |
| 信号转发 TODO | runners.py | 492 |

---

## 总结

### 并发与异步执行模式的核心设计要点：

1. **并发模型**：
   - Invoke 核心是串行执行，但提供了 `ExceptionHandlingThread` 作为并发基础
   - `StreamWatcher` 继承 `threading.local` 保证了 stdout/stderr 线程间的状态隔离

2. **异步模式**：
   - `asynchronous=True` 返回 `Promise`，需手动 `join()`
   - 自动禁用 stdin 转发和终端输出
   - 自定义流（`out_stream`/`err_stream`）不受隐藏影响
   - `disown=True` 是更轻量的异步，完全放弃控制

3. **pty 预留路径**：
   - TODO 注释指向 `tty.setraw`、`getrlimit`、`signal` 三个未实现领域
   - 最具实用价值的是 **SIGWINCH 信号处理**，用于窗口大小变更
   - 当前实现对于大多数自动化场景已足够，交互式场景可能有局限
