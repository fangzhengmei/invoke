# Invoke Collection 任务名解析深度分析报告

## 1. 概述

本文档深入分析 invoke 库中 `Collection` 体系的任务组织和解析机制，包括：
- `Collection.from_module()` 扫描并注册 Task 对象的机制
- 多级 Collection（嵌套命名空间）通过 `__getitem__` / `__contains__` 解析任务名的流程
- `Executor.execute()` 在调用链中定位正确 Task 实例的完整路径

---

## 2. Collection 体系架构概览

### 2.1 核心数据结构

```
Collection (任务集合/命名空间)
  ├── .tasks: Lexicon      # 存储 Task 对象（支持别名）
  ├── .collections: Lexicon  # 存储子 Collection（嵌套命名空间）
  ├── .default: str | None   # 默认任务名
  ├── .name: str | None      # 集合名称（用于嵌套时的路径前缀）
  ├── ._configuration: dict  # 集合级别的配置
  └── .auto_dash_names: bool # 是否自动将下划线转换为连字符
```

### 2.2 Lexicon 简介

**关键代码位置**：`invoke/vendor/lexicon/__init__.py:6-24`

`Lexicon` 是 `Collection` 内部使用的核心数据结构，继承自 `AttributeDict` 和 `AliasDict`：

```python
class Lexicon(AttributeDict, AliasDict):
    def __init__(self, *args, **kwargs):
        dict.__init__(self, *args, **kwargs)
        dict.__setattr__(self, "aliases", {})
```

**继承关系**：

| 父类 | 职责 |
|------|------|
| `dict` | 基础字典功能 |
| `AttributeDict` | 支持属性语法访问：`lex.key` 等价于 `lex["key"]` |
| `AliasDict` | 支持别名映射：一个 key 可以有多个别名 |

---

## 3. Collection.from_module() 机制分析

### 3.1 核心实现

**关键代码位置**：`invoke/collection.py:145-236`

```python
@classmethod
def from_module(
    cls,
    module: ModuleType,
    name: Optional[str] = None,
    config: Optional[Dict[str, Any]] = None,
    loaded_from: Optional[str] = None,
    auto_dash_names: Optional[bool] = None,
) -> "Collection":
```

### 3.2 执行流程

#### 阶段 1: 确定模块名称

```python
module_name = module.__name__.split(".")[-1]
```

- 从 `module.__name__` 提取最后一个部分
- 例如：`package.module` → `"module"`

#### 阶段 2: 定义实例化辅助函数

```python
def instantiate(obj_name: Optional[str] = None) -> "Collection":
    # 名称优先级：显式 name > root ns name > module_name
    args = [name or obj_name or module_name]
    kwargs = dict(
        loaded_from=loaded_from, auto_dash_names=auto_dash_names
    )
    instance = cls(*args, **kwargs)
    instance.__doc__ = module.__doc__
    return instance
```

#### 阶段 3: 检查显式命名空间

```python
# 检查模块是否提供了默认的 NS（ns 或 namespace）
for candidate in ("ns", "namespace"):
    obj = getattr(module, candidate, None)
    if obj and isinstance(obj, Collection):
        # 使用显式定义的 Collection
        ret = instantiate(obj_name=obj.name)
        # 复制 tasks 和 collections（应用 transform）
        ret.tasks = ret._transform_lexicon(obj.tasks)
        ret.collections = ret._transform_lexicon(obj.collections)
        ret.default = (
            ret.transform(obj.default) if obj.default else None
        )
        # 合并配置
        obj_config = copy_dict(obj._configuration)
        if config:
            merge_dicts(obj_config, config)
        ret._configuration = obj_config
        return ret
```

**设计意图**：允许用户在模块中显式定义命名空间结构，例如：

```python
# tasks.py
from invoke import Collection, task

@task
def top_level(c):
    pass

@task
def sub_task(c):
    pass

# 显式命名空间
sub = Collection("sub_level")
sub.add_task(sub_task)

ns = Collection(top_level, sub)
ns.name = "explicit_root"
```

#### 阶段 4: 隐式扫描（无显式 NS 时）

