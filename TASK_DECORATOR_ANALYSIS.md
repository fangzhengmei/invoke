# Invoke @task 装饰器与参数映射分析报告

## 1. 概述

本文档深入分析 invoke 中 `@task` 装饰器如何通过函数签名自动生成 CLI 参数，包括：
- `@task` 装饰器的两种调用方式
- `inspect` 如何提取函数参数信息
- 函数参数如何映射为 CLI `Argument` 对象
- 不同类型参数的特殊处理（布尔、列表、递增、可选值）

---

## 2. @task 装饰器核心实现

### 2.1 两种调用方式

**关键代码位置**：`invoke/tasks.py:289-360`

```python
def task(*args: Any, **kwargs: Any) -> Callable:
    klass: Type[Task] = kwargs.pop("klass", Task)
    
    # 方式 1: @task (无括号形式)
    # 单个可调用对象，且不是 Task 实例
    if len(args) == 1 and callable(args[0]) and not isinstance(args[0], Task):
        return klass(args[0], **kwargs)
    
    # 方式 2: @task(...) (带参数形式)
    # 如果有位置参数，视为 pre 依赖的快捷方式
    if args:
        if "pre" in kwargs:
            raise TypeError(
                "May not give *args and 'pre' kwarg simultaneously!"
            )
        kwargs["pre"] = args
    
    # 内层装饰器
    def inner(body: Callable) -> Task[T]:
        _task = klass(body, **kwargs)
        return _task
    
    return inner
```

#### 调用方式对比：

| 方式 | 示例 | 说明 |
|------|------|------|
| 无括号 | `@task` | 单个可调用对象，直接返回 Task |
| 带括号 | `@task(...)` | 返回 inner 装饰器，支持参数配置 |
| 位置参数快捷方式 | `@task(pre_task1, pre_task2)` | 位置参数作为 pre 依赖 |

### 2.2 支持的装饰器参数

