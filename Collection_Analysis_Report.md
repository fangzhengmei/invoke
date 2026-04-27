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

### 4.2 scoped_collection：控制输出树的范围

**⚠️ 关键概念**：`scoped_collection` 直接决定了 `inv --list` 输出树的**起点和范围**。本节将清晰回答三个问题：

1. **默认值是什么？**
2. **`inv --list <名称>` 时如何切换到子集合？**
3. **对输出树范围有什么影响？**

---

#### 问题1：scoped_collection 的默认值是什么？

**答案**：默认值是 `self.collection`，即整个任务树的**根集合**。

从 `Program.__init__` 的代码可以看到：

```python
# Program 初始化时
self.scoped_collection = self.collection  # 默认指向根集合
```
*invoke/program.py:486*

**`self.collection` 是什么？**

`self.collection` 是 `Program` 类中存储**整个任务树**的根 Collection。它通过 `Loader` 从 `tasks.py` 模块加载：

```python
# Program 中 collection 的来源（简化）
self.collection = self.loader.load_collection(...)
```

**关键点总结**：

| 属性 | 默认值 | 含义 |
|------|--------|------|
| `self.scoped_collection` | `self.collection` | `inv --list` 输出的起始集合 |
| `self.collection` | 从 `tasks.py` 加载的根 Collection | 整个任务树的根 |

**这意味着**：
- 默认情况下，`inv --list` 会从**根集合**开始，显示**所有任务**
- 所有列表输出方法（`list_flat()`, `list_nested()`, `list_json()`）都使用 `self.scoped_collection` 作为递归起点

---

#### 问题2：`inv --list <名称>` 时如何切换到子集合？

**答案**：当使用 `inv --list <namespace>` 形式时，会通过 `subcollection_from_path()` 方法找到对应的子集合，并更新 `scoped_collection`。

让我们追踪完整的代码流程：

##### 步骤1：处理 `--list` 参数

从 `program.py` 的参数处理逻辑：

```python
# 处理 --list 参数的逻辑
if list_root:
    if isinstance(list_root, str):
        # 1. 保存 list_root 值（用于显示格式）
        self.list_root = list_root
        try:
            # 2. 通过路径获取子集合
            sub = self.collection.subcollection_from_path(list_root)
            # 3. ⚠️ 更新 scoped_collection！
            self.scoped_collection = sub
        except KeyError:
            msg = "Sub-collection '{}' not found!"
            raise Exit(msg.format(list_root))
    # 4. 调用 list_tasks() 输出
    self.list_tasks()
```
*invoke/program.py:520-530*

##### 步骤2：`subcollection_from_path()` 方法

这个方法负责**递归查找子集合**：

```python
def subcollection_from_path(self, path: str) -> "Collection":
    # 1. 按点号分割路径
    parts = path.split(".")
    collection = self
    # 2. 逐级查找子集合
    while parts:
        # 从 collections 字典中获取子集合
        collection = collection.collections[parts.pop(0)]
    return collection
```
*invoke/collection.py:346-356*

##### 完整执行流程图

以 `inv --list build.docs` 为例：

```
用户输入: inv --list build.docs
        │
        ▼
┌─────────────────────────────────────────┐
│ 1. list_root = "build.docs"             │
│    (从命令行参数解析)                     │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 2. self.list_root = "build.docs"        │
│    (保存用于显示格式)                     │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 3. self.collection.subcollection_from_path("build.docs")│
│    ├── parts = ["build", "docs"]       │
│    ├── 第一循环: collection = root.collections["build"] │
│    └── 第二循环: collection = build.collections["docs"] │
│    返回: docs 子集合                     │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 4. self.scoped_collection = docs        │
│    ⚠️ 关键：scoped_collection 被更新了！  │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 5. 调用 list_tasks()                    │
│    focus = self.scoped_collection = docs│
│    从 docs 开始输出任务树                 │
└─────────────────────────────────────────┘
```

**关键点**：
- `scoped_collection` 被更新为 `docs` 子集合
- `list_tasks()` 中的 `focus = self.scoped_collection` 现在指向 `docs`
- 所有输出方法都会从 `docs` 开始递归

---

#### 问题3：对输出树范围有什么影响？

**答案**：`scoped_collection` 决定了输出树的**起点**，所有输出方法都会从这个集合开始递归。

让我们用一个具体的任务结构来对比不同参数的输出：

**假设任务结构**：