```python
# 扫描模块中的所有 Task 实例
tasks = filter(lambda x: isinstance(x, Task), vars(module).values())

# 创建 Collection 并添加所有任务
collection = instantiate()
for task in tasks:
    collection.add_task(task)

# 应用配置
if config:
    collection.configure(config)

return collection
```

**扫描逻辑**：
- `vars(module)` 获取模块的所有成员字典
- `filter(lambda x: isinstance(x, Task), ...)` 过滤出 Task 实例
- 逐个调用 `add_task()` 注册

### 3.3 add_task() 注册机制

**关键代码位置**：`invoke/collection.py:238-283`

```python
def add_task(
    self,
    task: "Task",
    name: Optional[str] = None,
    aliases: Optional[Tuple[str, ...]] = None,
    default: Optional[bool] = None,
) -> None:
```

#### 步骤 1: 确定任务名称

```python
if name is None:
    if task.name:
        name = task.name
    elif hasattr(task.body, "func_name"):
        name = task.body.func_name  # Python 2 兼容
    elif hasattr(task.body, "__name__"):
        name = task.__name__
    else:
        raise ValueError("Could not obtain a name for this task!")
```

**名称优先级**：
1. 显式传入的 `name` 参数
2. `task.name` 属性
3. `task.body.func_name`（Python 2 兼容）
4. `task.__name__`（通常是函数的 `__name__`）

#### 步骤 2: 应用名称转换

```python
name = self.transform(name)
```

**关键代码位置**：`invoke/collection.py:456-493`

```python
def transform(self, name: str) -> str:
    if not name:
        return name
    from_, to = "_", "-"
    if not self.auto_dash_names:
        from_, to = "-", "_"
    replaced = []
    end = len(name) - 1
    for i, char in enumerate(name):
        # 不替换开头/结尾的下划线，也不替换点号旁边的下划线
        if (
            i not in (0, end)
            and char == from_
            and name[i - 1] != "."
            and name[i + 1] != "."
        ):
            char = to
        replaced.append(char)
    return "".join(replaced)
```

**转换规则**（`auto_dash_names=True` 时）：
- `my_task` → `my-task`
- `_my_task_` → `_my-task_`（保留首尾下划线）
- `my_inner.task_name` → `my-inner.task-name`（点号分隔的各部分独立转换）

#### 步骤 3: 检查命名冲突

```python
if name in self.collections:
    err = "Name conflict: this collection has a sub-collection named {!r} already"
    raise ValueError(err.format(name))
```

- 任务名不能与已有的子 Collection 同名

#### 步骤 4: 注册到 Lexicon

```python
self.tasks[name] = task
```

#### 步骤 5: 注册别名

```python
for alias in list(task.aliases) + list(aliases or []):
    self.tasks.alias(self.transform(alias), to=name)
```

**示例**：
```python
@task(aliases=["t1", "task-one"])
def my_task(c):
    pass

# 注册后：
# tasks["my-task"] = task
# tasks.aliases["t1"] = "my-task"
# tasks.aliases["task-one"] = "my-task"
```

#### 步骤 6: 设置默认任务

```python
if default is True or (default is None and task.is_default):
    self._check_default_collision(name)
    self.default = name
```

**默认任务优先级**：
1. `default=True` 参数
2. `task.is_default` 属性（通常来自 `@task(default=True)`）

### 3.4 扫描流程图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Collection.from_module(module)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 提取模块名称                                                              │
│     module_name = module.__name__.split(".")[-1]                           │
│                                                                              │
│                           │                                                  │
│                           ▼                                                  │
│                                                                              │
│  2. 检查显式命名空间                                                          │
│     for candidate in ("ns", "namespace"):                                   │
│         obj = getattr(module, candidate, None)                              │
│         if obj and isinstance(obj, Collection):                             │
│             → 使用显式 Collection，复制 tasks/collections/config           │
│             → return                                                         │
│                                                                              │
│                           │ (无显式 NS)                                      │
│                           ▼                                                  │
│                                                                              │
│  3. 隐式扫描 Task 实例                                                        │
│     tasks = filter(lambda x: isinstance(x, Task), vars(module).values())  │
│                                                                              │
│                           │                                                  │
│                           ▼                                                  │
│                                                                              │
│  4. 逐个 add_task() 注册                                                      │
│     for task in tasks:                                                       │
│         collection.add_task(task)                                            │
│                                                                              │
│                           │                                                  │
│                           ▼                                                  │
│                                                                              │
│  5. 返回 Collection                                                          │
│     return collection                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.5 实际示例分析

