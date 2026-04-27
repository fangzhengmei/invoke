# Invoke Runner 与 Config 体系深度分析报告

## 1. 概述

本文档深入分析 invoke 库中 `Runner` 体系和 `Config`/`DataProxy` 体系的实现机制，包括：
- `run()` 方法内部 `_run_body`、`_finish` 等方法的协同机制
- `ExceptionHandlingThread` 的设计意图和生命周期管理
- `Config` 对象的合并/克隆机制及与 `DataProxy` 的关系

---

## 2. Runner 体系核心分析

### 2.1 体系架构概览

`Runner` 是 invoke 中执行命令的核心抽象类，其架构层次如下：

```
Runner (抽象基类)
  ├── 核心方法: run(), _run_body(), _finish()
  ├── IO 线程: handle_stdout(), handle_stderr(), handle_stdin()
  └── 子类: Local (本地执行器)
```

### 2.2 `run()` 方法 - 入口包装器

**关键代码位置**：`invoke/runners.py:126-406`

`run()` 方法是一个轻量级包装器，主要职责：
1. 委托实际工作给 `_run_body()`
2. 在 finally 块中调用 `stop()` 进行资源清理

```python
def run(self, command: str, **kwargs: Any) -> "Result":
    try:
        return self._run_body(command, **kwargs)
    finally:
        if not (self._asynchronous or self._disowned):
            self.stop()
```

### 2.3 `_run_body()` 方法 - 子进程调度核心

**关键代码位置**：`invoke/runners.py:439-469`

`_run_body()` 是整个执行流程的核心，按以下顺序协同工作：

#### 阶段 1: 准备阶段 (`_setup`)

**关键代码位置**：`invoke/runners.py:411-437`

```python
def _setup(self, command: str, kwargs: Any) -> None:
    # 1. 统一 kwargs 和 config，设置 self.opts 和 self.streams
    self._unify_kwargs_with_config(kwargs)
    # 2. 生成环境变量
    self.env = self.generate_env(self.opts["env"], self.opts["replace_env"])
    # 3. 确定编码
    self.encoding = self.opts["encoding"] or self.default_encoding()
    # 4. 如果开启 echo，打印命令
    if self.opts["echo"]:
        self.echo(command)
    # 5. 准备 result_kwargs，供后续生成 Result 使用
    self.result_kwargs = dict(
        command=command,
        shell=self.opts["shell"],
        env=self.env,
        pty=self.using_pty,
        hide=self.opts["hide"],
        encoding=self.encoding,
    )
```

#### 阶段 2: 干运行检查

如果 `dry=True`，直接返回模拟的 `Result`，不启动实际进程：

```python
if self.opts["dry"]:
    return self.generate_result(
        **dict(self.result_kwargs, stdout="", stderr="", exited=0)
    )
```

#### 阶段 3: 启动子进程 (`start`)

**关键代码位置**：`invoke/runners.py:1335-1371` (Local.start)

`Local.start()` 支持两种模式：

1. **PTY 模式**（`pty=True`）：使用 `pty.fork()` 创建伪终端
2. **普通模式**（默认）：使用 `subprocess.Popen`

```python
def start(self, command: str, shell: str, env: Dict[str, Any]) -> None:
    if self.using_pty:
        # PTY 模式：fork 子进程
        self.pid, self.parent_fd = pty.fork()
        if self.pid == 0:
            # 子进程：执行实际命令
            os.execvpe(shell, [shell, "-c", command], env)
    else:
        # 普通模式：使用 Popen
        self.process = Popen(
            command,
            shell=True,
            executable=shell,
            env=env,
            stdout=PIPE,
            stderr=PIPE,
            stdin=PIPE,
        )
```

启动后获取 PID：

```python
self.result_kwargs["pid"] = self.get_pid()
```

#### 阶段 4: disowned 模式检查

如果 `disown=True`，直接返回有限的 `Result`，不启动 IO 线程：

```python
if self._disowned:
    return self.generate_result(
        **dict(
            self.result_kwargs,
            stdout="",
            stderr="",
            exited=None,
            disowned=True,
        )
    )
```

#### 阶段 5: 启动 IO 线程和定时器

**关键代码位置**：`invoke/runners.py:464-467`

```python
# 启动超时定时器
self.start_timer(self.opts["timeout"])
# 创建 IO 线程（stdout、stderr、stdin）
self.threads, self.stdout, self.stderr = self.create_io_threads()
# 启动所有线程
for thread in self.threads.values():
    thread.start()
```

#### 阶段 6: 异步/同步决策

**关键代码位置**：`invoke/runners.py:469`

```python
return self.make_promise() if self._asynchronous else self._finish()
```

- **异步模式**：返回 `Promise` 对象，由调用者决定何时 `join()`
- **同步模式**：直接调用 `_finish()` 等待完成