```
root (Collection)
├── tasks:
│   └── test (Task)
└── collections:
    └── build (Collection)
        ├── tasks:
        │   └── compile (Task, aliases=['c'])
        └── collections:
            └── docs (Collection)
                └── tasks:
                    └── html (Task, default=True)
```

---

##### 场景1：`inv --list`（默认值）

**scoped_collection**：`root`（根集合）

**list_flat() 输出**：
```
Available tasks:

  build.compile (build.c)   Compile the project
  build.docs.html (build.docs)  Build HTML docs
  test                       Run tests

Default task: test
```

**list_nested() 输出**：
```
Available tasks (* denotes collection defaults):

  test*                      Run tests
  build                      Build-related tasks
    compile (c)              Compile the project
    docs                     Documentation tasks
      html*                  Build HTML docs
```

**说明**：
- 从 `root` 开始递归
- 显示**所有任务**和子集合
- Flat 格式显示完整路径：`build.compile`, `build.docs.html`
- Nested 格式显示缩进层级

---

##### 场景2：`inv --list build`

**scoped_collection**：`build` 子集合

**list_flat() 输出**：
```
Available tasks:

  .compile (.c)             Compile the project
  .docs.html (.docs)        Build HTML docs
```

**list_nested() 输出**：
```
Available tasks (* denotes collection defaults):

  .compile (.c)             Compile the project
  .docs                     Documentation tasks
    .html*                  Build HTML docs
```

**与场景1的关键区别**：

| 维度 | `inv --list` | `inv --list build` |
|------|-------------|-------------------|
| 起始集合 | `root` | `build` |
| 显示范围 | 所有任务 | 只显示 `build` 及其子集合 |
| 任务名格式 | `build.compile`（完整路径） | `.compile`（前导点表示从当前命名空间开始） |
| 别名格式 | `build.c` | `.c` |

**为什么任务名前有 `.`？**

因为设置了 `self.list_root = "build"`，`_make_pairs()` 中的逻辑：

```python
# 在 _make_pairs 中
if ancestors or self.list_root:
    displayname = ".{}".format(displayname)  # 添加前导点
    aliases = [".{}".format(x) for x in aliases]
```
*invoke/program.py:845-847*

**前导点的含义**：表示这个任务名是**相对于当前命名空间**的，调用时需要使用完整路径（如 `inv build.compile`）或从对应命名空间调用。

---

##### 场景3：`inv --list build.docs`

**scoped_collection**：`docs` 子集合

**list_flat() 输出**：
```
Available tasks:

  .html (.build.docs)       Build HTML docs
```

**list_nested() 输出**：
```
Available tasks (* denotes collection defaults):

  .html*                    Build HTML docs
```

**与场景2的关键区别**：

| 维度 | `inv --list build` | `inv --list build.docs` |
|------|-------------------|------------------------|
| 起始集合 | `build` | `docs` |
| 显示范围 | `build` 的任务 + `docs` 的任务 | 只显示 `docs` 的任务 |
| 任务名格式 | `.compile`, `.docs.html` | `.html` |

---

##### 对比总结表

| 命令 | scoped_collection | 输出的任务 | 任务名格式 |
|------|-------------------|-------------|-----------|
| `inv --list` | `root` | `test`, `build.compile`, `build.docs.html` | `test`, `build.compile`, `build.docs.html` |
| `inv --list build` | `build` | `compile`, `docs.html` | `.compile`, `.docs.html` |
| `inv --list build.docs` | `docs` | `html` | `.html` |

---

#### 扩展：与其他参数的配合

`scoped_collection` 还会与其他参数配合使用：

##### 与 `--list-depth` 的配合

`list_depth` 控制输出树的**深度**，与 `scoped_collection` 配合使用时，深度是**从 `scoped_collection` 开始计算**的：

```python
truncate = self.list_depth and (len(ancestors) + 1) >= self.list_depth
if truncate:
    # 到达最大深度，显示摘要
    tallies = [
        "{} {}".format(len(getattr(subcoll, attr)), attr)
        for attr in ("tasks", "collections")
        if getattr(subcoll, attr)
    ]
    displayname += " [{}]".format(", ".join(tallies))
```
*invoke/program.py:870-881*

**示例对比**：

| 命令 | 输出 | 说明 |
|------|------|------|
| `inv --list --list-depth 1` | `test`, `build [2 tasks, 1 collections]` | 从 `root` 开始，深度1 |
| `inv --list build --list-depth 1` | `.compile`, `.docs [1 tasks, 0 collections]` | 从 `build` 开始，深度1 |

##### 与 `--list-format` 的配合

不同的输出格式都会使用 `scoped_collection` 作为起点：