#### 示例 1: 简单模块（隐式扫描）

```python
# tasks.py
from invoke import task

@task
def build(c):
    print("Building...")

@task(aliases=["t"])
def test(c):
    print("Testing...")
```

**扫描结果**：
```python
collection = Collection.from_module(module)
# collection.tasks["build"] = build_task
# collection.tasks["test"] = test_task
# collection.tasks.aliases["t"] = "test"
```

#### 示例 2: 显式命名空间

```python
# tasks.py
from invoke import Collection, task

@task
def top_level(c):
    pass

@task
def sub_task(c):
    pass

# 显式构建命名空间
sub = Collection("sub")
sub.add_task(sub_task)

ns = Collection("root", top_level, sub)
# 或者等价地：
# ns = Collection("root")
# ns.add_task(top_level)
# ns.add_collection(sub)
```

**扫描结果**：
- 模块有 `ns` 属性，是 Collection 类型
- 直接使用 `ns`，不进行隐式扫描
- 可以访问 `"top-level"` 和 `"sub.sub-task"`

---

## 4. 任务名解析机制分析

### 4.1 Lexicon 的别名机制

在深入 `__getitem__` 之前，需要理解 `Lexicon` 如何处理别名。

**关键代码位置**：`invoke/vendor/lexicon/alias_dict.py:37-86`

```python
def _handle(self, key, value, single, multi, unaliased):
    if key in getattr(self, "aliases", {}):
        target = self.aliases[key]
        if isinstance(target, str):
            return single(self, target, value)
        else:
            # 多目标别名（特殊情况）
            if multi:
                return multi(self, target, value)
            else:
                for subkey in target:
                    single(self, subkey, value)
    else:
        return unaliased(self, key, value)
```

#### 别名解析示例

```python
from invoke.vendor.lexicon import Lexicon

lex = Lexicon()
lex["my-task"] = TaskObject()
lex.alias("t", to="my-task")
lex.alias("task-one", to="my-task")

# 访问别名时自动解析
lex["t"]        # → lex["my-task"]
lex["task-one"] # → lex["my-task"]

# 检查存在性
"t" in lex       # → True
"my-task" in lex # → True
```

### 4.2 `__getitem__` 解析机制

**关键代码位置**：`invoke/collection.py:358-414`

```python
def __getitem__(self, name: Optional[str] = None) -> Any:
    return self.task_with_config(name)[0]
```

实际解析逻辑在 `task_with_config()` 中：

```python
def task_with_config(
    self, name: Optional[str]
) -> Tuple[str, Dict[str, Any]]:
```

#### 解析流程

**阶段 1: 获取当前集合的配置**

```python
ours = self.configuration()
```

**阶段 2: 处理空名称（默认任务）**

```python
if not name:
    if not self.default:
        raise ValueError("This collection has no default task.")
    return self[self.default], ours
```

- 空字符串或 `None` → 返回默认任务
- 如果没有默认任务，抛出 `ValueError`

**阶段 3: 应用名称转换**

```python
name = self.transform(name)
```

**阶段 4: 处理带点号的路径**

```python
if "." in name:
    coll, rest = self._split_path(name)
    return self._task_with_merged_config(coll, rest, ours)
```

**路径拆分**（`invoke/collection.py:331-344`）：

```python
def _split_path(self, path: str) -> Tuple[str, str]:
    parts = path.split(".")
    coll = parts.pop(0)
    rest = ".".join(parts)
    return coll, rest
```

**示例**：
- `"build.docs.html"` → `("build", "docs.html")`
- `"sub.task"` → `("sub", "task")`

**递归查找**（`invoke/collection.py:374-378`）：

```python
def _task_with_merged_config(
    self, coll: str, rest: str, ours: Dict[str, Any]
) -> Tuple[str, Dict[str, Any]]:
    task, config = self.collections[coll].task_with_config(rest)
    return task, dict(config, **ours)
```