**关键代码位置**：`invoke/tasks.py:60-101` (Task.__init__

```python
def __init__(
    self,
    body: Callable,
    name: Optional[str] = None,
    aliases: Iterable[str] = (),
    positional: Optional[Iterable[str]] = None,
    optional: Iterable[str] = (),
    default: bool = False,
    auto_shortflags: bool = True,
    help: Optional[Dict[str, Any]] = None,
    pre: Optional[Union[List[str], str]] = None,
    post: Optional[Union[List[str], str]] = None,
    autoprint: bool = False,
    iterable: Optional[Iterable[str]] = None,
    incrementable: Optional[Iterable[str]] = None,
) -> None:
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | `str` | `None` | 覆盖任务名称 |
| `aliases` | `Iterable[str]` | `()` | 任务别名 |
| `positional` | `Optional[Iterable[str]]` | `None` | 显式指定位置参数 |
| `optional` | `Iterable[str]` | `()` | 指定可选值参数 |
| `default` | `bool` | `False` | 是否默认任务 |
| `auto_shortflags` | `bool` | `True` | 自动生成短标志 |
| `help` | `Optional[Dict[str, Any]]` | `None` | 参数帮助文本映射 |
| `pre` | `Optional[Union[List[str], str]]` | `None` | 前置依赖任务 |
| `post` | `Optional[Union[List[str], str]]` | `None` | 后置依赖任务 |
| `autoprint` | `bool` | `False` | 自动打印返回值 |
| `iterable` | `Optional[Iterable[str]]` | `None` | 列表类型参数 |
| `incrementable` | `Optional[Iterable[str]]` | `None` | 递增类型参数 |

### 2.3 实际使用示例

```python
from invoke import task

# 方式 1: 最简单的用法
@task
def mytask(c):
    """A simple task
    """
    pass

# 方式 2: 带配置
@task(
    name="build",
    aliases=["b"],
    default=True,
    help={
        "clean": "Clean before building",
        "verbose": "Verbose output"
    }
)
def build(c, clean=False, verbose=0):
    pass

# 方式 3: 前置依赖快捷方式
@task
def setup(c):
    pass

@task(setup)  # 等价于 @task(pre=[setup])
def deploy(c):
    pass
```

---

## 3. 函数参数提取机制

### 3.1 argspec() - 提取函数签名

**关键代码位置**：`invoke/tasks.py:147-175`

```python
def argspec(self, body: Callable) -> "Signature":
    """
    Returns a modified `inspect.Signature` based on that of ``body``.
    
    :returns:
        an `inspect.Signature` matching that of ``body``, but with the
        initial context argument removed.
    :raises TypeError:
        if the task lacks an initial positional `.Context` argument.
    """
    # 处理可调用但不是函数的对象
    if isinstance(body, types.FunctionType):
        func = body
    else:
        func = body.__call__
    
    # 获取函数签名
    sig = inspect.signature(func)
    params = list(sig.parameters.values())
    
    # 验证必须有初始 Context 参数
    if not len(params):
        raise TypeError("Tasks must have an initial Context argument!")
    
    # 返回去掉第一个参数（Context）的签名
    return sig.replace(parameters=params[1:])
```

### 3.2 签名转换示例

**原始函数**：
```python
@task
def mytask(c, arg1, arg2=False, arg3=5):
    pass
```

**原始签名**（`inspect.signature(mytask.body)`：
```
(c, arg1, arg2=False, arg3=5)
```

**转换后签名**（`argspec()` 返回）：
```
(arg1, arg2=False, arg3=5)
```

**参数信息**：
| 参数名 | 默认值 | 类型推断 |
|--------|--------|---------|
| `arg1` | `inspect.Signature.empty` | 无默认值 → 位置参数 |
| `arg2` | `False` | `bool` 类型 |
| `arg3` | `5` | `int` 类型 |

### 3.3 fill_implicit_positionals() - 隐式位置参数

**关键代码位置**：`invoke/tasks.py:177-188`

```python
def fill_implicit_positionals(
    self, positional: Optional[Iterable[str]]
) -> Iterable[str]:
    """
    If positionals is None, everything lacking a default
    value will be automatically considered positional.
    """
    if positional is None:
        positional = [
            x.name
            for x in self.argspec(self.body).parameters.values()
            if x.default is inspect.Signature.empty
        ]
    return positional
```

**规则**：
- 如果用户没有显式指定 `positional` 参数
- 则**所有没有默认值**的参数自动成为位置参数
- 有默认值的参数只能通过 `--flag=value` 形式传递

**示例**：
```python
@task
def mytask(c, required, optional=False):
    pass

# 隐式位置参数: ["required"]
# 调用方式: invoke mytask myvalue
#          invoke mytask required_value --optional
```

---

## 4. 参数映射为 Argument 对象

### 4.1 get_arguments() - 生成 Argument 列表

**关键代码位置**：`invoke/tasks.py:239-286`

```python
def get_arguments(
    self, ignore_unknown_help: Optional[bool] = None
) -> List[Argument]:
    """
    Return a list of Argument objects representing this task's signature.
    """
    # 1. 获取函数签名（已去掉 Context 参数）
    sig = self.argspec(self.body)
    
    # 2. 初始化已占用名称集合（用于短标志冲突检测
    taken_names = set(sig.parameters.keys())
    
    # 3. 遍历每个参数，创建 Argument 对象
    args = []
    for param in sig.parameters.values():
        new_arg = Argument(
            **self.arg_opts(param.name, param.default, taken_names)
        )
        args.append(new_arg)
        # 更新已占用名称
        taken_names.update(set(new_arg.names))
    
    # 4. 检查未知的 help 条目
    if self.help and not ignore_unknown_help:
        raise ValueError(
            "Help field was set for param(s) that don't exist: {}".format(
                list(self.help.keys())
            )
        )
    
    # 5. 位置参数前移
    for posarg in reversed(list(self.positional)):
        for i, arg in enumerate(args):
            if arg.name == posarg:
                args.insert(0, args.pop(i))
                break
    
    return args
```

### 4.2 arg_opts() - 生成单个参数选项

**关键代码位置**：`invoke/tasks.py:190-237`

```python
def arg_opts(
    self, name: str, default: str, taken_names: Set[str]
) -> Dict[str, Any]:
    opts: Dict[str, Any] = {}
    
    # 1. 是否位置参数
    opts["positional"] = name in self.positional
    
    # 2. 是否可选值参数
    opts["optional"] = name in self.optional
    
    # 3. 是否列表类型
    if name in self.iterable:
        opts["kind"] = list
        # 默认值: 用户提供的非 None 默认值，否则 []
        opts["default"] = default if default is not None else []
    
    # 4. 是否递增类型
    if name in self.incrementable:
        opts["incrementable"] = True
    
    # 5. 处理下划线转连字符
    original_name = name
    if "_" in name:
        opts["attr_name"] = name  # Python 属性名（带下划线）
        name = translate_underscores(name)  # CLI 名（带连字符）
    
    # 6. 生成标志名称列表
    names = [name]
    if self.auto_shortflags:
        # 自动生成短标志
        for char in name:
            if not (char == name or char in taken_names):
                names.append(char)
                break
    opts["names"] = names
    
    # 7. 推断类型和默认值
    if default not in (None, inspect.Signature.empty):
        kind = type(default)
        # 特殊情况: optional + bool 类型 → 不设置 kind，保持为 str
        if not (opts["optional"] and kind is bool):
            opts["kind"] = kind
        opts["default"] = default
    
    # 8. 处理 help 文本
    for possibility in name, original_name:
        if possibility in self.help:
            opts["help"] = self.help.pop(possibility)
            break
    
    return opts
```

### 4.3 完整映射示例

**任务定义**：
```python
@task(
    positional=["arg_3", "arg1"],
    optional=["arg1"],
    iterable=["mylist"],
    incrementable=["verbose"],
    help={
        "arg1": "First argument",
        "arg_3": "Third argument",
        "mylist": "List of items",
        "verbose": "Verbosity level"
    }
)
def mytask(c, arg1, arg2=False, arg_3=5, mylist=None, verbose=0):
    """A test task
    """
    pass
```

**参数映射过程**：

| Python 参数 | 默认值 | 处理 | 生成的 Argument |
|------------|--------|------|-----------------|
| `arg1` | `empty` | 位置参数，可选值 | `names=("arg1", "a")`, `positional=True`, `optional=True` |
| `arg2` | `False` | `bool` 类型 | `names=("arg2", "r")`, `kind=bool`, `default=False` |
| `arg_3` | `5` | 下划线转连字符，位置参数 | `names=("arg-3", "g")`, `attr_name="arg_3"`, `kind=int`, `default=5` |
| `mylist` | `None` | `iterable` → list` 类型 | `names=("mylist", "m")`, `kind=list`, `default=[]` |
| `verbose` | `0` | `incrementable` → 递增 | `names=("verbose", "v")`, `kind=int`, `default=0`, `incrementable=True` |

---

## 5. 不同类型参数的特殊处理

### 5.1 布尔类型参数

**关键代码位置**：`invoke/parser/argument.py:117-122` (takes_value)

```python
@property
def takes_value(self) -> bool:
    if self.kind is bool:
        return False  # 布尔类型不接受值
    if self.incrementable:
        return False  # 递增类型也不接受值
    return True
```

**解析行为**（`invoke/parser/argument.py:133-163`）：

```python
def set_value(self, value: Any, cast: bool = True) -> None:
    self.raw_value = value
    func = lambda x: x
    
    if cast:
        func = self.kind
    
    # list 类型: 追加而不是覆盖
    if self.kind is list:
        func = lambda x: self.value + [x]
    
    # incrementable 类型: 递增
    if self.incrementable:
        func = lambda x: self.value + 1
    
    self._value = func(value)
```

**布尔参数使用示例**：

```python
@task
def build(c, clean=False, verbose=False):
    pass
```

**CLI 使用方式**：
```bash
# 作为开关使用
invoke build --clean           # clean=True
invoke build --clean --verbose  # clean=True, verbose=True

# 不能这样是错误的（布尔类型不接受值
invoke build --clean=True     # ❌ 错误！
```

**注意**：布尔参数的特殊限制（`invoke/tasks.py:224-231`）：

```python
# NOTE: skip setting 'kind' if optional is True + type(default) is bool;
# that results in a nonsensical Argument which gives the
# parser grief in a few ways.
kind = type(default)
if not (opts["optional"] and kind is bool):
    opts["kind"] = kind
opts["default"] = default
```

**解释**：
- 如果一个参数同时是 `optional` 和 `bool` 类型
- **不设置 `kind=bool`**，保持默认的 `str`
- 因为 `optional=True` 意味着可以用作开关，也可以带值
- 而 `kind=bool` 意味着不能带值，两者矛盾

### 5.2 iterable (列表) 类型参数

**关键代码位置**：`invoke/parser/argument.py:67-68, 156-157`

```python
# __init__ 中
if kind is list:
    initial_value = []  # 初始值是空列表

# set_value 中
if self.kind is list:
    func = lambda x: self.value + [x]  # 追加而不是覆盖
```

**使用示例**：

```python
@task(iterable=["files"])
def process(c, files=None):
    print("Processing:", files)
```

**CLI 使用方式**：
```bash
# 多次使用 --files 会追加
invoke process --files=file1.txt --files=file2.txt
# 结果: files = ["file1.txt", "file2.txt"]
```

**默认值处理（`invoke/tasks.py:199-204`）：

```python
if name in self.iterable:
    opts["kind"] = list
    # 如果用户给了非 None 默认值，用用户的；否则用 []
    opts["default"] = default if default is not None else []
```

### 5.3 incrementable (递增) 类型参数

**关键代码位置**：`invoke/parser/argument.py:70-71, 159-162`

```python
# __init__ 中
if incrementable:
    initial_value = default  # 从默认值开始

# set_value 中
if self.incrementable:
    func = lambda x: self.value + 1  # 每次出现递增 1
```

**使用示例**：

```python
@task(incrementable=["verbose"])
def mytask(c, verbose=0):
    print("Verbosity level:", verbose)
```

**CLI 使用方式**：
```bash
invoke mytask                    # verbose = 0
invoke mytask -v                 # verbose = 1
invoke mytask -vvv               # verbose = 3
invoke mytask --verbose --verbose # verbose = 2
```

**注意**：
- `incrementable` 类型的 `takes_value` 返回 `False`
- 即不接受值，每次出现只是递增计数
- 必须配合 `default=0` 使用才有意义

### 5.4 optional (可选值) 类型参数

**关键代码位置**：`invoke/tasks.py:197`

```python
opts["optional"] = name in self.optional
```

**`Argument.optional 的含义**：
- 这个参数可以**两种方式使用**：
  1. 作为开关（不带值）→ 得到 `True`
  2. 带值使用 → 得到指定值

**使用示例**：

```python
@task(optional=["output"])
def process(c, output="default.txt"):
    pass
```

**CLI 使用方式**：
```bash
# 作为开关 → output=True
invoke process --output

# 带值 → output="custom.txt"
invoke process --output=custom.txt
```

**与布尔类型的特殊限制**：
如前所述，`optional=True` + `bool` 类型的参数：
- **不设置 `kind=bool`**
- 保持默认的 `str` 类型
- 因为 `optional=True` 已经隐含 "可以不带值"
- 而 `kind=bool` 意味着 "不能带值"

### 5.5 类型处理总结

| 参数类型 | 装饰器参数 | `kind` | `takes_value` | 行为 | CLI 示例 |
|---------|-----------|--------|---------------|------|----------|
| 字符串 | 默认 | `str` | `True` | 存储值 | `--name=value` |
| 整数 | 默认 | `int` | `True` | 转换为整数 | `--count=5` |
| 布尔 | 默认 | `bool` | `False` | 开关 | `--flag` |
| 列表 | `iterable=["x"]` | `list` | `True` | 追加 | `--x=a --x=b` → `["a", "b"]` |
| 递增 | `incrementable=["x"]` | `int` | `False` | 递增计数 | `-vvv` → `3` |
| 可选值 | `optional=["x"]` | 视默认 | 视默认 | 开关或带值 | `--x` 或 `--x=val` |

---

## 6. 完整执行链路

### 6.1 从函数到 CLI 参数的完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 用户定义任务函数                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  @task(                                                                      │
│      positional=["name"],                                                    │
│      iterable=["files"],                                                      │
│      help={"name": "Your name"}                                                  │
│  )                                                                            │
│  def greet(c, name, files=None, verbose=False):                           │
│      pass                                                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. @task 装饰器创建 Task 对象                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Task(                                                                       │
│      body=greet,                                                             │
│      positional=["name"],                                                        │
│      iterable=["files"],                                                      │
│      help={"name": "Your name"}                                               │
│      # 其他默认属性...                                                          │
│  )                                                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 注册到 Collection（CLI 启动时                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Collection.from_module() → collection.py                                         │
│      │                                                                        │
│      └── collection.add_task(greet_task)                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. Parser 准备阶段（Program.parse_tasks）                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  collection.to_contexts()                                                    │
│      │                                                                       │
│      └── 对每个任务:                                                         │
│          ├── task.get_arguments()                                               │
│          │   ├── argspec(body) → 去掉 Context 参数                           │
│          │   └── arg_opts() → 生成 Argument 选项                               │
│          │       ├── positional: True/False                                       │
│          │       ├── kind: str/int/bool/list                                   │
│          │       ├── default: ...                                             │
│          │       ├── help: ...                                                  │
│          │       └── names: ["greet", "g"]                                      │
│          │                                                                     │
│          └── 创建 ParserContext:                                                │
│              ├── name: "greet"                                                    │
│              ├── aliases: [...]                                                    │
│              └── args: [Argument(...), ...]                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 解析命令行参数                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Parser.parse_argv(["greet", "John", "--files=a.txt", "--files=b.txt", "-v"]│
│                                                                              │
│  解析结果:                                                                   │
│  ├── name (位置参数): "John"                                                │
│  ├── files (iterable): ["a.txt", "b.txt"]                                   │
│  └── verbose (bool): True                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. 执行任务                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Executor.execute()                                                          │
│      │                                                                       │
│      ├── normalize() → ParserContext → Call                                  │
│      │   ├── name = parser_ctx.name                                            │
│      │   └── kwargs = parser_ctx.as_kwargs                                  │
│      │       → {"name": "John", "files": ["a.txt", "b.txt"], "verbose": True}│
│      │                                                                       │
│      └── call.task(context, **call.kwargs)                                   │
│          │                                                                       │
│          └── greet(context, name="John", files=["a.txt", "b.txt"], verbose=True)│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| `@task` 装饰器 | `invoke/tasks.py` | 289-360 |
| `Task.__init__` | `invoke/tasks.py` | 60-101 |
| `Task.argspec()` | `invoke/tasks.py` | 147-175 |
| `Task.fill_implicit_positionals()` | `invoke/tasks.py` | 177-188 |
| `Task.arg_opts()` | `invoke/tasks.py` | 190-237 |
| `Task.get_arguments()` | `invoke/tasks.py` | 239-286 |
| `Argument.__init__` | `invoke/parser/argument.py` | 42-78 |
| `Argument.takes_value` | `invoke/parser/argument.py` | 117-122 |
| `Argument.set_value()` | `invoke/parser/argument.py` | 133-163 |
| `Collection.to_contexts()` | `invoke/collection.py` | 423-449 |
| `ParserContext.as_kwargs` | `invoke/parser/context.py` | 162-174 |

---

## 7. 常见使用场景示例

### 7.1 完整示例：带位置参数的任务

```python
from invoke import task

@task(
    help={
        "name": "Name of the person to greet",
        "lang": "Language code (en, fr, es)",
    }
)
def greet(c, name, lang="en"):
    """
    Greet someone in a specific language.
    
    Example:
        invoke greet John
        invoke greet John --lang=fr
    """
    greetings = {
        "en": "Hello",
        "fr": "Bonjour",
        "es": "Hola",
    }
    print(f"{greetings[lang]}, {name}!")
```

**CLI 生成的 CLI 参数**：
- `name`: 位置参数（无默认值）
- `--lang/-l`: 可选参数，默认值 `"en"`

**调用方式**：
```bash
invoke greet John              # name="John", lang="en"
invoke greet John --lang=fr   # name="John", lang="fr"
invoke greet John -l es      # 短标志
```

### 7.2 示例：列表类型参数

```python
@task(
    iterable=["paths"],
    help={
        "paths": "Paths to process (can be repeated)",
    }
)
def process(c, paths=None, verbose=False):
    """
    Process multiple files/directories.
    
    Example:
        invoke process --paths=/dir1 --paths=/dir2
    """
    print(f"Processing: {paths}")
    if verbose:
        print("Verbose mode enabled")
```

**调用方式**：
```bash
invoke process --paths=/a --paths=/b --paths=/c
# 结果: paths = ["/a", "/b", "/c"]
```

### 7.3 示例：递增类型参数

```python
@task(
    incrementable=["verbose"],
    help={
        "verbose": "Increase verbosity (can be repeated)",
    }
)
def deploy(c, verbose=0):
    """
    Deploy with configurable verbosity.
    
    Example:
        invoke deploy
        invoke deploy -v
        invoke deploy -vvv
    """
    if verbose >= 1:
        print("Starting deployment")
    if verbose >= 2:
        print("Loading config...")
    if verbose >= 3:
        print("Debug info...")
```

**调用方式**：
```bash
invoke deploy        # verbose = 0
invoke deploy -v   # verbose = 1
invoke deploy -vv  # verbose = 2
invoke deploy -vvv # verbose = 3
```

### 7.4 示例：可选值参数

```python
@task(
    optional=["output"],
    help={
        "output": "Output file (default: stdout)",
    }
)
def generate(c, output="stdout"):
    """
    Generate output, optionally to a file.
    
    Example:
        invoke generate           # output = "stdout"
        invoke generate --output # output = True
        invoke generate --output=result.txt  # output = "result.txt"
    """
    if output is True:
        print("Output to default file")
    elif output == "stdout":
        print("Output to stdout")
    else:
        print(f"Output to {output}")
```

**调用方式**：
```bash
invoke generate              # output = "stdout" (默认值)
invoke generate --output     # output = True (作为开关)
invoke generate --output=result.txt  # output = "result.txt"
```

---

## 8. 总结与关键洞察

### 8.1 核心设计原则

1. **基于函数签名的推断
   - 使用 `inspect.signature` 自动提取参数信息
   - 默认值决定参数类型 (`str`, `int`, `bool`)
   - 无默认值的参数自动成为位置参数

2. **灵活的类型注解**
   - `iterable`: 列表类型，支持多次追加
   - `incrementable`: 递增计数，支持 `-vvv` 风格
   - `optional`: 可选值，支持开关或带值

3. **自动短标志生成**
   - 自动从参数名第一个字符生成短标志
   - 检测冲突，避免重复

4. **下划线/连字符自动转换
   - Python 侧：`my_arg`
   - CLI 侧：`--my-arg`
   - 通过 `attr_name` 保持映射

### 8.2 类型推断规则

| Python 默认值 | 推断类型 | CLI 行为 |
|-----------|-----------|----------|
| 无默认值 | 无类型（str? | 位置参数，必须提供 |
| `"string" | `str` | 带值：`--arg=value` |
| `42` | `int` | 带值：`--count=5` |
| `True`/`False` | `bool` | 开关：`--flag`（不带值）|
| `None` + `iterable` | `list` | 多次追加：`--x=a --x=b` |
| `0` + `incrementable` | `int` | 递增：`-vvv` |

### 8.3 与前几份报告的关联

| 报告 | 关联点 |
|------|---------|
| R1 (Executor) | `Call.kwargs` 来自 `ParserContext.as_kwargs` |
| R3 (Collection) | `collection.to_contexts()` 调用 `task.get_arguments()` |
| R4 (CLI 生命周期) | `Program.parse_tasks()` 使用 `to_contexts()` 构建 Parser 解析 CLI 调用 `task.get_arguments()` |

---

**分析日期**：2026-04-27
**分析版本**：invoke-7066