```python
def list_flat(self) -> None:
    pairs = self._make_pairs(self.scoped_collection)  # 使用 scoped_collection
    self.display_with_columns(pairs=pairs)

def list_nested(self) -> None:
    pairs = self._make_pairs(self.scoped_collection)  # 使用 scoped_collection
    self.display_with_columns(pairs=pairs, extra="'*' denotes collection defaults")

def list_json(self) -> None:
    coll = self.scoped_collection  # 使用 scoped_collection
    data = coll.serialized()
    print(json.dumps(data))
```
*invoke/program.py:817-909*

---

#### 本节核心结论

| 问题 | 答案 |
|------|------|
| **默认值是什么？** | `self.scoped_collection = self.collection`（根集合），默认显示所有任务 |
| **如何切换到子集合？** | `inv --list <namespace>` 时，通过 `subcollection_from_path()` 递归查找子集合，然后更新 `self.scoped_collection = sub` |
| **对输出范围的影响？** | 所有输出方法都从 `scoped_collection` 开始递归，只显示该集合及其子集合的任务；任务名格式会添加前导点表示相对路径 |

### 4.3 支持的输出格式

| 格式 | 方法 | 说明 |
|------|------|------|
| `flat` | `list_flat()` | 扁平列表，显示完整路径 |
| `nested` | `list_nested()` | 树状结构，缩进显示层级 |
| `json` | `list_json()` | JSON 格式输出 |

### 4.4 核心生成方法：_make_pairs()

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

### 5.1 两种不同的合并场景

Invoke 中有两种不同的配置合并场景，使用不同的合并策略：

| 场景 | 方法 | 合并策略 | 嵌套字典行为 |
|------|------|----------|-------------|
| **同一个 Collection 内部** | `configure()` | `merge_dicts` | **递归合并** |
| **不同 Collection 层级之间** | `_task_with_merged_config()` | `dict(config, **ours)` | **浅合并（整体替换）** |

### 5.2 场景1：同一个 Collection 内部（递归合并）

当在**同一个 Collection** 上多次调用 `configure()` 时，使用 `merge_dicts` 进行**递归合并**：

```python
def configure(self, options: Dict[str, Any]) -> None:
    merge_dicts(self._configuration, options)
```
*invoke/collection.py:562-579*

`merge_dicts` 的实现（来自 `config.py`）：

```python
def merge_dicts(
    base: Dict[str, Any], updates: Dict[str, Any]
) -> Dict[str, Any]:
    """
    Recursively merge dict ``updates`` into dict ``base`` (mutating ``base``.)

    * Values which are themselves dicts will be recursed into.
    * Values which are a dict in one input and *not* a dict in the other input
      are irreconciliable and will generate an exception.
    """
    for key, value in (updates or {}).items():
        # Dict values whose keys also exist in 'base' -> recurse
        if key in base:
            if isinstance(value, dict):
                if isinstance(base[key], dict):
                    merge_dicts(base[key], value)  # 递归合并！
                else:
                    raise _merge_error(base[key], value)
            else:
                if isinstance(base[key], dict):
                    raise _merge_error(base[key], value)
                else:
                    base[key] = copy.copy(value)
        else:
            # New values get set anew
            if isinstance(value, dict):
                base[key] = copy_dict(value)
            else:
                base[key] = copy.copy(value)
    return base
```
*invoke/config.py:1168-1224*

**递归合并非例**：

```python
# 同一个 Collection 上多次调用 configure()
coll = Collection()
coll.configure({'db': {'host': 'localhost'}})
coll.configure({'db': {'port': 5432}})

# 结果：嵌套字典递归合并
# coll.configuration() = {'db': {'host': 'localhost', 'port': 5432}}
```

### 5.3 场景2：不同 Collection 层级之间（浅合并）

**⚠️ 关键发现**：当通过嵌套路径查找任务时，不同 Collection 层级之间的配置合并使用的是**浅合并**，嵌套字典会被**整体替换**！

```python
def _task_with_merged_config(
    self, coll: str, rest: str, ours: Dict[str, Any]
) -> Tuple[str, Dict[str, Any]]:
    # 递归获取子集合的任务和配置
    task, config = self.collections[coll].task_with_config(rest)
    # ⚠️ 浅合并：dict(config, **ours)
    return task, dict(config, **ours)
```
*invoke/collection.py:374-378*

**`dict(config, **ours)` 的行为**：
1. 首先复制 `config`（子集合配置）的所有键值对
2. 然后用 `ours`（父集合配置）中的键值对**覆盖**相同的键
3. **对于嵌套字典，这是整体替换，不是递归合并**