**关键点**：
- 递归调用子 Collection 的 `task_with_config()`
- 配置合并：`dict(config, **ours)` → 父集合的配置覆盖子集合

**阶段 5: 处理子集合默认任务**

```python
if name in self.collections:
    return self._task_with_merged_config(name, "", ours)
```

- 如果名称直接是一个子 Collection 的名称
- 递归查找时传入空字符串 `""` → 触发子集合的默认任务查找

**阶段 6: 普通任务查找**

```python
return self.tasks[name], ours
```

- 从 `self.tasks` Lexicon 中查找
- 支持别名自动解析

### 4.3 解析流程图解

#### 示例 1: 直接任务名 `"my-task"`

```
collection["my-task"]
         │
         ▼
    task_with_config("my-task")
         │
         ▼
    name = "my-task" (transform 后)
         │
         ▼
    "." in "my-task"? → No
         │
         ▼
    "my-task" in collections? → No
         │
         ▼
    return tasks["my-task"], config
```

#### 示例 2: 嵌套路径 `"sub.my-task"`

```
collection["sub.my-task"]
         │
         ▼
    task_with_config("sub.my-task")
         │
         ▼
    "." in "sub.my-task"? → Yes
         │
         ▼
    coll, rest = ("sub", "my-task")
         │
         ▼
    collections["sub"].task_with_config("my-task")
         │
         ▼
    (递归到子 Collection)
         │
         ▼
    子 Collection 返回 (task, sub_config)
         │
         ▼
    return task, dict(sub_config, **parent_config)
```

#### 示例 3: 默认任务（空名称）

```
collection[""] 或 collection[None]
         │
         ▼
    task_with_config("")
         │
         ▼
    not name? → Yes
         │
         ▼
    self.default 存在?
    ├── Yes → return self[self.default], config
    └── No  → raise ValueError("This collection has no default task.")
```

#### 示例 4: 子集合默认任务

```
collection["sub"]  # "sub" 是子 Collection 名
         │
         ▼
    task_with_config("sub")
         │
         ▼
    "." in "sub"? → No
         │
         ▼
    "sub" in self.collections? → Yes
         │
         ▼
    _task_with_merged_config("sub", "", parent_config)
         │
         ▼
    collections["sub"].task_with_config("")
         │
         ▼
    (子 Collection 处理空名称 → 返回其默认任务)
```

### 4.4 完整解析流程图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    collection[name] → 调用 task_with_config(name)           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 获取配置: ours = self.configuration()                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 空名称检查?                                                               │
│     if not name:                                                              │
│         if self.default:                                                      │
│             → return self[self.default], ours  (递归查找默认任务)           │
│         else:                                                                 │
│             → raise ValueError("This collection has no default task.")      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │ (name 非空)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 应用名称转换: name = self.transform(name)                                 │
│     - auto_dash_names=True: my_task → my-task                               │
│     - auto_dash_names=False: my-task → my_task                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 带点号的路径?                                                            │
│     if "." in name:                                                          │
│         coll, rest = self._split_path(name)                                  │
│         → 递归: self.collections[coll].task_with_config(rest)               │
│         → 合并配置: dict(sub_config, **ours)                                 │
│         → return task, merged_config                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │ (无点号)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 是子 Collection 名?                                                       │
│     if name in self.collections:                                              │
│         → 递归查找默认任务: _task_with_merged_config(name, "", ours)        │
│         → 即: self.collections[name].task_with_config("")                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │ (不是)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. 普通任务查找: return self.tasks[name], ours                              │
│                                                                              │
│     Lexicon 支持别名自动解析:                                                │
│     - 如果 name 是别名 → 自动解析到实际 key                                  │
│     - 例如: tasks["t"] 等价于 tasks["my-task"] 如果有 alias("t", "my-task")│
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.5 `__contains__` 机制

**关键代码位置**：`invoke/collection.py:416-421`

```python
def __contains__(self, name: str) -> bool:
    try:
        self[name]
        return True
    except KeyError:
        return False
```