### 2.4 `_finish()` 方法 - 结果收集与异常处理

**关键代码位置**：`invoke/runners.py:479-537`

`_finish()` 负责等待子进程完成并收集结果，其流程如下：

#### 阶段 1: 等待子进程完成

**关键代码位置**：`invoke/runners.py:481-493`

```python
try:
    while True:
        try:
            self.wait()
            break  # 等待完成
        except KeyboardInterrupt as e:
            # 捕获 Ctrl+C，转发给子进程
            self.send_interrupt(e)
```

`wait()` 方法的实现（`invoke/runners.py:1008-1021`）：

```python
def wait(self) -> None:
    while True:
        proc_finished = self.process_is_finished
        dead_threads = self.has_dead_threads
        if proc_finished or dead_threads:
            break
        time.sleep(self.input_sleep)
```

等待条件：
- 子进程已完成（`process_is_finished`）
- 或任何 IO 线程异常死亡（`has_dead_threads`）

#### 阶段 2: 清理 IO 线程

**关键代码位置**：`invoke/runners.py:497-514`

```python
finally:
    # 1. 通知 stdin 线程停止循环
    self.program_finished.set()
    # 2. join 所有线程，收集异常
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
```

**关键点**：
- `program_finished.set()` 是一个 `threading.Event`，用于通知 `handle_stdin` 退出无限循环
- 使用 `ExceptionHandlingThread.exception()` 获取线程中捕获的异常
- 区分 `WatcherError`（预期错误，如自动响应失败）和其他异常

#### 阶段 3: 异常处理

**关键代码位置**：`invoke/runners.py:520-537`

```python
# 1. 非 WatcherError 的线程异常 -> 抛出 ThreadException
if thread_exceptions:
    raise ThreadException(thread_exceptions)

# 2. 整理结果
result = self._collate_result(watcher_errors)

# 3. WatcherError -> 包装为 Failure 抛出
if watcher_errors:
    raise Failure(result, reason=watcher_errors[0])

# 4. 超时检查
timeout = self.opts["timeout"]
if timeout is not None and self.timed_out:
    raise CommandTimedOut(result, timeout=timeout)

# 5. 退出码检查
if not (result or self.opts["warn"]):
    raise UnexpectedExit(result)

return result
```

### 2.5 IO 线程的协同机制

**关键代码位置**：`invoke/runners.py:643-683`

`create_io_threads()` 创建三个可能的 IO 线程：

```python
def create_io_threads(self) -> Tuple[Dict[Callable, ExceptionHandlingThread], List[str], List[str]]:
    stdout: List[str] = []
    stderr: List[str] = []
    
    thread_args: Dict[Callable, Any] = {
        # stdout 处理线程：总是创建
        self.handle_stdout: {
            "buffer_": stdout,
            "hide": "stdout" in self.opts["hide"],
            "output": self.streams["out"],
        }
    }
    
    # stdin 处理线程：仅当 in_stream 存在时创建
    if self.streams["in"]:
        thread_args[self.handle_stdin] = {
            "input_": self.streams["in"],
            "output": self.streams["out"],
            "echo": self.opts["echo_stdin"],
        }
    
    # stderr 处理线程：仅当不使用 PTY 时创建
    if not self.using_pty:
        thread_args[self.handle_stderr] = {
            "buffer_": stderr,
            "hide": "stderr" in self.opts["hide"],
            "output": self.streams["err"],
        }
    
    # 包装为 ExceptionHandlingThread
    threads = {}
    for target, kwargs in thread_args.items():
        t = ExceptionHandlingThread(target=target, kwargs=kwargs)
        threads[target] = t
    
    return threads, stdout, stderr
```

#### 线程协作细节

**1. stdout/stderr 线程**（`invoke/runners.py:750-810`）：

```python
def _handle_output(
    self,
    buffer_: List[str],
    hide: bool,
    output: IO,
    reader: Callable,
) -> None:
    for data in self.read_proc_output(reader):
        # 1. 回显到终端（如果不 hide）
        if not hide:
            self.write_our_output(stream=output, string=data)
        # 2. 追加到共享 buffer
        buffer_.append(data)
        # 3. 触发自动响应器（如 sudo 密码输入）
        self.respond(buffer_)
```

**关键点**：
- `buffer_` 是主线程和 IO 线程共享的列表
- 注释说明：`this is threadsafe insofar as no reading occurs until after the thread is join()'d`
- 只有在线程 `join()` 后主线程才会读取 `buffer_`，因此无需锁

**2. stdin 线程**（`invoke/runners.py:852-916`）：

