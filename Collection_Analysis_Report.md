# Invoke Collection 机制深度分析报告

## 目录

1. [@task 装饰器的工作机制](#1-task-装饰器的工作机制)
2. [Collection 的注册机制](#2-collection-的注册机制)
3. [嵌套 Collection 的命名空间拼接](#3-嵌套-collection-的命名空间拼接)
4. [inv --list 输出树的生成](#4-inv---list-输出树的生成)
5. [层级合并规则](#5-层级合并规则)
6. [同名任务的查找优先级](#6-同名任务的查找优先级)

---

## 1. @task 装饰器的工作机制

### 1.1 核心实现

`@task` 装饰器的核心实现位于 `invoke/tasks.py`，它**并不直接将函数注册到 Collection**，而是将函数包装成 `Task` 对象。

```python
def task(*args: Any, **kwargs: Any) -> Callable:
    klass: Type[Task] = kwargs.pop("klass", Task)
    # @task -- 无参数形式
    if len(args) == 1 and callable(args[0]) and not isinstance(args[0], Task):
        return klass(args[0], **kwargs)
    # @task(pre, tasks, here) -- 带参数形式
    if args:
        if "pre" in kwargs:
            raise TypeError("May not give *args and 'pre' kwarg simultaneously!")
        kwargs["pre"] = args

    def inner(body: Callable) -> Task[T]:
        _task = klass(body, **kwargs)
        return _task

    return inner
```
*invoke/tasks.py:289-360*

### 1.2 两种使用方式

| 方式 | 示例 | 说明 |
|------|------|------|
| 无参数 | `@task` | 直接装饰函数，使用默认配置 |
| 带参数 | `@task(name="mytask", aliases=["t1"])` | 自定义任务名、别名等 |

### 1.3 Task 对象的属性

`Task` 类封装了任务的所有元数据：

| 属性 | 说明 |
|------|------|
| `body` | 实际的可调用函数 |
| `name` | 任务名称（默认使用函数名） |
| `aliases` | 任务别名列表 |
| `is_default` | 是否为所在集合的默认任务 |
| `pre` / `post` | 前置/后置任务列表 |
| `autoprint` | 是否自动打印返回值 |

### 1.4 关键发现

**`@task` 装饰器本身不负责注册到 Collection**，它只是创建 `Task` 对象。实际的注册发生在以下场景：

1. **显式调用 `Collection.add_task()`**
2. **通过 `Collection.from_module()` 自动发现**（扫描模块中的 `Task` 实例）

---

## 2. Collection 的注册机制

### 2.1 Collection 的核心结构

`Collection` 类位于 `invoke/collection.py`，使用两个 `Lexicon` 对象分别存储任务和子集合：

```python
class Collection:
    def __init__(self, *args: Any, **kwargs: Any) -> None:
        self.tasks = Lexicon()        # 存储任务：name -> Task
        self.collections = Lexicon()  # 存储子集合：name -> Collection
        self.default: Optional[str] = None  # 默认任务/集合名
        self.name = None              # 集合名称（用于嵌套时的命名空间）
        self._configuration: Dict[str, Any] = {}  # 配置数据
        # ...
```
*invoke/collection.py:12-114*

### 2.2 Lexicon 数据结构

`Lexicon` 是一个支持别名的字典，继承自 `AttributeDict` 和 `AliasDict`：

```python
class Lexicon(AttributeDict, AliasDict):
    def __init__(self, *args, **kwargs):
        dict.__init__(self, *args, **kwargs)
        dict.__setattr__(self, "aliases", {})  # 别名映射：alias -> real_name
```
*invoke/vendor/lexicon/__init__.py:6-16*

**关键特性**：
- 支持属性方式访问（`coll.tasks.my_task`）
- 支持别名机制（`tasks.alias("alias", to="real_name")`）

### 2.3 任务注册：add_task()

```python
def add_task(
    self,
    task: "Task",
    name: Optional[str] = None,
    aliases: Optional[Tuple[str, ...]] = None,
    default: Optional[bool] = None,
) -> None:
    # 1. 确定任务名称
    if name is None:
        if task.name:
            name = task.name
        elif hasattr(task.body, "func_name"):
            name = task.body.func_name
        elif hasattr(task.body, "__name__"):
            name = task.__name__
        else:
            raise ValueError("Could not obtain a name for this task!")
    
    # 2. 名称转换（下划线转连字符）
    name = self.transform(name)
    
    # 3. 冲突检查：不能与现有集合同名
    if name in self.collections:
        raise ValueError(f"Name conflict: sub-collection named {name!r} already exists")
    
    # 4. 注册任务
    self.tasks[name] = task
    
    # 5. 注册别名
    for alias in list(task.aliases) + list(aliases or []):
        self.tasks.alias(self.transform(alias), to=name)
    
    # 6. 设置默认任务
    if default is True or (default is None and task.is_default):
        self._check_default_collision(name)
        self.default = name
```
*invoke/collection.py:238-283*

### 2.4 子集合注册：add_collection()

```python
def add_collection(
    self,
    coll: "Collection",
    name: Optional[str] = None,
    default: Optional[bool] = None,
) -> None:
    # 1. 支持模块对象（自动转换为 Collection）
    if isinstance(coll, ModuleType):
        coll = Collection.from_module(coll)
    
    # 2. 确定集合名称
    name = name or coll.name
    if not name:
        raise ValueError("Non-root collections must have a name!")
    name = self.transform(name)
    
    # 3. 冲突检查：不能与现有任务同名
    if name in self.tasks:
        raise ValueError(f"Name conflict: task named {name!r} already exists")
    
    # 4. 注册子集合
    self.collections[name] = coll
    
    # 5. 设置默认集合
    if default:
        self._check_default_collision(name)
        self.default = name
```
*invoke/collection.py:285-324*

### 2.5 模块自动发现：from_module()

这是最常用的注册方式，`Collection.from_module()` 会自动扫描模块中的任务：

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
    # 1. 检查是否有显式定义的命名空间（ns 或 namespace）
    for candidate in ("ns", "namespace"):
        obj = getattr(module, candidate, None)
        if obj and isinstance(obj, Collection):
            # 使用显式定义的 Collection
            ret = instantiate(obj_name=obj.name)
            ret.tasks = ret._transform_lexicon(obj.tasks)
            ret.collections = ret._transform_lexicon(obj.collections)
            ret.default = ret.transform(obj.default) if obj.default else None
            # 配置合并
            obj_config = copy_dict(obj._configuration)
            if config:
                merge_dicts(obj_config, config)
            ret._configuration = obj_config
            return ret
    
    # 2. 如果没有显式命名空间，自动扫描模块中的 Task 实例
    tasks = filter(lambda x: isinstance(x, Task), vars(module).values())
    collection = instantiate()
    for task in tasks:
        collection.add_task(task)
    if config:
        collection.configure(config)
    return collection
```
*invoke/collection.py:146-236*

### 2.6 注册流程图

```
用户编写 tasks.py
        │
        ▼
┌─────────────────┐
│ @task 装饰函数  │ ──▶ 创建 Task 对象
└─────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│ Collection.from_module(tasks_module)│
└─────────────────────────────────────┘
        │
        ├─────────────────────────────┐
        │ 检查是否有 ns/namespace      │
        ▼                             ▼
┌─────────────────┐         ┌──────────────────┐
│ 有显式命名空间   │         │ 无显式命名空间     │
└─────────────────┘         └──────────────────┘
        │                             │
        ▼                             ▼
┌─────────────────┐         ┌──────────────────┐
│ 使用已定义的    │         │ 扫描模块中的      │
│ Collection      │         │ Task 实例         │
└─────────────────┘         └──────────────────┘
        │                             │
        └─────────────────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │ 完成注册     │
              └──────────────┘
```

---

## 3. 嵌套 Collection 的命名空间拼接

### 3.1 核心机制：task_names 属性

`Collection.task_names` 是一个属性，它递归地生成所有任务的完整路径名（带命名空间前缀）：

```python
@property
def task_names(self) -> Dict[str, List[str]]:
    ret = {}
    # 1. 当前集合的任务：无前缀
    for name, task in self.tasks.items():
        ret[name] = list(map(self.transform, task.aliases))
    
    # 2. 子集合的任务：添加命名空间前缀
    for coll_name, coll in self.collections.items():
        for task_name, aliases in coll.task_names.items():
            # 别名也需要添加前缀
            aliases = list(
                map(lambda x: self.subtask_name(coll_name, x), aliases)
            )
            # 如果是子集合的默认任务，添加子集合名作为别名
            if coll.default == task_name:
                aliases += (coll_name,)
            # 拼接完整路径
            ret[self.subtask_name(coll_name, task_name)] = aliases
    return ret
```
*invoke/collection.py:512-542*

### 3.2 命名空间拼接函数

```python
def subtask_name(self, collection_name: str, task_name: str) -> str:
    return ".".join(
        [self.transform(collection_name), self.transform(task_name)]
    )
```
*invoke/collection.py:451-454*

### 3.3 示例分析

假设有以下结构：

```python
# 根集合
root = Collection()

# 任务：test
@task
def test(c):
    pass
root.add_task(test)

# 子集合：build
build = Collection('build')

@task
def compile(c):
    pass
build.add_task(compile)

# build 的子集合：docs
docs = Collection('docs')

@task(default=True)
def html(c):
    pass
docs.add_task(html)

build.add_collection(docs)
root.add_collection(build)
```

**生成的 task_names 结果**：

```python
{
    'test': [],                    # 根集合的任务
    'build.compile': [],           # build 子集合的任务
    'build.docs.html': ['build.docs']  # docs 子集合的默认任务
}
```

### 3.4 命名空间转换规则

**自动连字符转换**（由 `transform()` 方法控制）：

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
        # 不替换首尾的下划线，也不替换点号旁边的下划线
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
*invoke/collection.py:456-493*

**转换示例**：

| 原始名称 | auto_dash_names=True | auto_dash_names=False |
|----------|----------------------|-----------------------|
| `my_task` | `my-task` | `my_task` |
| `_private_` | `_private_` | `_private_` |
| `sub.my_task` | `sub.my-task` | `sub.my_task` |

### 3.5 命名空间拼接流程图

```
根 Collection (root)
        │
        ├── tasks: {'test': Task}
        │
        └── collections: {'build': Collection}
                              │
                              ├── tasks: {'compile': Task}
                              │
                              └── collections: {'docs': Collection}
                                                    │
                                                    └── tasks: {'html': Task}
                                                              (default=True)

                              ▼ 递归计算 task_names

root.task_names = {
    'test': [],
    'build.compile': [],
    'build.docs.html': ['build.docs']  # 默认任务的额外别名
}
```

---

## 4. inv --list 输出树的生成

### 4.1 入口方法：list_tasks()

```python
def list_tasks(self) -> None:
    focus = self.scoped_collection
    if not focus:
        msg = "No tasks found in collection '{}'!"
        raise Exit(msg.format(focus.name))
    # 根据 list_format 选择不同的输出方式
    getattr(self, "list_{}".format(self.list_format))()
```
*invoke/program.py:807-815*

### 4.2 支持的输出格式

| 格式 | 方法 | 说明 |
|------|------|------|
| `flat` | `list_flat()` | 扁平列表，显示完整路径 |
| `nested` | `list_nested()` | 树状结构，缩进显示层级 |
| `json` | `list_json()` | JSON 格式输出 |

### 4.3 核心生成方法：_make_pairs()

这是生成任务列表的核心方法，支持递归和多种显示格式：

```python
def _make_pairs(
    self,
    coll: "Collection",
    ancestors: Optional[List[str]] = None,
) -> List[Tuple[str, Optional[str]]]:
    if ancestors is None:
        ancestors = []
    
    pairs = []
    indent = len(ancestors) * self.indent  # 缩进计算
    ancestor_path = ".".join(x for x in ancestors)  # 祖先路径
    
    # 1. 处理当前集合的任务
    for name, task in sorted(coll.tasks.items()):
        is_default = name == coll.default
        displayname = name
        aliases = list(map(coll.transform, sorted(task.aliases)))
        
        # 如果是子集合中的任务，添加前导点
        if ancestors or self.list_root:
            displayname = ".{}".format(displayname)
            aliases = [".{}".format(x) for x in aliases]
        
        # Nested 格式：缩进 + 默认任务标记
        if self.list_format == "nested":
            prefix = indent
            if is_default:
                displayname += "*"  # 默认任务用 * 标记
        
        # Flat 格式：完整路径 + 默认任务别名
        if self.list_format == "flat":
            prefix = ancestor_path
            if prefix and self.list_root:
                prefix = "." + prefix
            aliases = [prefix + alias for alias in aliases]
            if is_default and ancestors:
                aliases.insert(0, prefix)  # 默认任务添加集合名作为别名
        
        # 生成显示字符串
        alias_str = " ({})".format(", ".join(aliases)) if aliases else ""
        full = prefix + displayname + alias_str
        pairs.append((full, helpline(task)))
    
    # 2. 处理子集合（递归）
    truncate = self.list_depth and (len(ancestors) + 1) >= self.list_depth
    for name, subcoll in sorted(coll.collections.items()):
        displayname = name
        if ancestors or self.list_root:
            displayname = ".{}".format(displayname)
        
        # 如果到达最大深度，显示摘要
        if truncate:
            tallies = [
                "{} {}".format(len(getattr(subcoll, attr)), attr)
                for attr in ("tasks", "collections")
                if getattr(subcoll, attr)
            ]
            displayname += " [{}]".format(", ".join(tallies))
        
        # Nested 格式：显示集合名
        if self.list_format == "nested":
            pairs.append((indent + displayname, helpline(subcoll)))
        elif self.list_format == "flat" and truncate:
            pairs.append((ancestor_path + displayname, helpline(subcoll)))
        
        # 递归处理子集合
        if not truncate:
            recursed_pairs = self._make_pairs(
                coll=subcoll, ancestors=ancestors + [name]
            )
            pairs.extend(recursed_pairs)
    
    return pairs
```
*invoke/program.py:826-893*

### 4.4 输出示例

假设任务结构如下：

```
root
├── test (default=True, aliases=['t'])
└── build
    ├── compile (aliases=['c'])
    └── docs
        └── html (default=True)
```

**Flat 格式输出**：
```
Available tasks:

  build.compile (build.c)   Compile the project
  build.docs.html (build.docs)  Build HTML docs
  test (t)                   Run tests

Default task: test
```

**Nested 格式输出**：
```
Available tasks (* denotes collection defaults):

  test* (t)                   Run tests
  build                       Build-related tasks
    compile (c)               Compile the project
    docs                      Documentation tasks
      html*                   Build HTML docs
```

### 4.5 默认任务的特殊处理

| 场景 | Flat 格式 | Nested 格式 |
|------|-----------|-------------|
| 根集合默认任务 | 显示 `(别名)` + "Default task" 提示 | 任务名后加 `*` |
| 子集合默认任务 | 别名列表包含子集合名 | 任务名后加 `*` |
| 通过子集合名调用 | `inv build` → 调用 `build.docs.html` | 同上 |

---

## 5. 层级合并规则

### 5.1 配置合并机制

配置合并发生在 `task_with_config()` 方法中，当查找任务时会合并路径上所有集合的配置：

```python
def task_with_config(
    self, name: Optional[str]
) -> Tuple[str, Dict[str, Any]]:
    # 1. 获取当前集合的配置
    ours = self.configuration()
    
    # 2. 默认任务处理
    if not name:
        if not self.default:
            raise ValueError("This collection has no default task.")
        return self[self.default], ours
    
    # 3. 名称转换
    name = self.transform(name)
    
    # 4. 带点号的路径：递归查找子集合
    if "." in name:
        coll, rest = self._split_path(name)
        return self._task_with_merged_config(coll, rest, ours)
    
    # 5. 子集合名：查找子集合的默认任务
    if name in self.collections:
        return self._task_with_merged_config(name, "", ours)
    
    # 6. 普通任务查找
    return self.tasks[name], ours
```
*invoke/collection.py:380-414*

### 5.2 核心合并方法

```python
def _task_with_merged_config(
    self, coll: str, rest: str, ours: Dict[str, Any]
) -> Tuple[str, Dict[str, Any]]:
    # 递归获取子集合的任务和配置
    task, config = self.collections[coll].task_with_config(rest)
    # 合并：父集合配置覆盖子集合配置
    return task, dict(config, **ours)
```
*invoke/collection.py:374-378*

### 5.3 合并规则详解

**关键代码**：`dict(config, **ours)`

这表示：
- `config`：子集合的配置（内层）
- `ours`：当前集合的配置（外层）
- **父集合的配置会覆盖子集合的配置**

### 5.4 合并顺序示例

假设有以下嵌套结构和配置：

```python
# 根集合配置
root.configure({
    'log_level': 'info',
    'timeout': 30,
    'db': {'host': 'localhost'}
})

# build 子集合配置
build.configure({
    'log_level': 'debug',  # 覆盖父级
    'parallel': 4,         # 新增
    'db': {'port': 5432}   # 嵌套合并
})

# docs 子集合配置
docs.configure({
    'log_level': 'warning'  # 覆盖父级
})
```

**调用不同任务时的配置**：

| 任务路径 | 合并后的配置 |
|----------|-------------|
| `root_task` | `{'log_level': 'info', 'timeout': 30, 'db': {'host': 'localhost'}}` |
| `build.compile` | `{'log_level': 'info', 'timeout': 30, 'parallel': 4, 'db': {'host': 'localhost', 'port': 5432}}` |
| `build.docs.html` | `{'log_level': 'info', 'timeout': 30, 'db': {'host': 'localhost'}}` |

### 5.5 配置合并流程图

```
调用: inv build.docs.html
        │
        ▼
┌─────────────────────────────────────────┐
│ root.task_with_config("build.docs.html")│
└─────────────────────────────────────────┘
        │
        ├── ours = root.configuration()
        │   {'log_level': 'info', 'timeout': 30}
        │
        └── name 包含 "." → 递归
            │
            ▼
┌─────────────────────────────────────────────┐
│ _task_with_merged_config("build", "docs.html", ours) │
└─────────────────────────────────────────────┘
        │
        ├── 递归调用 build.task_with_config("docs.html")
        │   │
        │   ├── build_ours = build.configuration()
        │   │   {'log_level': 'debug', 'parallel': 4}
        │   │
        │   └── 继续递归
        │       │
        │       ▼
        ┌─────────────────────────────────────────┐
        │ _task_with_merged_config("docs", "html", build_ours) │
        └─────────────────────────────────────────┘
        │
        ├── 递归调用 docs.task_with_config("html")
        │   │
        │   ├── 返回: (html_task, docs.configuration())
        │   │          {'log_level': 'warning'}
        │   │
        │   └── 合并: dict(docs_config, **build_ours)
        │          → {'log_level': 'debug', 'parallel': 4}
        │
        └── 合并: dict(build_merged_config, **root_ours)
               → {'log_level': 'info', 'timeout': 30, 'parallel': 4}
```

### 5.6 嵌套字典的合并

`configure()` 方法使用 `merge_dicts()` 进行递归合并：

```python
def configure(self, options: Dict[str, Any]) -> None:
    merge_dicts(self._configuration, options)
```
*invoke/collection.py:562-579*

**merge_dicts 的行为**：
- 对于普通键：新值覆盖旧值
- 对于字典值：递归合并
- 这意味着子集合的嵌套配置会与父集合的嵌套配置合并

---

## 6. 同名任务的查找优先级

### 6.1 查找入口：__getitem__

```python
def __getitem__(self, name: Optional[str] = None) -> Any:
    return self.task_with_config(name)[0]
```
*invoke/collection.py:358-372*

### 6.2 查找顺序详解

根据 `task_with_config()` 的实现，查找顺序如下：

```
查找 "task_name"
        │
        ▼
┌─────────────────────────┐
│ 1. name 为空？          │
│    (默认任务查找)        │
└─────────────────────────┘
        │ 是
        ▼
┌─────────────────────────┐
│ 检查 self.default       │
│ 存在 → 返回默认任务      │
│ 不存在 → ValueError     │
└─────────────────────────┘
        │ 否
        ▼
┌─────────────────────────┐
│ 2. 名称转换             │
│    transform(name)      │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 3. 包含 "." ？          │
│    (嵌套路径查找)        │
└─────────────────────────┘
        │ 是
        ▼
┌─────────────────────────┐
│ 拆分路径: coll, rest    │
│ 递归查找子集合           │
│ _task_with_merged_config│
└─────────────────────────┘
        │ 否
        ▼
┌─────────────────────────┐
│ 4. 在 collections 中？  │
│    (子集合默认任务)      │
└─────────────────────────┘
        │ 是
        ▼
┌─────────────────────────┐
│ 查找子集合的默认任务      │
│ _task_with_merged_config│
└─────────────────────────┘
        │ 否
        ▼
┌─────────────────────────┐
│ 5. 在 tasks 中？        │
│    (普通任务)            │
└─────────────────────────┘
        │ 是
        ▼
┌─────────────────────────┐
│ 返回 self.tasks[name]   │
└─────────────────────────┘
        │ 否
        ▼
┌─────────────────────────┐
│ KeyError                │
└─────────────────────────┘
```

### 6.3 关键发现：同层级不会重名

从 `add_task()` 和 `add_collection()` 的实现可以看到：

```python
# add_task 中的冲突检查
if name in self.collections:
    raise ValueError(f"Name conflict: sub-collection named {name!r} already exists")

# add_collection 中的冲突检查
if name in self.tasks:
    raise ValueError(f"Name conflict: task named {name!r} already exists")
```
*invoke/collection.py:275-279, 317-319*

**结论**：在**同一个 Collection 层级**，任务名和集合名**不能重名**。

### 6.4 不同层级的同名处理

虽然同层级不能重名，但**不同层级**可以有同名任务。

#### 示例场景

```python
# 根集合
root = Collection()

@task
def foo(c):
    print("root foo")
root.add_task(foo)

# 子集合 sub
sub = Collection('sub')

@task
def foo(c):
    print("sub foo")
sub.add_task(foo)

root.add_collection(sub)
```

#### 查找行为

| 调用方式 | 实际执行的任务 |
|----------|---------------|
| `inv foo` | `root.foo`（根集合的任务） |
| `inv sub.foo` | `sub.foo`（子集合的任务） |

#### 查找过程详解

**查找 `foo`**：
1. 名称不包含 `.` → 跳过嵌套路径查找
2. 检查 `collections` 中是否有 `foo` → 没有
3. 检查 `tasks` 中是否有 `foo` → 有，返回 `root.foo`

**查找 `sub.foo`**：
1. 名称包含 `.` → 拆分为 `coll="sub"`, `rest="foo"`
2. 递归调用 `sub.task_with_config("foo")`
3. 在 `sub` 中：
   - 名称不包含 `.`
   - 检查 `sub.collections` 中是否有 `foo` → 没有
   - 检查 `sub.tasks` 中是否有 `foo` → 有，返回 `sub.foo`

### 6.5 别名的查找机制

由于 `Collection.tasks` 使用 `Lexicon`（支持别名的字典），查找时会自动解析别名。

**Lexicon 的别名查找**：
```python
# 当访问 tasks["alias"] 时
# 实际会先查 aliases 映射
# aliases["alias"] → "real_name"
# 然后返回 tasks["real_name"]
```

#### 示例

```python
@task(aliases=['f'])
def foo(c):
    pass

root.add_task(foo)
```

| 调用方式 | 实际任务 |
|----------|---------|
| `inv foo` | `foo` |
| `inv f` | `foo`（别名解析） |

### 6.6 默认任务的查找

默认任务有两种访问方式：

| 方式 | 触发条件 | 示例 |
|------|----------|------|
| 空字符串/None | 显式指定 | `collection[""]` |
| 子集合名 | 子集合有默认任务 | `inv build` → 调用 `build` 的默认任务 |

#### 子集合默认任务示例

```python
build = Collection('build')

@task(default=True)
def compile(c):
    print("compiling")
build.add_task(compile)

root.add_collection(build)
```

| 调用方式 | 执行的任务 |
|----------|-----------|
| `inv build` | `build.compile`（默认任务） |
| `inv build.compile` | `build.compile` |

### 6.7 查找优先级总结

| 优先级 | 查找类型 | 说明 |
|--------|----------|------|
| 1 | 显式默认任务 | `collection[""]` 或 `collection[None]` |
| 2 | 嵌套路径（带点号） | `sub.sub.task` → 逐级递归 |
| 3 | 子集合默认任务 | 名称匹配 `collections` 中的键 |
| 4 | 普通任务 | 名称匹配 `tasks` 中的键 |
| 5 | 别名解析 | 上述都不匹配时，尝试别名 |

### 6.8 潜在冲突场景

虽然 Invoke 的设计避免了大多数冲突，但仍有一些需要注意的场景：

#### 场景1：子集合名与任务别名冲突

```python
@task(aliases=['sub'])
def foo(c):
    pass
root.add_task(foo)

sub = Collection('sub')
# 这会失败！因为 'sub' 已经在 tasks 的别名中
root.add_collection(sub)  # ValueError!
```

**原因**：`add_collection()` 检查 `name in self.tasks`，而 `Lexicon.__contains__` 会考虑别名。

#### 场景2：不同层级的任务和子集合同名

```python
# 这是允许的
@task
def build(c):
    pass
root.add_task(build)

build_coll = Collection('build')  # 名称不同（变量名不影响）
# 但这样不行：
root.add_collection(Collection('build'))  # ValueError!
```

---

## 附录

### A. 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| `@task` 装饰器 | `invoke/tasks.py` | 289-360 |
| `Task` 类 | `invoke/tasks.py` | 37-188 |
| `Collection.__init__` | `invoke/collection.py` | 19-114 |
| `Collection.add_task` | `invoke/collection.py` | 238-283 |
| `Collection.add_collection` | `invoke/collection.py` | 285-324 |
| `Collection.from_module` | `invoke/collection.py` | 146-236 |
| `Collection.task_names` | `invoke/collection.py` | 512-542 |
| `Collection.task_with_config` | `invoke/collection.py` | 380-414 |
| `Program._make_pairs` | `invoke/program.py` | 826-893 |
| `Lexicon` 类 | `invoke/vendor/lexicon/__init__.py` | 6-24 |

### B. 数据结构关系图

```
┌─────────────────────────────────────────────────────────────┐
│                         Program                              │
│  (CLI 入口，处理 --list 等参数)                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Collection (root)                       │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │ tasks        │  │ collections      │  │ _configura-  │ │
│  │ (Lexicon)    │  │ (Lexicon)        │  │ tion (dict)  │ │
│  ├──────────────┤  ├──────────────────┤  └──────────────┘ │
│  │ 'test': Task │  │ 'build':         │                    │
│  │ 'foo': Task  │  │  Collection      │                    │
│  ├──────────────┤  └──────────────────┘                    │
│  │ aliases:     │                    │                    │
│  │ 't': 'test'  │                    ▼                    │
│  └──────────────┘         ┌──────────────────┐            │
│                           │ Collection (build)│            │
│                           │  ┌──────────────┐ │            │
│                           │  │ tasks        │ │            │
│                           │  │ 'compile':   │ │            │
│                           │  │ Task         │ │            │
│                           │  └──────────────┘ │            │
│                           └──────────────────┘            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                           Task                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ body: Callable          (实际函数)                    │  │
│  │ name: str                (任务名)                     │  │
│  │ aliases: List[str]       (别名列表)                  │  │
│  │ is_default: bool         (是否默认任务)              │  │
│  │ pre/post: List           (前置/后置任务)             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### C. 术语表

| 术语 | 定义 |
|------|------|
| `Task` | 被 `@task` 装饰的函数包装而成的对象 |
| `Collection` | 任务和子集合的容器，形成命名空间层级 |
| `Lexicon` | 支持别名和属性访问的字典数据结构 |
| 默认任务 | 当调用集合名本身时执行的任务 |
| 命名空间 | 嵌套 Collection 形成的层级结构，用点号分隔 |
| 别名 | 任务的备用名称，可用于调用 |
| `task_names` | Collection 的属性，返回所有任务的完整路径 |

---

**报告生成时间**：2026-04-27

**分析基础**：invoke 项目源码（commit: 未指定）

**分析范围**：
- `invoke/tasks.py` - Task 类和 @task 装饰器
- `invoke/collection.py` - Collection 类实现
- `invoke/program.py` - CLI 入口和列表输出
- `invoke/vendor/lexicon/` - Lexicon 数据结构
- `tests/collection.py` - 测试用例验证