**简单直接的实现**：
- 尝试调用 `self[name]`
- 如果成功（不抛异常）→ 返回 `True`
- 如果捕获 `KeyError` → 返回 `False`

**注意**：
- 空名称 `""` 或 `None` 如果没有默认任务，会抛出 `ValueError`，不是 `KeyError`
- 这种情况下 `__contains__` 不会捕获，异常会向上传播

---

## 5. Executor.execute() 任务定位流程

### 5.1 完整调用链

**关键代码位置**：`invoke/executor.py:52-179`

```python
def execute(self, *tasks) -> Dict["Task", "Result"]:
    # 阶段 1: 标准化输入 → Call 对象列表
    calls = self.normalize(tasks)
    
    # 阶段 2: 展开 pre/post 任务
    expanded = self.expand_calls(calls)
    
    # 阶段 3: 去重
    calls = self.dedupe(expanded) if dedupe else expanded
    
    # 阶段 4: 执行
    for call in calls:
        # ...
        result = call.task(*args, **call.kwargs)
        # ...
```

### 5.2 normalize() 详细分析

**关键代码位置**：`invoke/executor.py:151-179`

```python
def normalize(
    self,
    tasks: Tuple[
        Union[str, Tuple[str, Dict[str, Any]], ParserContext], ...
    ],
) -> List["Call"]:
    calls = []
    for task in tasks:
        name: Optional[str]
        if isinstance(task, str):
            # 情况 1: 字符串任务名
            name = task
            kwargs = {}
        elif isinstance(task, ParserContext):
            # 情况 2: ParserContext（CLI 解析结果）
            name = task.name
            kwargs = task.as_kwargs
        else:
            # 情况 3: 元组 (name, kwargs)
            name, kwargs = task
        
        # 关键：通过 Collection.__getitem__ 定位 Task
        c = Call(self.collection[name], kwargs=kwargs, called_as=name)
        calls.append(c)
    
    # 处理空任务列表 → 使用默认任务
    if not tasks and self.collection.default is not None:
        calls = [Call(self.collection[self.collection.default])]
    
    return calls
```

### 5.3 任务定位的关键步骤

#### 步骤 1: 输入类型判断

`normalize()` 接受三种输入格式：

| 输入类型 | 示例 | name 来源 | kwargs 来源 |
|---------|------|----------|-------------|
| `str` | `"build"` | 字符串本身 | `{}` |
| `ParserContext` | CLI 解析结果 | `task.name` | `task.as_kwargs` |
| `Tuple[str, dict]` | `("build", {"verbose": True})` | 元组第一个元素 | 元组第二个元素 |

#### 步骤 2: 核心定位 - `self.collection[name]`

这是整个定位流程的核心：

```python
c = Call(self.collection[name], kwargs=kwargs, called_as=name)
```

**发生的事情**：
1. `self.collection[name]` 调用 `Collection.__getitem__`
2. `__getitem__` 调用 `task_with_config(name)`
3. 按照第 4 节的解析流程查找任务
4. 返回的 Task 被包装成 `Call` 对象

#### 步骤 3: Call 对象包装

**关键代码位置**：`invoke/tasks.py:363-490`（部分）

```python
class Call:
    def __init__(
        self,
        task: "Task",
        called_as: Optional[str] = None,
        args: Optional[Tuple[Any, ...]] = None,
        kwargs: Optional[Dict[str, Any]] = None,
    ):
        self.task = task
        self.called_as = called_as
        self.args = args if args else ()
        self.kwargs = kwargs if kwargs else {}
```

**`called_as` 的作用**：
- 记录任务是通过什么名称被调用的
- 用于 `Executor` 中的配置加载：
  ```python
  # executor.py:134
  collection_config = self.collection.configuration(call.called_as)
  ```