### 5.4 浅合并示例详解

假设有以下三层嵌套结构和配置：

```python
# 根集合
root = Collection()
root.configure({
    'log_level': 'info',
    'timeout': 30,
    'db': {'host': 'localhost'}  # 嵌套字典
})

# build 子集合
build = Collection('build')
build.configure({
    'log_level': 'debug',      # 覆盖父级的顶级字段
    'parallel': 4,              # 新增顶级字段
    'db': {'port': 5432}        # 嵌套字典（⚠️ 会被整体替换）
})

# docs 子集合（build 的子集合）
docs = Collection('docs')
docs.configure({
    'log_level': 'warning',    # 覆盖父级
    'db': {'user': 'admin'}    # 嵌套字典（⚠️ 会被整体替换）
})

# 任务
@task
def html(c):
    pass
docs.add_task(html)

# 组装结构
build.add_collection(docs)
root.add_collection(build)
```

**调用 `inv build.docs.html` 时的配置合并过程**：

```
步骤1：从最内层 docs 开始
  └── docs.configuration() = {'log_level': 'warning', 'db': {'user': 'admin'}}

步骤2：与 build 合并（浅合并）
  └── dict({'log_level': 'warning', 'db': {'user': 'admin'}},
           **{'log_level': 'debug', 'parallel': 4, 'db': {'port': 5432}})
  └── 结果：{
           'log_level': 'debug',      # 被 build 覆盖
           'parallel': 4,              # 来自 build（新增）
           'db': {'port': 5432}        # ⚠️ 整体替换！'user' 丢失了！
         }

步骤3：与 root 合并（浅合并）
  └── dict({'log_level': 'debug', 'parallel': 4, 'db': {'port': 5432}},
           **{'log_level': 'info', 'timeout': 30, 'db': {'host': 'localhost'}})
  └── 结果：{
           'log_level': 'info',       # 被 root 覆盖
           'parallel': 4,              # 来自 build（保留）
           'timeout': 30,              # 来自 root（新增）
           'db': {'host': 'localhost'} # ⚠️ 再次整体替换！'port' 也丢失了！
         }
```

### 5.5 合并结果分析

**最终配置（`build.docs.html`）**：

```python
{
    'log_level': 'info',          # 来自 root（覆盖了子集合的值）
    'parallel': 4,                # 来自 build（中间层的顶级字段保留）
    'timeout': 30,                # 来自 root
    'db': {'host': 'localhost'}   # 来自 root（嵌套字典只保留最外层）
}
```

**关键结论**：

| 字段类型 | 行为 | 示例 |
|----------|------|------|
| **顶级字段** | 所有层级的顶级字段都会被保留 | `parallel`（来自 build）、`timeout`（来自 root） |
| **嵌套字典** | 只保留**最外层**（父集合）的值，中间层丢失 | `db` 只保留 `{'host': 'localhost'}`，`db.port` 和 `db.user` 都丢失了 |
| **顶级字段冲突** | 父集合的值覆盖子集合的值 | `log_level` 最终为 `'info'`（来自 root） |

### 5.6 不同任务路径的配置对比

| 任务路径 | 合并后的配置 | 说明 |
|----------|-------------|------|
| `root_task` | `{'log_level': 'info', 'timeout': 30, 'db': {'host': 'localhost'}}` | 只使用 root 的配置 |
| `build.compile` | `{'log_level': 'info', 'timeout': 30, 'parallel': 4, 'db': {'host': 'localhost'}}` | `db.port` 丢失！ |
| `build.docs.html` | `{'log_level': 'info', 'timeout': 30, 'parallel': 4, 'db': {'host': 'localhost'}}` | `db.port` 和 `db.user` 都丢失！ |

### 5.7 配置合并流程图

```
调用: inv build.docs.html
        │
        ▼
┌─────────────────────────────────────────┐
│ root.task_with_config("build.docs.html")│
└─────────────────────────────────────────┘
        │
        ├── ours = root.configuration()
        │   {'log_level': 'info', 'timeout': 30, 'db': {'host': 'localhost'}}
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
        │   │   {'log_level': 'debug', 'parallel': 4, 'db': {'port': 5432}}
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
        │   │          {'log_level': 'warning', 'db': {'user': 'admin'}}
        │   │
        │   └── 浅合并: dict(docs_config, **build_ours)
        │          → {'log_level': 'debug', 'parallel': 4, 'db': {'port': 5432}}
        │          ⚠️ 'db.user' 丢失了！
        │
        └── 浅合并: dict(build_merged_config, **root_ours)
               → {'log_level': 'info', 'timeout': 30, 'parallel': 4, 'db': {'host': 'localhost'}}
               ⚠️ 'db.port' 也丢失了！
```