```python
def handle_stdin(
    self,
    input_: IO,
    output: IO,
    echo: bool = False,
) -> None:
    with character_buffered(input_):
        while True:
            data = self.read_our_stdin(input_)
            if data:
                # 1. 写入子进程 stdin
                self.write_proc_stdin(data)
                # 2. 回显到 stdout（如果需要）
                if echo:
                    self.write_our_output(stream=output, string=data)
            elif data is not None:
                # EOF 信号：关闭子进程 stdin
                if not self.using_pty and not closed_stdin:
                    self.close_proc_stdin()
                    closed_stdin = True
            
            # 退出条件：程序完成 + 无数据可读
            if self.program_finished.is_set() and not data:
                break
            
            # 避免 CPU 空转
            time.sleep(self.input_sleep)
```

**关键点**：
- `program_finished.is_set()` 由 `_finish()` 在 finally 块中设置
- 这是一个**无限循环**，必须依赖外部信号退出

### 2.6 Promise 异步模式

**关键代码位置**：`invoke/runners.py:1618-1687`

当 `asynchronous=True` 时，`_run_body()` 返回 `Promise` 而非 `Result`：

```python
class Promise(Result, AbstractContextManager):
    runner: Runner
    
    def __init__(self, runner: "Runner") -> None:
        self.runner = runner
        # 复制 result_kwargs 中的属性
        for key, value in self.runner.result_kwargs.items():
            setattr(self, key, value)
    
    def join(self) -> Result:
        try:
            return self.runner._finish()
        finally:
            self.runner.stop()
    
    def __enter__(self) -> "Promise":
        return self
    
    def __exit__(self, ...) -> None:
        self.join()
```

**使用模式**：

```python
# 方式 1：手动 join
promise = c.run("long-running-command", asynchronous=True)
# ... 做其他事情 ...
result = promise.join()

# 方式 2：上下文管理器
with c.run("long-running-command", asynchronous=True) as promise:
    # ... 做其他事情 ...
# 退出时自动 join
```

### 2.7 Runner 执行流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           run(command, **kwargs)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         _run_body(command, **kwargs)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. _setup()                                                                   │
│     ├── 统一 kwargs 与 config → self.opts                                     │
│     ├── 生成环境变量 self.env                                                  │
│     ├── 确定编码 self.encoding                                                 │
│     ├── 可选：echo 打印命令                                                    │
│     └── 准备 result_kwargs                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  2. dry-run 检查                                                              │
│     └── if dry=True → return dummy Result                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  3. start() 启动子进程                                                        │
│     ├── PTY 模式: pty.fork() + os.execvpe()                                  │
│     └── 普通模式: subprocess.Popen(stdout/stderr/stdin=PIPE)                │
├─────────────────────────────────────────────────────────────────────────────┤
│  4. disowned 检查                                                             │
│     └── if disown=True → return 有限 Result（无 stdout/stderr）             │
├─────────────────────────────────────────────────────────────────────────────┤
│  5. 启动 IO 线程和定时器                                                      │
│     ├── start_timer(timeout) → 超时后 kill 子进程                            │
│     ├── create_io_threads()                                                   │
│     │   ├── handle_stdout (总是创建)                                         │
│     │   ├── handle_stderr (非 PTY 时创建)                                    │
│     │   └── handle_stdin (in_stream 存在时创建)                              │
│     └── thread.start() 启动所有线程                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  6. 异步/同步决策                                                              │
│     ├── asynchronous=True → return Promise()                                 │
│     └── asynchronous=False → call _finish()                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              ▼                                               ▼
┌─────────────────────────┐                   ┌─────────────────────────────────┐
│    Promise 对象         │                   │           _finish()              │
├─────────────────────────┤                   ├─────────────────────────────────┤
│ - 继承 Result 属性       │                   │  1. wait() 循环等待              │
│ - join() 调用 _finish() │                   │    └── 捕获 KeyboardInterrupt    │
│ - 支持上下文管理器       │                   │        并转发给子进程            │
└─────────────────────────┘                   ├─────────────────────────────────┤
                                              │  2. finally 块清理                │
                                              │     ├── program_finished.set()    │
                                              │     ├── thread.join()             │
                                              │     └── 收集线程异常              │
                                              ├─────────────────────────────────┤
                                              │  3. 异常处理                      │
                                              │     ├── ThreadException (普通异常)│
                                              │     ├── Failure (WatcherError)   │
                                              │     ├── CommandTimedOut (超时)    │
                                              │     └── UnexpectedExit (非0退出码)│
                                              ├─────────────────────────────────┤
                                              │  4. return Result                 │
                                              └─────────────────────────────────┘