### 5.4 完整定位流程图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Executor.execute("build.docs", "test")                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  normalize(("build.docs", "test"))                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  遍历 tasks:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ task = "build.docs" (str)                                                ││
│  │ → name = "build.docs", kwargs = {}                                       ││
│  │ → c = Call(self.collection["build.docs"], kwargs={}, called_as="build.docs")│
│  │                                                                           ││
│  │     self.collection["build.docs"] 触发:                                   ││
│  │     ├── task_with_config("build.docs")                                   ││
│  │     ├── "." in name? → Yes                                               ││
│  │     ├── coll, rest = ("build", "docs")                                   ││
│  │     ├── collections["build"].task_with_config("docs")                    ││
│  │     └── 返回 (task, merged_config)                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ task = "test" (str)                                                       ││
│  │ → name = "test", kwargs = {}                                              ││
│  │ → c = Call(self.collection["test"], kwargs={}, called_as="test")        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  calls = [Call(build_docs_task, ...), Call(test_task, ...)]                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  expand_calls(calls) → 展开 pre/post 任务                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  dedupe(expanded) → 去重（可选）                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  for call in calls:                                                          │
│      # call.task 就是实际的 Task 实例                                        │
│      result = call.task(context, *call.args, **call.kwargs)                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 配置加载的联动

**关键代码位置**：`invoke/executor.py:129-136`

```python
for call in calls:
    config = self.config
    # 使用 called_as 加载集合配置
    collection_config = self.collection.configuration(call.called_as)
    config.load_collection(collection_config)
    config.load_shell_env()
```

**`called_as` 的重要性**：
- 任务可能有多个别名，但配置是按调用路径加载的
- 例如：`collection["sub.task"]` 的 `called_as` 是 `"sub.task"`
- 这会触发路径上所有 Collection 的配置合并

**`configuration(taskpath)` 机制**（`invoke/collection.py:544-560`）：

```python
def configuration(self, taskpath: Optional[str] = None) -> Dict[str, Any]:
    if taskpath is None:
        return copy_dict(self._configuration)
    return self.task_with_config(taskpath)[1]
```

- 传入任务路径时，调用 `task_with_config()` 获取合并后的配置
- 这就是为什么父 Collection 的配置会覆盖子 Collection

---

## 6. 实际场景分析

### 6.1 场景 1: 基础任务查找

```python
# tasks.py
from invoke import task

@task
def build(c):
    print("Building")

@task(aliases=["t"])
def test(c):
    print("Testing")
```

**使用方式**：
```python
# CLI
invoke build
invoke test
invoke t  # 别名

# 代码
from invoke import Collection, Executor
from invoke.config import Config

c = Collection.from_module(__import__('tasks'))
executor = Executor(collection=c, config=Config())

executor.execute("build")      # 查找 tasks["build"]
executor.execute("test")       # 查找 tasks["test"]
executor.execute("t")          # 别名解析 → tasks["test"]
```

### 6.2 场景 2: 嵌套命名空间

```python
# tasks.py
from invoke import Collection, task

@task
def top(c):
    pass

@task
def html(c):
    pass

@task
def pdf(c):
    pass

# 构建嵌套结构
docs = Collection("docs")
docs.add_task(html, default=True)  # html 是 docs 的默认任务
docs.add_task(pdf)

ns = Collection(top, docs)
```

**任务名解析**：
```python
ns["top"]           # → top 任务
ns["docs"]          # → docs 的默认任务 (html)
ns["docs.html"]     # → html 任务
ns["docs.pdf"]      # → pdf 任务

# 验证
assert ns["docs"] is ns["docs.html"]  # True
```

### 6.3 场景 3: 多级嵌套 + 配置合并

```python
# tasks.py
from invoke import Collection, task

@task
def show_config(c):
    print(c.config.get("key", "not found"))

# 三级嵌套
leaf = Collection("leaf")
leaf.add_task(show_config)
leaf.configure({"key": "leaf-value"})

middle = Collection("middle", leaf)
middle.configure({"key": "middle-value", "extra": "foo"})

root = Collection("root", middle)
root.configure({"key": "root-value"})
```

**配置优先级**（父覆盖子）：
```python
# 查找路径: root.middle.leaf.show-config
task, config = root.task_with_config("middle.leaf.show-config")

# config 合并结果:
# {
#   "key": "root-value",    # root 覆盖 middle 和 leaf
#   "extra": "foo"           # 来自 middle
# }

# 验证
assert config["key"] == "root-value"   # 父级覆盖子级
assert config["extra"] == "foo"         # 来自 middle
```