### 5.8 重要提示

**如果需要在不同层级之间共享嵌套配置**，应该：

1. **只在根集合定义嵌套配置**，子集合不要覆盖
2. **或者使用顶级字段**（非嵌套字典）来传递配置
3. **避免在不同层级定义相同键的嵌套字典**

**示例（推荐做法）**：

```python
# 推荐：只在根集合定义嵌套配置
root.configure({'db': {'host': 'localhost', 'port': 5432, 'user': 'admin'}})

# 子集合使用顶级字段，不覆盖嵌套字典
build.configure({'parallel': 4, 'log_level': 'debug'})
docs.configure({'log_level': 'warning'})

# 结果：所有层级都能访问完整的 db 配置
# build.docs.html 的配置 = {
#     'log_level': 'info',
#     'parallel': 4,
#     'timeout': 30,
#     'db': {'host': 'localhost', 'port': 5432, 'user': 'admin'}  # 完整！
# }
```

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

### 6.6 别名解析机制

**⚠️ 重要发现**：别名解析**不是单独的查找步骤**，而是由 `Lexicon`（继承自 `AliasDict`）的 `__getitem__` 和 `__contains__` 方法**透明完成**的。

从 `alias_dict.py` 的实现可以看到：

```python
def __getitem__(self, key):
    def single(d, target, value):
        return d[target]  # 返回真正的目标键的值
    
    def unaliased(d, key, value):
        return super(AliasDict, d).__getitem__(key)
    
    def multi(d, target, value):
        raise ValueError("Multi-target aliases can't be read.")
    
    return self._handle(key, None, single, multi, unaliased)

def __contains__(self, key):
    def single(d, target, value):
        return target in d  # 检查真正的目标键是否存在
    
    def multi(d, target, value):
        return all(subkey in self for subkey in self.aliases[key])
    
    def unaliased(d, key, value):
        return super(AliasDict, d).__contains__(key)
    
    return self._handle(key, None, single, multi, unaliased)

def _handle(self, key, value, single, multi, unaliased):
    # 首先检查 key 是否在 aliases 中
    if key in getattr(self, "aliases", {}):
        target = self.aliases[key]
        if isinstance(target, str):
            return single(self, target, value)  # 用目标键执行操作
        # ... 多目标处理
    else:
        return unaliased(self, key, value)
```
*invoke/vendor/lexicon/alias_dict.py:63-86*

**别名解析的时机**：

| 操作 | 别名解析发生的位置 | 行为 |
|------|-------------------|------|
| `self.tasks[name]` | `__getitem__` → `_handle` | 如果是别名，自动解析为目标键并返回其值 |
| `name in self.tasks` | `__contains__` → `_handle` | 如果是别名，检查目标键是否存在 |
| `self.tasks.get(name)` | 继承自 `dict`，**不考虑别名** | ⚠️ 注意：这个方法**不会**解析别名 |

**对查找流程的影响**：

当查找任务时：

```python
# 步骤：检查是否在 collections 中
if name in self.collections:  # __contains__ 会自动解析别名！
    return self._task_with_merged_config(name, "", ours)

# 步骤：检查是否在 tasks 中
return self.tasks[name]  # __getitem__ 会自动解析别名！
```

**关键结论**：别名解析在查找的**每一步**都已经透明发生，不需要单独的"别名解析"步骤。

### 6.7 默认任务的查找

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

### 6.8 查找优先级总结

| 优先级 | 查找类型 | 说明 | 别名解析时机 |
|--------|----------|------|-------------|
| 1 | 显式默认任务 | `name` 为空或 `None` → 检查 `self.default` | 不涉及 |
| 2 | 嵌套路径（带点号） | `name` 包含 `.` → 递归查找子集合 | 不涉及（在子集合中查找时会透明解析） |
| 3 | 子集合默认任务 | `name in self.collections` → 查找子集合的默认任务 | `__contains__` 自动解析 |
| 4 | 普通任务 | `self.tasks[name]` → 直接返回 | `__getitem__` 自动解析 |

**⚠️ 注意**：没有单独的"别名解析"步骤，因为 `Lexicon` 的 `__getitem__` 和 `__contains__` 已经在每次访问时自动处理了别名。

### 6.9 潜在冲突场景

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