```

---

## 3. ExceptionHandlingThread 深度分析

### 3.1 设计意图

**关键代码位置**：`invoke/util.py:146-257`

`ExceptionHandlingThread` 的核心设计意图是：**捕获子线程中的异常，使其能够被主线程感知和处理**。

在标准 Python `threading.Thread` 中：
- 子线程的异常只会打印到 stderr
- 主线程无法知道子线程是否发生了异常
- 异常信息（类型、值、traceback）会丢失

`ExceptionHandlingThread` 解决了这个问题：

```python
class ExceptionHandlingThread(threading.Thread):
    def __init__(self, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.daemon = True  # 守护线程，不阻塞主进程退出
        self.kwargs = kwargs  # 保存创建参数，用于异常报告
        self.exc_info: Optional[Tuple[Type[BaseException], BaseException, TracebackType], ...] = None
```

### 3.2 核心实现机制

#### 异常捕获

**关键代码位置**：`invoke/util.py:187-224`

```python
def run(self) -> None:
    try:
        # 两种使用方式：
        # 1. 直接使用：提供 target 参数，调用 super().run()
        # 2. 子类化：定义 _run() 方法
        if hasattr(self, "_run") and callable(self._run):
            self._run()
        else:
            super().run()
    except BaseException:
        # 捕获所有异常，保存 exc_info
        self.exc_info = sys.exc_info()
        # 立即记录日志，方便调试
        msg = "Encountered exception {!r} in thread for {!r}"
        name = "_run"
        if "target" in self.kwargs:
            name = self.kwargs["target"].__name__
        debug(msg.format(self.exc_info[1], name))
```

#### 异常获取

**关键代码位置**：`invoke/util.py:225-238`

```python
def exception(self) -> Optional["ExceptionWrapper"]:
    if self.exc_info is None:
        return None
    # 包装为 ExceptionWrapper namedtuple
    return ExceptionWrapper(self.kwargs, *self.exc_info)
```

`ExceptionWrapper` 定义（`invoke/util.py:266-268`）：

```python
ExceptionWrapper = namedtuple(
    "ExceptionWrapper", "kwargs type value traceback"
)
```

包含信息：
- `kwargs`：线程创建时的参数（用于识别是哪个线程）
- `type`：异常类型
- `value`：异常值
- `traceback`：traceback 对象

#### 死亡检测

**关键代码位置**：`invoke/util.py:240-252`

```python
@property
def is_dead(self) -> bool:
    """
    返回 True 当线程已停止且有异常存储
    """
    return (not self.is_alive()) and self.exc_info is not None
```

`Runner.wait()` 使用这个属性来检测线程是否异常死亡：

```python
@property
def has_dead_threads(self) -> bool:
    return any(x.is_dead for x in self.threads.values())
```

### 3.3 生命周期管理

`ExceptionHandlingThread` 的完整生命周期：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        阶段 1: 创建 (__init__)                                │
├──────────────────────────────────────────────────────────────────────────────┤
│  ExceptionHandlingThread(target=handle_stdout, kwargs={...})                │
│  ├── super().__init__(**kwargs)                                               │
│  ├── self.daemon = True                                                       │
│  ├── self.kwargs = kwargs  (保存用于异常报告)                                 │
│  └── self.exc_info = None                                                     │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        阶段 2: 启动 (start())                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│  thread.start() → 触发 run() 方法                                             │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        阶段 3: 执行 (run())                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│  try:                                                                          │
│      if has _run: self._run()         ──┐                                    │
│      else: super().run()              ──┤ 正常执行路径                       │
│  except BaseException:                    │                                    │
│      self.exc_info = sys.exc_info()  ─────┘ 异常捕获路径                     │
│      debug(...)                             记录日志                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              ▼                                               ▼
┌─────────────────────────┐                   ┌─────────────────────────────────┐
│     正常完成            │                   │         异常完成                  │
├─────────────────────────┤                   ├─────────────────────────────────┤
│ self.exc_info = None    │                   │ self.exc_info = (type, value,   │
│ is_dead = False         │                   │              traceback)           │
│ is_alive() = False      │                   │ is_dead = True                  │
└─────────────────────────┘                   │ is_alive() = False               │
                                              └─────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        阶段 4: 主线程收集                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│  thread.join(timeout)                                                         │
│  exception = thread.exception()                                               │
│  if exception is not None:                                                    │
│      # 处理异常：ThreadException 或 Failure                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 与 Runner 的集成

**关键代码位置**：`invoke/runners.py:504-521`

`_finish()` 中收集和处理线程异常：

```python
watcher_errors = []
thread_exceptions = []
for target, thread in self.threads.items():
    thread.join(self._thread_join_timeout(target))
    exception = thread.exception()
    if exception is not None:
        real = exception.value
        if isinstance(real, WatcherError):
            # WatcherError 是"预期"错误，单独收集
            watcher_errors.append(real)
        else:
            # 其他异常，收集为 ExceptionWrapper
            thread_exceptions.append(exception)

# 非 WatcherError 异常 -> 抛出 ThreadException
if thread_exceptions:
    raise ThreadException(thread_exceptions)
```

**ThreadException**（`invoke/exceptions.py`）是一个聚合异常，包含所有线程的异常信息。

### 3.5 两种使用模式

#### 模式 1：直接使用 target（Runner 采用的方式）

```python
# 创建时指定 target
t = ExceptionHandlingThread(target=handle_stdout, kwargs={"buffer_": stdout, ...})
t.start()
t.join()
exc = t.exception()
```

#### 模式 2：子类化并定义 _run()

```python
class MyThread(ExceptionHandlingThread):
    def __init__(self, queue):
        super().__init__()
        self.queue = queue
    
    def _run(self):
        # 这里的异常会被自动捕获
        self.queue.put(7)
        # 可能抛出异常的代码...
```

### 3.6 为什么 Executor 不直接使用？

在 R1 报告中提到 `ExceptionHandlingThread` 存在但未被核心 `Executor` 使用。原因分析：

1. **职责不同**：
   - `Executor` 负责**任务调度**（pre/post 展开、去重、执行顺序）
   - `Runner` 负责**单个命令执行**（子进程、IO 线程）
   - `ExceptionHandlingThread` 是 **IO 级别的工具**，用于捕获 stdin/stdout/stderr 处理线程的异常

2. **Executor 的任务执行是同步的**：
   ```python
   # executor.py:124-148
   for call in calls:
       context = call.make_context(config, ...)
       result = call.task(*args, **call.kwargs)  # 同步调用
   ```
   - 任务函数本身是同步执行的
   - 只有当任务函数**内部**调用 `c.run()` 时，才会触发 `Runner` 和 `ExceptionHandlingThread`

3. **继承关系**：
   - `ExceptionHandlingThread` 来自 Fabric 1 的遗产
   - Fabric 2 可能在远程执行场景中更多地使用这类线程机制

---

## 4. Config 与 DataProxy 体系分析

### 4.1 体系架构概览

```
DataProxy (基类)
  ├── 提供嵌套 dict + attr 访问能力
  └── Config 继承自 DataProxy
      
Config (配置管理核心)
  ├── 多层级配置管理
  ├── 合并机制
  ├── 克隆机制
  └── 运行时修改追踪
```

### 4.2 DataProxy - 嵌套访问代理

**关键代码位置**：`invoke/config.py:36-312`

`DataProxy` 的核心价值是：**让嵌套字典可以同时使用 dict 语法和属性语法访问**。

#### 核心设计思想

```python
# 传统 dict 访问：
config["run"]["echo"] = True

# DataProxy 支持：
config.run.echo = True  # 属性语法
```

#### 属性访问实现

**关键代码位置**：`invoke/config.py:111-188`

```python
def __getattr__(self, key: str) -> Any:
    try:
        return self._get(key)
    except KeyError:
        # 特殊方法代理到 _config
        if key in self._proxies:
            return getattr(self._config, key)
        raise AttributeError(...)

def _get(self, key: str) -> Any:
    value = self._config[key]
    # 嵌套 dict 自动包装为 DataProxy
    if isinstance(value, dict):
        keypath = (key,)
        if hasattr(self, "_keypath"):
            keypath = self._keypath + keypath
        root = getattr(self, "_root", self)
        value = DataProxy.from_data(data=value, root=root, keypath=keypath)
    return value
```

#### 属性设置实现

**关键代码位置**：`invoke/config.py:131-140`

```python
def __setattr__(self, key: str, value: Any) -> None:
    has_real_attr = key in dir(self)
    if not has_real_attr:
        # 不是真实属性，作为 config key 处理
        self[key] = value
    else:
        # 是真实属性，正常设置
        super().__setattr__(key, value)
```

#### 真实属性的特殊处理

**关键代码位置**：`invoke/config.py:190-206`

由于 `__setattr__` 会拦截所有属性设置，`DataProxy` 提供 `_set()` 方法来设置"真实属性"：

```python
def _set(self, *args: Any, **kwargs: Any) -> None:
    """
    使用 object.__setattr__ 绕过代理行为
    """
    if args:
        object.__setattr__(self, *args)
    for key, value in kwargs.items():
        object.__setattr__(self, key, value)
```

**使用示例**（Config `__init__` 中）：

```python
def __init__(self, ...):
    # 使用 _set 设置真实属性
    self._set(_config={})
    self._set(_defaults=defaults)
    self._set(_collection={})
    # ...
```

#### 根-叶关系

`DataProxy` 支持**根节点**和**叶子节点**的概念：

```python
@property
def _is_leaf(self) -> bool:
    return hasattr(self, "_root")

@property
def _is_root(self) -> bool:
    return hasattr(self, "_modify")
```

当叶子节点被修改时，会通知根节点：

```python
def _track_modification_of(self, key: str, value: str) -> None:
    target = None
    if self._is_leaf:
        target = self._root
    elif self._is_root:
        target = self
    if target is not None:
        # 调用根节点的 _modify 方法
        target._modify(getattr(self, "_keypath", tuple()), key, value)
```

### 4.3 Config - 多层级配置管理

**关键代码位置**：`invoke/config.py:313-1161`

`Config` 继承 `DataProxy`，添加了**多层级配置管理**能力。

#### 配置层级设计

**关键代码位置**：`invoke/config.py:512-655`

```python
def __init__(self, ...):
    # 层级 1: defaults (最低优先级)
    self._set(_defaults=defaults)
    
    # 层级 2: collection
    self._set(_collection={})
    
    # 层级 3: system
    self._set(_system={})
    
    # 层级 4: user
    self._set(_user={})
    
    # 层级 5: project
    self._set(_project={})
    
    # 层级 6: env
    self._set(_env={})
    
    # 层级 7: runtime
    self._set(_runtime={})
    
    # 层级 8: overrides
    self._set(_overrides=overrides)
    
    # 层级 9: modifications (最高优先级 - 运行时修改)
    self._set(_modifications={})
    
    # 特殊: deletions (删除追踪)
    self._set(_deletions={})
    
    # 合并所有层级到 _config
    self.merge()
```

#### 配置优先级（从低到高）

```
defaults < collection < system < user < project < env < runtime < overrides < modifications
```

#### 合并机制

**关键代码位置**：`invoke/config.py:941-964`

```python
def merge(self) -> None:
    # 1. 从空 dict 开始
    self._set(_config={})
    
    # 2. 按优先级顺序合并
    merge_dicts(self._config, self._defaults)
    merge_dicts(self._config, self._collection)
    self._merge_file("system", "System-wide")
    self._merge_file("user", "Per-user")
    self._merge_file("project", "Per-project")
    merge_dicts(self._config, self._env)
    self._merge_file("runtime", "Runtime")
    merge_dicts(self._config, self._overrides)
    merge_dicts(self._config, self._modifications)
    
    # 3. 应用删除
    obliterate(self._config, self._deletions)
```

#### merge_dicts 深度合并

**关键代码位置**：`invoke/config.py:1168-1224`

```python
def merge_dicts(
    base: Dict[str, Any], updates: Dict[str, Any]
) -> Dict[str, Any]:
    for key, value in (updates or {}).items():
        if key in base:
            if isinstance(value, dict):
                if isinstance(base[key], dict):
                    # 都是 dict：递归合并
                    merge_dicts(base[key], value)
                else:
                    # 类型冲突：dict vs non-dict → 抛出异常
                    raise AmbiguousMergeError(...)
            else:
                if isinstance(base[key], dict):
                    # 类型冲突：non-dict vs dict → 抛出异常
                    raise AmbiguousMergeError(...)
                elif hasattr(value, "fileno"):
                    # 文件对象：直接引用（不拷贝）
                    base[key] = value
                else:
                    # 普通值：浅拷贝
                    base[key] = copy.copy(value)
        else:
            # 新 key：添加
            if isinstance(value, dict):
                base[key] = copy_dict(value)  # 递归拷贝
            elif hasattr(value, "fileno"):
                base[key] = value
            else:
                base[key] = copy.copy(value)
    return base
```

**合并规则总结**：

| 场景 | 行为 |
|------|------|
| 两边都是 dict | 递归合并 |
| 一边 dict，一边非 dict | 抛出 `AmbiguousMergeError` |
| 两边都是非 dict | 用 updates 的值覆盖 base（浅拷贝）|
| 值是文件对象（有 fileno）| 直接引用（不拷贝）|
| 新 key（base 中不存在）| 添加到 base |

#### 运行时修改追踪

**关键代码位置**：`invoke/config.py:1102-1130`

当用户修改配置时：

```python
# 用户代码
c.run.echo = True
```

这会触发：
1. `DataProxy.__setattr__` → `self["run"] = ...`
2. `DataProxy.__setitem__` → `self._config[key] = value`
3. `_track_modification_of` → 通知根节点
4. `Config._modify` → 记录到 `_modifications` 层级

```python
def _modify(self, keypath: Tuple[str, ...], key: str, value: str) -> None:
    # 1. 从 deletions 中移除（如果之前被删除过）
    excise(self._deletions, keypath + (key,))
    
    # 2. 构建嵌套结构，记录到 _modifications
    data = self._modifications
    keypath_list = list(keypath)
    while keypath_list:
        subkey = keypath_list.pop(0)
        if subkey not in data:
            data[subkey] = {}
        data = data[subkey]
    data[key] = value
    
    # 3. 重新合并
    self.merge()
```

### 4.4 克隆机制

**关键代码位置**：`invoke/config.py:985-1071`

`Config.clone()` 创建一个独立的配置副本：

```python
def clone(self, into: Optional[Type["Config"]] = None) -> "Config":
    # 1. 创建新实例（lazy=True，不自动加载文件）
    klass = self.__class__ if into is None else into
    new = klass(**self._clone_init_kwargs(into=into))
    
    # 2. 复制各层级数据
    for name in """
        collection
        system_prefix
        system_path
        system_found
        system
        user_prefix
        user_path
        user_found
        user
        project_prefix
        project_path
        project_found
        project
        env_prefix
        env
        runtime_path
        runtime_found
        runtime
        overrides
        modifications
    """.split():
        name = "_{}".format(name)
        my_data = getattr(self, name)
        if not isinstance(my_data, dict):
            # 非 dict：浅拷贝
            new._set(name, copy.copy(my_data))
        else:
            # dict：merge_dicts（递归浅拷贝）
            merge_dicts(getattr(new, name), my_data)
    
    # 3. 加载基础配置文件
    new.load_base_conf_files()
    
    # 4. 合并
    new.merge()
    
    return new
```

#### 克隆 vs 直接赋值

```python
# 直接赋值：共享所有状态
config2 = config1
config2.run.echo = True  # 同时修改 config1

# clone()：独立副本
config2 = config1.clone()
config2.run.echo = True  # 不影响 config1
```

#### 拷贝策略

`clone()` 使用的是**浅拷贝**策略：

```python
# merge_dicts 内部
base[key] = copy.copy(value)  # 不是 copy.deepcopy
```

**原因**（文档注释）：
> "as this can cause issues with various objects such as compiled regexen or threading locks, often found buried deep within rich aggregates like API or DB clients"

**含义**：
- 简单值（字符串、数字、列表）会被拷贝
- 复杂对象（锁、编译的正则、API 客户端）保持引用
- 这是设计选择，避免 deepcopy 带来的问题

### 4.5 父子任务间的 Config 传播

#### Executor 中的传播

**关键代码位置**：`invoke/executor.py:129-136`

```python
for call in calls:
    # 同一个 config 引用
    config = self.config
    
    # 但每次重新加载 collection 和 shell env
    collection_config = self.collection.configuration(call.called_as)
    config.load_collection(collection_config)
    config.load_shell_env()
    
    # 创建新的 Context，但共享 config
    context = call.make_context(config, core_parse_result=self.core)
```

#### Context 中的共享

**关键代码位置**：`invoke/context.py:63-80`

```python
class Context(DataProxy):
    def __init__(self, config: Optional[Config] = None, remainder: str = ""):
        config = config if config is not None else Config()
        self._set(
            _config=config,  # 直接引用，不是拷贝
            command_prefixes=[],
            command_cwds=[],
            remainder=remainder,
        )
```

**测试验证**（`tests/executor.py:337-406`）：

```python
def context_is_new_but_config_is_same(self):
    ret = Executor(collection=coll).execute("task1", "task2")
    c1 = ret[task1]
    c2 = ret[task2]
    
    assert c1 is not c2        # Context 实例不同
    assert c1.config is c2.config  # 但共享同一个 Config
```

#### 状态持久化示例

```python
@task
def task1(c):
    c.foo = "bar"  # 修改 config

@task
def task2(c):
    print(c.foo)  # 能看到 "bar"

# 执行顺序：task1 -> task2
# task2 能看到 task1 的修改
```

#### 线程安全风险

**当前状态**：
- 多个 Context 共享同一个 Config
- Config 没有任何锁机制
- DataProxy 的修改追踪也不是线程安全的

**并行场景风险**：

```python
# 假设在并行执行场景（非当前实现）
@task
def task_a(c):
    c.config.some_key = "value_a"  # 竞态条件！

@task
def task_b(c):
    c.config.some_key = "value_b"  # 可能覆盖或被覆盖
```

**这也是为什么 R1 报告中提到**：
- 当前 `Executor` 是纯串行的
- 如果要实现并行，需要解决 Config 的线程安全问题

### 4.6 Config 与 DataProxy 关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DataProxy (基类)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  核心能力：                                                                    │
│  ├── dict 语法: config["key"], config["nested"]["key"]                      │
│  ├── attr 语法: config.key, config.nested.key                                │
│  ├── 嵌套 dict 自动包装为 DataProxy                                           │
│  └── 根-叶修改追踪（_track_modification_of）                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  关键方法：                                                                    │
│  ├── __getattr__/__setattr__: 属性访问代理                                    │
│  ├── _get: 嵌套包装逻辑                                                        │
│  ├── _set: 绕过代理的真实属性设置                                              │
│  └── _track_modification_of/_track_removal_of: 修改追踪                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │ 继承
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Config (配置核心)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  核心能力：                                                                    │
│  ├── 多层级配置管理 (9 个层级)                                                │
│  ├── 按优先级合并 (merge_dicts)                                               │
│  ├── 运行时修改追踪 (_modifications)                                          │
│  ├── 删除追踪 (_deletions)                                                    │
│  └── 克隆机制 (clone)                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  配置层级（优先级从低到高）：                                                  │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌─────────┐ ┌─────────┐ │
│  │ defaults │ │collection│ │ system │ │ user  │ │ project │ │   env   │ │
│  └──────────┘ └──────────┘ └────────┘ └────────┘ └─────────┘ └─────────┘ │
│                                                                               │
│  ┌─────────┐ ┌──────────┐ ┌───────────────┐                                │
│  │ runtime │ │ overrides│ │ modifications │ ◄── 最高优先级                  │
│  └─────────┘ └──────────┘ └───────────────┘                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  关键方法：                                                                    │
│  ├── merge(): 按优先级合并所有层级                                            │
│  ├── load_collection(): 加载任务集合配置                                      │
│  ├── load_shell_env(): 加载环境变量                                           │
│  ├── clone(): 创建独立副本                                                     │
│  └── _modify(): 记录运行时修改                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 总结与架构洞察

### 5.1 Runner 体系设计亮点

1. **清晰的职责分离**：
   - `run()`: 入口包装 + 资源清理
   - `_run_body()`: 核心执行流程
   - `_finish()`: 结果收集 + 异常处理
   - IO 线程：各自独立的职责

2. **灵活的执行模式**：
   - 同步模式：直接返回 `Result`
   - 异步模式：返回 `Promise`，支持 `join()` 和上下文管理器
   - disowned 模式：完全分离，不等待

3. **健壮的异常处理**：
   - `ExceptionHandlingThread` 捕获子线程异常
   - 区分不同异常类型（`WatcherError` vs 普通异常）
   - `ThreadException` 聚合多个线程异常

### 5.2 Config 体系设计亮点

1. **优雅的访问语法**：
   - `DataProxy` 让嵌套字典像对象一样访问
   - 同时保留 dict 协议的完整功能

2. **强大的层级管理**：
   - 9 个配置层级，优先级清晰
   - `merge_dicts` 递归合并，类型安全
   - 运行时修改自动追踪到最高优先级

3. **实用的克隆机制**：
   - `clone()` 创建独立副本
   - 浅拷贝策略避免复杂对象的问题
   - 适用于需要隔离配置的场景

### 5.3 与 Executor 体系的关联

回顾 R1 报告的发现，与本次分析呼应：

| 问题 | 根本原因 |
|------|---------|
| `ExceptionHandlingThread` 未被 Executor 使用 | Executor 负责任务调度，Runner 负责命令执行；`ExceptionHandlingThread` 是 IO 级别的工具，在 Runner 内部使用 |
| Context 间共享 Config | 设计如此！通过共享实现任务间状态传递；如果需要并行，这是需要解决的点 |
| 无循环依赖检测 | Executor 采用线性展开，不是基于依赖图的调度 |
| 无并行执行 | 核心设计是串行的；Config 共享也是串行友好的设计 |

### 5.4 未来改进方向

如果要增强 invoke 的能力，可能的改进点：

1. **并行执行**：
   - 需要引入依赖图（DAG）
   - 需要解决 Config 的线程安全（锁或拷贝）
   - 需要考虑异常时的回滚策略

2. **循环依赖检测**：
   - 在 `expand_calls()` 中添加访问栈
   - 检测递归调用时的重复节点

3. **显式隔离选项**：
   - 提供 `clone_context=True` 选项
   - 让用户选择是否隔离任务间的 Config

---

## 6. 代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| Runner.run() | `invoke/runners.py` | 126-406 |
| Runner._run_body() | `invoke/runners.py` | 439-469 |
| Runner._finish() | `invoke/runners.py` | 479-537 |
| Runner.create_io_threads() | `invoke/runners.py` | 643-683 |
| Local.start() | `invoke/runners.py` | 1335-1371 |
| Promise 类 | `invoke/runners.py` | 1618-1687 |
| ExceptionHandlingThread | `invoke/util.py` | 146-257 |
| DataProxy 类 | `invoke/config.py` | 36-312 |
| Config 类 | `invoke/config.py` | 313-1161 |
| Config.merge() | `invoke/config.py` | 941-964 |
| Config.clone() | `invoke/config.py` | 985-1071 |
| merge_dicts() | `invoke/config.py` | 1168-1224 |
| Context 类 | `invoke/context.py` | 22-411 |

---

**分析日期**：2026-04-27
**分析版本**：invoke-7066