### 6.4 场景 4: 连字符/下划线转换

```python
# tasks.py
from invoke import task

@task
def my_task(c):
    pass

@task
def another_task(c):
    pass
```

**不同调用方式**：
```python
# auto_dash_names=True (默认)
c = Collection.from_module(module)
# c.tasks 的 key 是: "my-task", "another-task"

# 都能解析:
c["my_task"]       # transform 后 → "my-task" ✓
c["my-task"]       # 直接匹配 ✓
c["another_task"]  # transform 后 → "another-task" ✓

# auto_dash_names=False
c = Collection.from_module(module, auto_dash_names=False)
# c.tasks 的 key 是: "my_task", "another_task"

# 解析:
c["my_task"]       # ✓
c["my-task"]       # transform 后 → "my_task" ✓
```

---

## 7. 总结与关键洞察

### 7.1 Collection.from_module() 核心机制

| 机制 | 说明 |
|------|------|
| **双模式扫描** | 先检查显式 `ns`/`namespace`，无则隐式扫描 |
| **过滤条件** | `isinstance(x, Task)` - 只扫描 Task 实例 |
| **名称优先级** | 显式 name > task.name > func_name > `__name__` |
| **自动转换** | `auto_dash_names` 控制下划线↔连字符转换 |
| **别名注册** | `task.aliases` + `add_task(aliases=...)` 都生效 |
| **默认任务** | `default=True` 参数 或 `task.is_default` 属性 |

### 7.2 任务名解析流程

**解析顺序**：
1. **空名称** → 默认任务
2. **带点号** → 递归查找子 Collection
3. **子 Collection 名** → 子 Collection 的默认任务
4. **普通任务** → 从 `tasks` Lexicon 查找（支持别名）

**配置合并**：
- 父 Collection 的配置覆盖子 Collection
- 路径上所有 Collection 的配置都会合并

### 7.3 Executor 定位链

**关键路径**：
```
Executor.execute(name)
    → Executor.normalize()
        → Collection.__getitem__(name)
            → Collection.task_with_config(name)
                → (递归解析嵌套路径)
                → Lexicon.__getitem__ (别名解析)
        → Call(task, called_as=name)
    → (展开/去重)
    → call.task(context, **call.kwargs)
```

### 7.4 设计亮点

1. **Lexicon 的别名设计**：
   - 透明的别名解析，用户无需关心是真实 key 还是别名
   - 多目标别名支持（虽然 invoke 中主要用单目标）

2. **递归路径解析**：
   - 点号分隔的路径自然映射到嵌套 Collection
   - 配置随路径逐层合并，父级覆盖子级

3. **called_as 追踪**：
   - 记录实际调用名称，用于配置加载
   - 区分"任务本身"和"如何被调用"

4. **显式/隐式双重模式**：
   - 简单场景：隐式扫描，无需手动构建 Collection
   - 复杂场景：显式构建 `ns` Collection，完全控制结构

---

## 8. 代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| Collection.from_module() | `invoke/collection.py` | 145-236 |
| Collection.add_task() | `invoke/collection.py` | 238-283 |
| Collection.add_collection() | `invoke/collection.py` | 285-324 |
| Collection.transform() | `invoke/collection.py` | 456-493 |
| Collection.__getitem__() | `invoke/collection.py` | 358-372 |
| Collection.task_with_config() | `invoke/collection.py` | 380-414 |
| Collection.__contains__() | `invoke/collection.py` | 416-421 |
| Collection._split_path() | `invoke/collection.py` | 331-344 |
| Collection.configuration() | `invoke/collection.py` | 544-560 |
| Executor.execute() | `invoke/executor.py` | 52-149 |
| Executor.normalize() | `invoke/executor.py` | 151-179 |
| Lexicon 类 | `invoke/vendor/lexicon/__init__.py` | 6-24 |
| AliasDict 类 | `invoke/vendor/lexicon/alias_dict.py` | 1-95 |
| AttributeDict 类 | `invoke/vendor/lexicon/attribute_dict.py` | 1-16 |

---

**分析日期**：2026-04-27
**分析版本**：invoke-7066
