# Invoke 子进程执行机制（Runner）与流式 IO 协作设计（Watchers）分析报告

## 目录
1. [阻塞模式与伪终端（pty）模式两种子进程启动策略](#1-阻塞模式与伪终端pty模式两种子进程启动策略)
2. [流观察器（StreamWatcher）的正则匹配与标准输入回写机制](#2-流观察器streamwatcher的正则匹配与标准输入回写机制)
3. [应答型观察器（Responder）与失败型观察器（FailingResponder）的差异](#3-应答型观察器responder与失败型观察器failingresponder的差异)
4. [控制输出行为的参数分析](#4-控制输出行为的参数分析)
5. [pty 模式下的文件描述符复制与 SIGWINCH 信号处理](#5-pty-模式下的文件描述符复制与-sigwinch-信号处理)

---

## 1. 阻塞模式与伪终端（pty）模式两种子进程启动策略

### 1.1 核心实现差异

#### 阻塞模式（Blocking Mode）

阻塞模式使用标准的 `subprocess.Popen` 来启动子进程，在 `Local.start()` 方法中实现：

```python
# runners.py:1362-1371
else:
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

**关键特性：**
- **独立管道**：stdout、stderr、stdin 各自使用独立的管道
- **双输出线程**：`handle_stdout` 和 `handle_stderr` 两个独立线程分别处理标准输出和标准错误
- **可关闭 stdin**：可以通过 `close_proc_stdin()` 关闭子进程的标准输入
- **退出码获取**：通过 `self.process.returncode` 获取退出码

#### pty 模式（Pseudoterminal Mode）

pty 模式使用 `pty.fork()` 创建伪终端，实现代码如下：

```python
# runners.py:1336-1361
if self.using_pty:
    if pty is None:  # Encountered ImportError
        err = "You indicated pty=True, but your platform doesn't support the 'pty' module!"
        sys.exit(err)
    cols, rows = pty_size()
    self.pid, self.parent_fd = pty.fork()
    # If we're the child process, load up the actual command in a shell
    if self.pid == 0:
        # Set pty window size based on what our own controlling terminal's window size appears to be.
        winsize = struct.pack("HHHH", rows, cols, 0, 0)
        fcntl.ioctl(sys.stdout.fileno(), termios.TIOCSWINSZ, winsize)
        # Use execvpe for bare-minimum "exec w/ variable # args + env" behavior.
        os.execvpe(shell, [shell, "-c", command], env)
```

**关键特性：**
- **fork 机制**：使用 `os.fork()` 创建子进程，子进程通过 `os.execvpe()` 替换为目标命令
- **单一输出流**：pty 只有一个输出流，stdout 和 stderr 合并在一起
- **文件描述符**：父进程通过 `self.parent_fd` 读写 pty
- **窗口尺寸设置**：在子进程中设置 pty 的窗口尺寸（TIOCSWINSZ）

### 1.2 IO 线程创建差异

两种模式下创建的 IO 线程数量不同：

```python
# runners.py:643-683
def create_io_threads(self, ...):
    thread_args: Dict[Callable, Any] = {
        self.handle_stdout: {
            "buffer_": stdout,
            "hide": "stdout" in self.opts["hide"],
            "output": self.streams["out"],
        }
    }
    # stdin 线程...
    if self.streams["in"]:
        thread_args[self.handle_stdin] = {...}
    # 只有非 pty 模式才创建 stderr 线程
    if not self.using_pty:
        thread_args[self.handle_stderr] = {
            "buffer_": stderr,
            "hide": "stderr" in self.opts["hide"],
            "output": self.streams["err"],
        }
```

### 1.3 流读取实现差异

**pty 模式读取：**
```python
# runners.py:1269-1294
def read_proc_stdout(self, num_bytes: int) -> Optional[bytes]:
    if self.using_pty:
        try:
            data = os.read(self.parent_fd, num_bytes)
        except OSError as e:
            # 处理某些 Linux 平台上的虚假 OSError
            io_errors = ("Input/output error", "I/O error")
            if not any(error in stringified for error in io_errors):
                raise
            data = None
```

**阻塞模式读取：**
```python
elif self.process and self.process.stdout:
    data = os.read(self.process.stdout.fileno(), num_bytes)
```

### 1.4 标准输入写入与关闭

**写入差异：**
```python
# runners.py:1303-1321
def _write_proc_stdin(self, data: bytes) -> None:
    if self.using_pty:
        fd = self.parent_fd
    elif self.process and self.process.stdin:
        fd = self.process.stdin.fileno()
```

**关闭差异：**
```python
# runners.py:1323-1333
def close_proc_stdin(self) -> None:
    if self.using_pty:
        # pty 模式下无法关闭 stdin
        raise SubprocessPipeError("Cannot close stdin when pty=True")
    elif self.process and self.process.stdin:
        self.process.stdin.close()
```

### 1.5 进程完成检测与退出码获取

**进程完成检测：**
```python
# runners.py:1384-1397
@property
def process_is_finished(self) -> bool:
    if self.using_pty:
        # 使用 os.waitpid 非阻塞检查
        pid_val, self.status = os.waitpid(self.pid, os.WNOHANG)
        return pid_val != 0
    else:
        # 使用 Popen.poll()
        return self.process.poll() is not None
```

**退出码获取：**
```python
# runners.py:1399-1418
def returncode(self) -> Optional[int]:
    if self.using_pty:
        code = None
        if os.WIFEXITED(self.status):
            code = os.WEXITSTATUS(self.status)
        elif os.WIFSIGNALED(self.status):
            code = os.WTERMSIG(self.status)
            code = -1 * code  # 信号转为负数退出码
        return code
    else:
        return self.process.returncode
```

### 1.6 使用场景与优劣取舍

| 特性 | 阻塞模式 | pty 模式 |
|------|---------|---------|
| ** stdout/stderr 分离** | ✅ 完全分离 | ❌ 合并为单一输出 |
| **关闭 stdin** | ✅ 支持 | ❌ 不支持 |
| **程序行缓冲** | ❌ 可能完全缓冲 | ✅ 行缓冲（模拟终端） |
| **交互式程序** | ⚠️ 部分支持 | ✅ 原生支持（如 sudo、vi） |
| **Windows 支持** | ✅ 完整支持 | ❌ 需要 pty 模块 |
| **资源消耗** | 较低 | 较高（需要 pty 分配） |

**推荐场景：**
- **阻塞模式**：简单命令执行、需要独立捕获 stdout/stderr、Windows 环境
- **pty 模式**：交互式程序（sudo、passwd）、需要终端行为的程序、密码输入场景

---

## 2. 流观察器（StreamWatcher）的正则匹配与标准输入回写机制

### 2.1 核心架构

StreamWatcher 是一个基于 `threading.local` 的基类，用于在多线程环境中安全地观察子进程输出流：

```python
# watchers.py:8-36
class StreamWatcher(threading.local):
    """
    子类必须实现 submit() 方法，该方法：
    - 接收整个流的当前内容作为字符串
    - 可选返回可迭代的字符串（写入子进程 stdin）
    """
    
    def submit(self, stream: str) -> Iterable[str]:
        raise NotImplementedError
```

**设计要点：**
- 继承 `threading.local`，使得 stdout 和 stderr 两个观察线程拥有独立的实例状态
- `submit()` 方法接收**完整流内容**（而非增量），允许跨数据块的模式匹配

### 2.2 触发时机：respond() 方法

StreamWatcher 在输出处理循环中被调用：

```python
# runners.py:750-772
def _handle_output(self, buffer_, hide, output, reader):
    for data in self.read_proc_output(reader):
        # 1. 输出到终端（如果不隐藏）
        if not hide:
            self.write_our_output(stream=output, string=data)
        # 2. 存入缓冲区
        buffer_.append(data)
        # 3. 触发观察器
        self.respond(buffer_)
```

**respond() 方法实现：**
```python
# runners.py:932-957
def respond(self, buffer_: List[str]) -> None:
    # 将缓冲区内容合并为单个字符串
    stream = "".join(buffer_)
    for watcher in self.watchers:
        # 提交给每个观察器，获取响应
        for response in watcher.submit(stream):
            # 将响应写入子进程的 stdin
            self.write_proc_stdin(response)
```

### 2.3 Responder 的正则匹配机制

Responder 实现了基于索引的增量匹配策略，避免重复匹配：

```python
# watchers.py:62-110
class Responder(StreamWatcher):
    def __init__(self, pattern: str, response: str) -> None:
        self.pattern = pattern
        self.response = response
        self.index = 0  # 记录已处理的位置
    
    def pattern_matches(self, stream: str, pattern: str, index_attr: str):
        # 只查看尚未处理的流内容
        index = getattr(self, index_attr)
        new = stream[index:]
        # 使用 re.findall 进行跨行匹配（re.S）
        matches = re.findall(pattern, new, re.S)
        # 更新索引位置
        if matches:
            setattr(self, index_attr, index + len(new))
        return matches
    
    def submit(self, stream: str) -> Generator[str, None, None]:
        for _ in self.pattern_matches(stream, self.pattern, "index"):
            yield self.response
```

**匹配策略要点：**
1. **索引追踪**：`self.index` 记录上一次处理的位置
2. **增量扫描**：`stream[index:]` 只扫描新增内容
3. **跨行匹配**：`re.S` 标志使 `.` 匹配换行符
4. **多次响应**：`re.findall` 可能返回多个匹配，每个匹配触发一次响应

### 2.4 数据回写流程

完整的数据回写链路：

```
子进程 stdout/stderr
     ↓
read_proc_output() 读取字节
     ↓
decode() 解码为字符串
     ↓
buffer_.append(data) 存入缓冲区
     ↓
respond(buffer_) 调用
     ↓
watcher.submit(stream) 获取响应
     ↓
write_proc_stdin(response) 写入子进程 stdin
     ↓
_write_proc_stdin() 实际写入
     ↓
os.write(fd, data) 底层系统调用
```

---

## 3. 应答型观察器（Responder）与失败型观察器（FailingResponder）的差异

### 3.1 FailingResponder 的设计

FailingResponder 继承自 Responder，增加了失败检测能力：

```python
# watchers.py:113-145
class FailingResponder(Responder):
    def __init__(self, pattern: str, response: str, sentinel: str) -> None:
        super().__init__(pattern, response)
        self.sentinel = sentinel      # 失败标志模式
        self.failure_index = 0         # 失败检测的独立索引
        self.tried = False              # 是否已提交响应
    
    def submit(self, stream: str) -> Generator[str, None, None]:
        # 1. 先执行正常的 Responder 逻辑
        response = super().submit(stream)
        # 2. 同时检测失败标志
        failed = self.pattern_matches(stream, self.sentinel, "failure_index")
        # 3. 如果已提交过响应且检测到失败标志，抛出异常
        if self.tried and failed:
            err = 'Auto-response to r"{}" failed with {!r}!'.format(
                self.pattern, self.sentinel
            )
            raise ResponseNotAccepted(err)
        # 4. 记录是否提交过响应
        if response:
            self.tried = True
        return response
```

### 3.2 核心差异对比

| 特性 | Responder | FailingResponder |
|------|-----------|------------------|
| **索引数量** | 1 个（index） | 2 个（index + failure_index） |
| **模式数量** | 1 个（匹配模式） | 2 个（匹配模式 + 失败哨兵） |
| **状态追踪** | 无 | `tried` 标志 |
| **异常抛出** | 从不 | 检测到失败时抛出 `ResponseNotAccepted` |

### 3.3 触发时机差异

**Responder 的触发时机：**
- 每次 `respond()` 调用时扫描新增内容
- 匹配到 `pattern` 时立即 `yield response`
- 匹配后更新 `index`，避免重复响应

**FailingResponder 的触发时机：**

**成功路径：**
1. 检测到 `pattern` → yield response → 设置 `tried = True`
2. 后续输出中**不出现** `sentinel` → 继续执行

**失败路径：**
1. 检测到 `pattern` → yield response → 设置 `tried = True`
2. 后续输出中**出现** `sentinel` 且 `tried = True` → 抛出 `ResponseNotAccepted`

### 3.4 最大响应次数限制的处理

**当前实现的特点：**
- Responder 没有内置的最大响应次数限制
- 每次匹配都会触发响应，直到索引推进到流末尾

**状态管理机制：**
- 索引机制确保**同一内容不会被重复匹配**
- 例如：流内容为 `"Password: Password: Password:"`，pattern 为 `r"Password:"`
- 第一次 submit：匹配 3 次，yield 3 个 response，index 推进到流末尾
- 后续 submit：`new = stream[index:]` 为空，不再匹配

**自定义响应次数限制的方式：**
用户可以通过继承 Responder 来实现次数限制：

```python
class LimitedResponder(Responder):
    def __init__(self, pattern, response, max_times=1):
        super().__init__(pattern, response)
        self.max_times = max_times
        self.response_count = 0
    
    def submit(self, stream):
        for _ in self.pattern_matches(stream, self.pattern, "index"):
            if self.response_count < self.max_times:
                self.response_count += 1
                yield self.response
```

---

## 4. 控制输出行为的参数分析

### 4.1 参数总览

Runner.run() 方法提供了多个控制输出行为的参数：

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `echo` | bool | False | 是否在执行前打印命令字符串 |
| `echo_format` | str | ANSI 粗体 | 控制命令回显的格式 |
| `echo_stdin` | bool/None | None | 是否将 stdin 输入回显到 stdout |
| `hide` | bool/str | None | 隐藏 stdout/stderr 输出 |
| `warn` | bool | False | 非零退出码时警告而非抛出异常 |
| `out_stream` | IO | sys.stdout | 标准输出的目标流 |
| `err_stream` | IO | sys.stderr | 标准错误的目标流 |
| `in_stream` | IO/bool | sys.stdin | 标准输入的源 |

### 4.2 echo 参数：命令回显

**实现位置：**
```python
# runners.py:408-409
def echo(self, command: str) -> None:
    print(self.opts["echo_format"].format(command=command))
```

**调用时机：**
```python
# runners.py:423-425
if self.opts["echo"]:
    self.echo(command)
```

**特殊规则：**
1. `hide=True` 会覆盖 `echo=True`：
   ```python
   # runners.py:568-569
   if opts["hide"] is True:
       opts["echo"] = False
   ```
2. `dry=True` 会强制开启 echo：
   ```python
   # runners.py:571-572
   if opts["dry"] is True:
       opts["echo"] = True
   ```

### 4.3 hide 参数：输出隐藏

**normalize_hide 函数：**
```python
# runners.py:1689-1714
def normalize_hide(val, out_stream=None, err_stream=None):
    # 有效值验证
    hide_vals = (None, False, "out", "stdout", "err", "stderr", "both", True)
    # 转换为流名称列表
    if val in (None, False):
        hide = []
    elif val in ("both", True):
        hide = ["stdout", "stderr"]
    elif val == "out":
        hide = ["stdout"]
    elif val == "err":
        hide = ["stderr"]
    # 如果用户指定了自定义流，则不隐藏该流
    if out_stream is not None and "stdout" in hide:
        hide.remove("stdout")
    if err_stream is not None and "stderr" in hide:
        hide.remove("stderr")
    return tuple(hide)
```

**隐藏逻辑在输出处理中的应用：**
```python
# runners.py:750-772
def _handle_output(self, buffer_, hide, output, reader):
    for data in self.read_proc_output(reader):
        # hide 参数控制是否输出到终端
        if not hide:
            self.write_our_output(stream=output, string=data)
        # 无论是否隐藏，始终存入缓冲区（供 Result 使用）
        buffer_.append(data)
```

**关键点：**
- `hide` 只控制**是否输出到终端**
- 输出**始终被捕获**到 `Result.stdout` 和 `Result.stderr`

### 4.4 echo_stdin 参数：标准输入回显

**自动检测逻辑：**
```python
# runners.py:918-930
def should_echo_stdin(self, input_: IO, output: IO) -> bool:
    """
    决定是否回显 stdin 的条件：
    1. 不使用 pty（pty 会自动回显）
    2. 输入流是真实的 TTY
    """
    return (not self.using_pty) and isatty(input_)
```

**实际使用：**
```python
# runners.py:898-901
if echo is None:
    echo = self.should_echo_stdin(input_, output)
if echo:
    self.write_our_output(stream=output, string=data)
```

**设计原因：**
- pty 模式下，子进程的终端会自动回显输入
- 非 pty 模式下，需要手动回显才能让用户看到自己输入的内容

### 4.5 warn 参数：非零退出码处理

**warn 参数的影响范围：**
```python
# runners.py:535-536
if not (result or self.opts["warn"]):
    raise UnexpectedExit(result)
```

**Result 的布尔值：**
```python
# runners.py:1553-1554
def __bool__(self) -> bool:
    return self.ok

# runners.py:1580-1587
@property
def ok(self) -> bool:
    return bool(self.exited == 0)
```

**warn 不影响的异常：**
根据文档（runners.py:356-373），warn 只影响 `UnexpectedExit`，不影响：
- `ThreadException` - 后台线程异常
- `WatcherError` / `Failure` - 观察器错误
- `CommandTimedOut` - 超时

### 4.6 流重定向参数

**in_stream 参数：**
```python
# runners.py:585-590
in_stream = opts["in_stream"]
if in_stream is None:
    # 异步模式下自动禁用 stdin 读取
    in_stream = False if self._asynchronous else sys.stdin
```

**stdout/stderr 线程创建条件：**
```python
# runners.py:666-677
if self.streams["in"]:
    thread_args[self.handle_stdin] = {...}
if not self.using_pty:
    thread_args[self.handle_stderr] = {...}
```

---

## 5. pty 模式下的文件描述符复制与 SIGWINCH 信号处理

### 5.1 pty.fork() 的工作原理

**pty.fork() 返回值：**
```python
# runners.py:1341
self.pid, self.parent_fd = pty.fork()
```

| 返回值 | 含义 |
|--------|------|
| `pid == 0` | 子进程，需要执行目标命令 |
| `pid > 0` | 父进程，返回子进程 PID 和 pty 主设备文件描述符 |

### 5.2 子进程中的 pty 设置

**子进程执行流程：**
```python
# runners.py:1345-1361
if self.pid == 0:
    # 1. 获取当前终端尺寸
    cols, rows = pty_size()
    
    # 2. 设置 pty 窗口大小
    winsize = struct.pack("HHHH", rows, cols, 0, 0)
    fcntl.ioctl(sys.stdout.fileno(), termios.TIOCSWINSZ, winsize)
    
    # 3. 替换为目标命令
    os.execvpe(shell, [shell, "-c", command], env)
```

**winsize 结构体：**
```
struct winsize {
    unsigned short ws_row;    /* 行数 */
    unsigned short ws_col;    /* 列数 */
    unsigned short ws_xpixel; /* 水平像素（未使用） */
    unsigned short ws_ypixel; /* 垂直像素（未使用） */
};
```

### 5.3 pty_size() 的实现

**POSIX 平台实现：**
```python
# terminals.py:81-113
def _pty_size() -> Tuple[Optional[int], Optional[int]]:
    # TIOCGWINSZ 结构体格式：HHHH (rows, cols, xpixel, ypixel)
    fmt = "HHHH"
    buf = struct.pack(fmt, 0, 0, 0, 0)
    
    # 使用 ioctl 获取终端窗口大小
    try:
        result = fcntl.ioctl(sys.stdout, termios.TIOCGWINSZ, buf)
        rows, cols, *_ = struct.unpack(fmt, result)
        return (cols, rows)  # 注意：返回顺序是 (cols, rows)
    except (struct.error, TypeError, IOError, AttributeError):
        pass
    return (None, None)
```

**Windows 平台实现：**
```python
# terminals.py:50-77
def _pty_size() -> Tuple[Optional[int], Optional[int]]:
    class CONSOLE_SCREEN_BUFFER_INFO(Structure):
        _fields_ = [
            ("dwSize", _COORD),
            ("dwCursorPosition", _COORD),
            ("wAttributes", c_ushort),
            ("srWindow", _SMALL_RECT),
            ("dwMaximumWindowSize", _COORD),
        ]
    
    # 使用 Windows API 获取控制台信息
    GetStdHandle = windll.kernel32.GetStdHandle
    GetConsoleScreenBufferInfo = windll.kernel32.GetConsoleScreenBufferInfo
    
    hstd = GetStdHandle(-11)  # STD_OUTPUT_HANDLE
    csbi = CONSOLE_SCREEN_BUFFER_INFO()
    ret = GetConsoleScreenBufferInfo(hstd, byref(csbi))
    
    if ret:
        sizex = csbi.srWindow.Right - csbi.srWindow.Left + 1
        sizey = csbi.srWindow.Bottom - csbi.srWindow.Top + 1
        return sizex, sizey
```

### 5.4 SIGWINCH 信号处理现状

**当前实现的限制：**
通过代码分析，invoke 当前实现**没有**主动处理 SIGWINCH 信号的转发机制。

**现有代码中与信号相关的部分：**

1. **超时定时器实现：**
   ```python
   # runners.py:1087-1093
   def start_timer(self, timeout: int) -> None:
       if timeout is not None:
           self._timer = threading.Timer(timeout, self.kill)
           self._timer.start()
   ```

2. **键盘中断转发：**
   ```python
   # runners.py:490-491
   except KeyboardInterrupt as e:
       self.send_interrupt(e)
   
   # runners.py:1156-1172
   def send_interrupt(self, interrupt: "KeyboardInterrupt") -> None:
       self.write_proc_stdin("\x03")  # 发送 Ctrl+C
   ```

3. **强制终止：**
   ```python
   # runners.py:1376-1382
   def kill(self) -> None:
       try:
           os.kill(self.get_pid(), signal.SIGKILL)
       except ProcessLookupError:
           pass
   ```

### 5.5 窗口大小变更的潜在解决方案

**如果需要实现 SIGWINCH 转发，可能的实现方式：**

```python
# 伪代码：SIGWINCH 信号处理
class Local(Runner):
    def __init__(self, context):
        super().__init__(context)
        self._original_winch_handler = None
    
    def start(self, command, shell, env):
        if self.using_pty:
            # 安装 SIGWINCH 处理器
            self._original_winch_handler = signal.signal(
                signal.SIGWINCH, 
                self._handle_sigwinch
            )
            # ... 原有 pty.fork() 逻辑
    
    def _handle_sigwinch(self, signum, frame):
        """处理窗口大小变更信号"""
        if self.using_pty and self.parent_fd:
            # 获取新的窗口大小
            cols, rows = pty_size()
            # 更新 pty 的窗口大小
            winsize = struct.pack("HHHH", rows, cols, 0, 0)
            fcntl.ioctl(self.parent_fd, termios.TIOCSWINSZ, winsize)
    
    def stop(self):
        # 恢复原始信号处理器
        if self._original_winch_handler:
            signal.signal(signal.SIGWINCH, self._original_winch_handler)
        super().stop()
```

### 5.6 文件描述符生命周期

**pty 模式下的文件描述符管理：**

```
pty.fork()
    ↓
父进程获得 parent_fd
    ↓
读取：os.read(self.parent_fd, num_bytes)
写入：os.write(self.parent_fd, data)
    ↓
进程结束
    ↓
stop() 中关闭：os.close(self.parent_fd)
```

**关闭实现：**
```python
# runners.py:1420-1431
def stop(self) -> None:
    super().stop()
    # 确保关闭 pty 文件描述符，防止文件描述符泄漏
    if self.using_pty:
        try:
            os.close(self.parent_fd)
        except Exception:
            pass
```

### 5.7 设计权衡分析

**为什么不实现 SIGWINCH 转发：**

1. **复杂性**：信号处理需要考虑多线程安全、信号重入等问题
2. **使用场景**：invoke 主要用于脚本自动化，而非交互式终端会话
3. **替代方案**：初始设置窗口尺寸通常已足够满足大多数场景

**实际影响：**
- 运行时调整终端窗口大小，子进程中的全屏程序（如 vim、less）可能不会正确响应
- 对于大多数自动化任务，这不是问题

---

## 附录：核心类关系图

```
Runner (抽象基类)
├── 核心属性
│   ├── context: Context
│   ├── using_pty: bool
│   ├── watchers: List[StreamWatcher]
│   ├── program_finished: threading.Event
│   └── opts: Dict[str, Any]
│
├── 核心方法
│   ├── run(command, **kwargs)
│   ├── start(command, shell, env)  # 抽象
│   ├── wait()
│   ├── respond(buffer_)
│   └── create_io_threads()
│
└── 子类
    └── Local
        ├── 阻塞模式：Popen + PIPE
        └── pty 模式：pty.fork()

StreamWatcher (基类)
├── submit(stream: str) -> Iterable[str]  # 抽象
│
├── Responder
│   ├── pattern: str
│   ├── response: str
│   ├── index: int
│   └── pattern_matches()
│
└── FailingResponder
    ├── sentinel: str
    ├── failure_index: int
    ├── tried: bool
    └── 可能抛出 ResponseNotAccepted
```

---

## 关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Local.start() (pty 模式) | runners.py | 1335-1361 |
| Local.start() (阻塞模式) | runners.py | 1362-1371 |
| create_io_threads() | runners.py | 643-683 |
| respond() | runners.py | 932-957 |
| _handle_output() | runners.py | 750-772 |
| Responder 类 | watchers.py | 53-110 |
| FailingResponder 类 | watchers.py | 113-145 |
| normalize_hide() | runners.py | 1689-1714 |
| pty_size() | terminals.py | 116-128 |
| _pty_size() (POSIX) | terminals.py | 81-113 |
| 信号处理相关 | runners.py | 4, 1087-1093, 1376-1382 |
