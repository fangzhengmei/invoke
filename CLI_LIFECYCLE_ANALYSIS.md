# Invoke CLI 完整生命周期分析报告

## 1. 概述

本文档深入分析 invoke CLI 的完整启动链路，包括：
- `add_collection()` 方法的完整机制（模块对象转换、名称冲突检测、默认子集合注册）
- `Program` 类的启动流程
- `Loader` 如何定位并加载 `tasks.py`
- `Parser` 如何将命令行参数解析为 `ParserContext` 并最终传入 `Executor.normalize()`

---

## 2. add_collection() 深度分析

### 2.1 核心实现

**关键代码位置**：`invoke/collection.py:285-324`

```python
def add_collection(
    self,
    coll: "Collection",
    name: Optional[str] = None,
    default: Optional[bool] = None,
) -> None:
```

### 2.2 完整执行流程

#### 阶段 1: 模块对象自动转换

**关键代码位置**：`invoke/collection.py:308-310`

```python
# Handle module-as-collection
if isinstance(coll, ModuleType):
    coll = Collection.from_module(coll)
```

**设计意图**：允许用户直接传入模块对象，无需手动调用 `Collection.from_module()`。

**示例**：
```python
from invoke import Collection
import my_tasks_module  # 一个包含 @task 装饰器的模块

# 方式 1: 手动转换
coll = Collection()
coll.add_collection(Collection.from_module(my_tasks_module), name="tasks")

# 方式 2: 自动转换（推荐）
coll = Collection()
coll.add_collection(my_tasks_module, name="tasks")  # 自动调用 from_module
```

**与 `_add_object()` 的联动**（`invoke/collection.py:116-124`）：

```python
def _add_object(self, obj: Any, name: Optional[str] = None) -> None:
    method: Callable
    if isinstance(obj, Task):
        method = self.add_task
    elif isinstance(obj, (Collection, ModuleType)):  # 支持 ModuleType
        method = self.add_collection
    else:
        raise TypeError("No idea how to insert {!r}!".format(type(obj)))
    method(obj, name=name)
```

这意味着在 `Collection` 构造函数中也可以直接传入模块：

```python
# 构造函数自动处理
coll = Collection(my_tasks_module)  # 自动调用 add_collection
```

#### 阶段 2: 确定集合名称

**关键代码位置**：`invoke/collection.py:311-314`

```python
# Ensure we have a name, or die trying
name = name or coll.name
if not name:
    raise ValueError("Non-root collections must have a name!")
```

**名称优先级**：
1. 显式传入的 `name` 参数
2. `coll.name` 属性（Collection 自身的 name）

**特殊情况**：
- 根 Collection（最外层）可以没有 name
- 子 Collection **必须**有 name，否则抛出 `ValueError`

#### 阶段 3: 应用名称转换

**关键代码位置**：`invoke/collection.py:315`

```python
name = self.transform(name)
```

**转换规则**（取决于 `auto_dash_names`）：
- `auto_dash_names=True`（默认）：`my_collection` → `my-collection`
- `auto_dash_names=False`：`my-collection` → `my_collection`

#### 阶段 4: 名称冲突检测

**关键代码位置**：`invoke/collection.py:316-319`

```python
# Test for conflict
if name in self.tasks:
    err = "Name conflict: this collection has a task named {!r} already"
    raise ValueError(err.format(name))
```

**冲突规则**：
- 子 Collection 的名称 **不能** 与已有的任务同名
- 注意：**只检查 `self.tasks`，不检查 `self.collections`**

**与 `add_task()` 的对称检查**：

`add_task()` 也有类似的冲突检测（`invoke/collection.py:275-277`）：
```python
if name in self.collections:
    err = "Name conflict: this collection has a sub-collection named {!r} already"
    raise ValueError(err.format(name))
```

**双向冲突检测总结**：

| 操作 | 检查对象 | 错误消息 |
|------|---------|---------|
| `add_task("foo")` | `self.collections` | "has a sub-collection named 'foo' already" |
| `add_collection("foo")` | `self.tasks` | "has a task named 'foo' already" |

**注意**：不检查同级 Collection 之间的同名冲突。

#### 阶段 5: 注册子集合

**关键代码位置**：`invoke/collection.py:320-321`

```python
# Insert
self.collections[name] = coll
```

注册到 `self.collections` Lexicon 中，支持后续的：
- 按名称访问：`coll.collections["sub"]`
- 路径解析：`coll["sub.task"]`

#### 阶段 6: 默认子集合注册（可选）

**关键代码位置**：`invoke/collection.py:322-324`

```python
if default:
    self._check_default_collision(name)
    self.default = name
```

**默认子集合的含义**：
- 当用户通过该 Collection 的名称访问时，返回该子 Collection 的默认任务
- 例如：如果 `docs` 是默认子集合，且 `docs` 的默认任务是 `build`，则 `coll[""]` 或 `coll["docs"]` 都返回 `build` 任务

**默认冲突检测**（`invoke/collection.py:326-329`）：

```python
def _check_default_collision(self, name: str) -> None:
    if self.default:
        msg = "'{}' cannot be the default because '{}' already is!"
        raise ValueError(msg.format(name, self.default))
```

**设计约束**：
- 一个 Collection 只能有**一个**默认任务/子集合
- 如果已有默认，再设置新的默认会抛出 `ValueError`

### 2.3 实际场景分析

#### 场景 1: 模块自动转换

```python
# tasks/build.py
from invoke import task

@task
def compile(c):
    print("Compiling...")

@task
def test(c):
    print("Testing...")
```

```python
# 主 tasks.py
from invoke import Collection
import tasks.build as build_module

ns = Collection()

# 方式 1: 显式转换
ns.add_collection(Collection.from_module(build_module), name="build")

# 方式 2: 自动转换（等价）
ns.add_collection(build_module, name="build")

# 方式 3: 构造函数直接传入
ns = Collection(build_module)  # 自动以模块名注册
```

#### 场景 2: 名称冲突

```python
from invoke import Collection, task

@task
def build(c):
    print("Building...")

@task
def docs_build(c):
    print("Building docs...")

docs = Collection("docs")
docs.add_task(docs_build, name="build")

ns = Collection(build)  # 注册名为 "build" 的任务

# 尝试注册同名的子 Collection
try:
    ns.add_collection(docs, name="build")  # 冲突！
except ValueError as e:
    print(e)  # "Name conflict: this collection has a task named 'build' already"
```

#### 场景 3: 默认子集合

```python
from invoke import Collection, task

# 子 Collection
@task(default=True)
def html(c):
    print("Building HTML docs...")

@task
def pdf(c):
    print("Building PDF docs...")

docs = Collection("docs")
docs.add_task(html)  # 默认任务
docs.add_task(pdf)

# 主 Collection
@task
def deploy(c):
    print("Deploying...")

ns = Collection(deploy)
ns.add_collection(docs, default=True)  # 设置为默认

# 访问方式
ns[""]           # → html 任务（通过默认子集合）
ns["docs"]       # → html 任务（docs 的默认任务）
ns["docs.html"]  # → html 任务（显式路径）
ns["docs.pdf"]   # → pdf 任务
```

**执行时的效果**：
```bash
# 不带参数调用 invoke → 执行默认子集合的默认任务
invoke
# 等价于
invoke docs
# 等价于
invoke docs.html
```

### 2.4 add_collection() vs add_task() 对比

| 特性 | add_collection() | add_task() |
|------|-----------------|------------|
| 支持的入参类型 | Collection, ModuleType | Task |
| 模块自动转换 | ✅ `Collection.from_module()` | ❌ |
| 名称来源 | 参数 > coll.name | 参数 > task.name > func_name > \_\_name\_\_ |
| 冲突检测 | 检查 `self.tasks` | 检查 `self.collections` |
| 注册位置 | `self.collections[name]` | `self.tasks[name]` |
| 支持默认设置 | ✅ `default=True` | ✅ `default=True` |
| 别名支持 | ❌（子 Collection 无别名） | ✅ `aliases=(...)` |

---

## 3. Program CLI 启动流程

### 3.1 核心入口

**关键代码位置**：`invoke/program.py:355-422`

```python
def run(self, argv: Optional[List[str]] = None, exit: bool = True) -> None:
    try:
        self.create_config()           # 1. 创建初始配置
        self.parse_core(argv)          # 2. 解析核心参数
        self.parse_collection()        # 3. 加载任务集合
        self.parse_tasks()             # 4. 解析任务参数
        self.parse_cleanup()           # 5. 解析后清理（--help, --list 等）
        self.update_config()           # 6. 更新配置
        self.execute()                 # 7. 执行任务
    except (UnexpectedExit, Exit, ParseError) as e:
        # 异常处理
        if exit:
            sys.exit(code)
    except KeyboardInterrupt:
        sys.exit(1)
```

### 3.2 阶段 1: create_config() - 创建初始配置

**关键代码位置**：`invoke/program.py:287-300`

```python
def create_config(self) -> None:
    self.config = self.config_class()
```

**创建的配置包含**（来自 `Config` 默认值）：
- 系统默认配置
- 还未加载：项目配置、运行时配置、CLI 覆盖

### 3.3 阶段 2: parse_core() - 解析核心参数

**关键代码位置**：`invoke/program.py:424-452`

```python
def parse_core(self, argv: Optional[List[str]]) -> None:
    self.normalize_argv(argv)                    # 标准化 argv
    self.parse_core_args()                       # 解析核心参数
    # 设置解释器字节码标志
    sys.dont_write_bytecode = not self.args["write-pyc"].value
    # 启用调试日志
    if self.args.debug.value:
        enable_logging()
    # 短路：--version
    if self.args.version.value:
        self.print_version()
        raise Exit
    # 短路：--print-completion-script
    if self.args["print-completion-script"].value:
        print_completion_script(...)
        raise Exit
```

#### parse_core_args() 详解

**关键代码位置**：`invoke/program.py:688-700`

```python
def parse_core_args(self) -> None:
    debug("Parsing initial context (core args)")
    parser = Parser(initial=self.initial_context, ignore_unknown=True)
    self.core = parser.parse_argv(self.argv[1:])
    debug("Core-args parse result: {!r} & unparsed: {!r}".format(
        self.core, self.core.unparsed
    ))
```

**核心参数上下文**（`initial_context`）：

从 `core_args()` + `task_args()` 构建：

**core_args()**（`invoke/program.py:48-150`）：
- `--command-timeout, -T`：全局命令超时
- `--complete`：打印补全候选
- `--config, -f`：运行时配置文件
- `--debug, -d`：启用调试
- `--dry, -R`：干运行
- `--echo, -e`：回显命令
- `--help, -h`：显示帮助
- `--hide`：设置默认隐藏
- `--list, -l`：列出任务
- `--list-depth, -D`：列表深度
- `--list-format, -F`：列表格式
- `--print-completion-script`：打印补全脚本
- `--prompt-for-sudo-password`：提示 sudo 密码
- `--pty, -p`：使用 PTY
- `--version, -V`：显示版本
- `--warn-only, -w`：仅警告
- `--write-pyc`：启用 .pyc 创建

**task_args()**（`invoke/program.py:152-179`）：
- `--collection, -c`：指定集合名称
- `--no-dedupe`：禁用去重
- `--search-root, -r`：搜索根目录

**解析策略**：
- `ignore_unknown=True`：未知参数不报错，存入 `.unparsed`
- 这允许核心参数和任务参数混合出现

### 3.4 阶段 3: parse_collection() - 加载任务集合

**关键代码位置**：`invoke/program.py:454-489`

```python
def parse_collection(self) -> None:
    # 如果有预定义的 namespace，直接使用
    if self.namespace is not None:
        debug("Program was given default namespace, not loading collection")
        self.collection = self.namespace
    else:
        # 没有预定义 namespace，需要从文件系统加载
        # 短路：如果没有 bundled namespace 且 --help 直接给出
        if self.args.help.value is True:
            self.print_help()
            raise Exit
        # 加载集合
        self.load_collection()
    
    # 初始化列表相关属性
    self.list_root: Optional[str] = None
    self.list_depth: Optional[int] = None
    self.list_format = "flat"
    self.scoped_collection = self.collection
```

#### load_collection() 详解

**关键代码位置**：`invoke/program.py:702-729`

```python
def load_collection(self) -> None:
    # 获取搜索根目录（默认 CWD）
    start = self.args["search-root"].value
    # 创建 Loader
    loader = self.loader_class(
        config=self.config, start=start
    )
    # 获取集合名称（默认 "tasks"）
    coll_name = self.args.collection.value
    try:
        # 加载模块
        module, parent = loader.load(coll_name)
        # 设置项目配置位置并加载
        self.config.set_project_location(parent)
        self.config.load_project()
        # 创建 Collection
        self.collection = Collection.from_module(
            module,
            loaded_from=parent,
            auto_dash_names=self.config.tasks.auto_dash_names,
        )
    except CollectionNotFound as e:
        raise Exit("Can't find any collection named {!r}!".format(e.name))
```

**关键点**：
1. 使用 `FilesystemLoader` 从文件系统定位 `tasks.py`
2. 加载后设置项目配置位置
3. 通过 `Collection.from_module()` 创建集合
4. 支持 `--collection` 指定非默认名称

### 3.5 阶段 4: parse_tasks() - 解析任务参数

**关键代码位置**：`invoke/program.py:750-772`

```python
def parse_tasks(self) -> None:
    # 创建 Parser，包含所有任务的上下文
    self.parser = self._make_parser()
    debug("Parsing tasks against {!r}".format(self.collection))
    # 解析 core.unparsed 中剩余的参数
    result = self.parser.parse_argv(self.core.unparsed)
    # 第一个元素是 core_via_tasks（任务参数中出现的核心参数）
    self.core_via_tasks = result.pop(0)
    # 更新 core context
    self._update_core_context(
        context=self.core[0], new_args=self.core_via_tasks.args
    )
    # 剩余的是任务上下文列表
    self.tasks = result
    debug("Resulting task contexts: {!r}".format(self.tasks))
```

#### _make_parser() 详解

**关键代码位置**：`invoke/program.py:742-748`

```python
def _make_parser(self) -> Parser:
    return Parser(
        initial=self.initial_context,
        contexts=self.collection.to_contexts(
            ignore_unknown_help=self.config.tasks.ignore_unknown_help
        ),
    )
```

**Collection.to_contexts()**（`invoke/collection.py:423-449`）：

```python
def to_contexts(self, ignore_unknown_help: Optional[bool] = None) -> List[ParserContext]:
    result = []
    for primary, aliases in self.task_names.items():
        task = self[primary]
        result.append(
            ParserContext(
                name=primary,
                aliases=aliases,
                args=task.get_arguments(
                    ignore_unknown_help=ignore_unknown_help
                ),
            )
        )
    return result
```

**关键点**：
- `task_names` 展开所有嵌套任务为扁平路径（如 `build.docs.html`）
- 每个任务创建一个 `ParserContext`，包含：
  - `name`：任务主名
  - `aliases`：任务别名列表
  - `args`：任务参数列表（来自 `@task(arguments=[...])` 或函数签名）

### 3.6 阶段 5: parse_cleanup() - 解析后清理

**关键代码位置**：`invoke/program.py:490-557`

```python
def parse_cleanup(self) -> None:
    halp = self.args.help.value
    
    # 情况 1: --help 无参数（仅当 bundled namespace）
    if halp is True:
        self.print_help()
        raise Exit
    
    # 情况 2: --help <taskname>
    if halp:
        if halp in self.parser.contexts:
            self.print_task_help(halp)
            raise Exit
        else:
            raise ParseError("No idea what '{}' is!".format(halp))
    
    # 情况 3: --list
    list_root = self.args.list.value
    self.list_format = self.args["list-format"].value
    self.list_depth = self.args["list-depth"].value
    if list_root:
        # --list some-root
        if isinstance(list_root, str):
            self.list_root = list_root
            try:
                sub = self.collection.subcollection_from_path(list_root)
                self.scoped_collection = sub
            except KeyError:
                raise Exit("Sub-collection '{}' not found!".format(list_root))
        self.list_tasks()
        raise Exit
    
    # 情况 4: --complete
    if self.args.complete.value:
        complete(...)
        return
    
    # 情况 5: 无任务且无默认
    if not self.tasks and not self.collection.default:
        self.no_tasks_given()  # 打印帮助并退出
```

### 3.7 阶段 6: update_config() - 更新配置

**关键代码位置**：`invoke/program.py:302-353`

```python
def update_config(self, merge: bool = True) -> None:
    # 从解析结果构建 overrides
    run = {}
    if self.args["warn-only"].value:
        run["warn"] = True
    if self.args.pty.value:
        run["pty"] = True
    if self.args.hide.value:
        run["hide"] = self.args.hide.value
    if self.args.echo.value:
        run["echo"] = True
    if self.args.dry.value:
        run["dry"] = True
    
    tasks = {}
    if "no-dedupe" in self.args and self.args["no-dedupe"].value:
        tasks["dedupe"] = False
    
    timeouts = {}
    command = self.args["command-timeout"].value
    if command:
        timeouts["command"] = command
    
    sudo = {}
    if self.args["prompt-for-sudo-password"].value:
        prompt = "Desired 'sudo.password' config value: "
        sudo["password"] = getpass.getpass(prompt)
    
    # 加载到 config 的 overrides 层级
    overrides = dict(run=run, tasks=tasks, sudo=sudo, timeouts=timeouts)
    self.config.load_overrides(overrides, merge=False)
    
    # 加载运行时配置文件
    runtime_path = self.args.config.value
    if runtime_path is None:
        runtime_path = os.environ.get("INVOKE_RUNTIME_CONFIG", None)
    self.config.set_runtime_path(runtime_path)
    self.config.load_runtime(merge=False)
    
    # 最终合并
    if merge:
        self.config.merge()
```

### 3.8 阶段 7: execute() - 执行任务

**关键代码位置**：`invoke/program.py:559-583`

```python
def execute(self) -> None:
    klass = self.executor_class
    # 支持配置中指定的 executor_class
    config_path = self.config.tasks.executor_class
    if config_path is not None:
        # 动态导入
        module_path, _, class_name = config_path.rpartition(".")
        module = import_module(module_path)
        klass = getattr(module, class_name)
    
    # 创建 Executor 并执行
    executor = klass(self.collection, self.config, self.core)
    executor.execute(*self.tasks)
```

**关键链接**：
- `self.tasks` 是 `ParserContext` 列表
- 传入 `Executor.execute(*self.tasks)`
- 在 `Executor.normalize()` 中，`ParserContext` 被转换为 `Call`：
  ```python
  elif isinstance(task, ParserContext):
      name = task.name
      kwargs = task.as_kwargs
  c = Call(self.collection[name], kwargs=kwargs, called_as=name)
  ```

### 3.9 Program 启动流程图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    用户执行: invoke build --verbose test                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Program.run(argv)                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. create_config()                                                    │   │
│  │    - self.config = Config()  (初始配置，仅默认值)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. parse_core(argv)                                                   │   │
│  │    - normalize_argv() → 标准化为列表                                  │   │
│  │    - parse_core_args()                                                │   │
│  │      ├── Parser(initial=initial_context, ignore_unknown=True)        │   │
│  │      ├── parse_argv(["--verbose"])                                    │   │
│  │      └── 结果: self.core = ParseResult([core_context])               │   │
│  │           self.core.unparsed = ["build", "test"] (剩余任务)          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. parse_collection()                                                 │   │
│  │    - 检查是否有预定义 self.namespace                                   │   │
│  │    - 无则调用 load_collection()                                        │   │
│  │      ├── FilesystemLoader.find("tasks")                               │   │
│  │      │   └── 向上搜索目录，找到 tasks.py 或 tasks/__init__.py         │   │
│  │      ├── loader.load() → 导入模块                                      │   │
│  │      ├── config.set_project_location()                                 │   │
│  │      ├── config.load_project()                                         │   │
│  │      └── Collection.from_module(module)                                │   │
│  │           → 创建包含所有任务的 Collection                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 4. parse_tasks()                                                      │   │
│  │    - _make_parser()                                                   │   │
│  │      ├── collection.to_contexts()                                     │   │
│  │      │   └── 为每个任务创建 ParserContext:                             │   │
│  │      │       - name="build", aliases=[...], args=[--verbose, ...]    │   │
│  │      │       - name="test", aliases=[...], args=[...]                │   │
│  │      └── Parser(initial=initial_context, contexts=task_contexts)      │   │
│  │    - parser.parse_argv(self.core.unparsed)                            │   │
│  │      └── 解析 ["build", "--verbose", "test"]                          │   │
│  │    - result = [core_via_tasks, build_context, test_context]           │   │
│  │    - self.core_via_tasks = result.pop(0)                              │   │
│  │    - self.tasks = [build_context, test_context] (ParserContext 列表)  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 5. parse_cleanup()                                                    │   │
│  │    - 检查 --help, --list, --complete 等短路选项                       │   │
│  │    - 无任务且无默认 → 打印帮助并退出                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 6. update_config()                                                     │   │
│  │    - 从解析结果构建 overrides: run, tasks, sudo, timeouts             │   │
│  │    - config.load_overrides(overrides)                                  │   │
│  │    - config.set_runtime_path()                                          │   │
│  │    - config.load_runtime()                                              │   │
│  │    - config.merge()  (最终合并所有层级)                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
                                      ▼
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 7. execute()                                                          │   │
│  │    - executor = Executor(collection, config, core)                    │   │
│  │    - executor.execute(*self.tasks)                                     │   │
│  │      └── 传入 self.tasks = [build_context, test_context]               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Executor.execute() 接管                                   │
│                    (详见 R1/R3 报告)                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Loader 定位与加载机制

### 4.1 Loader 类层次

```
Loader (抽象基类)
  └── FilesystemLoader (文件系统加载器)
```

### 4.2 FilesystemLoader 初始化

**关键代码位置**：`invoke/loader.py:112-121`

```python
def __init__(self, start: Optional[str] = None, **kwargs: Any) -> None:
    super().__init__(**kwargs)
    if start is None:
        start = self.config.tasks.search_root
    self._start = start

@property
def start(self) -> str:
    # Lazily determine default CWD if configured value is falsey
    return self._start or os.getcwd()
```

**搜索起点**：
- 默认从当前工作目录（CWD）开始
- 可通过 `--search-root` 或 `tasks.search_root` 配置修改

### 4.3 find() - 定位模块

**关键代码位置**：`invoke/loader.py:123-154`

```python
def find(self, name: str) -> Optional[ModuleSpec]:
    debug("FilesystemLoader find starting at {!r}".format(self.start))
    spec = None
    module = "{}.py".format(name)  # 如 "tasks.py"
    paths = self.start.split(os.sep)
    
    # 从当前目录向上遍历到根目录
    for x in reversed(range(len(paths) + 1)):
        path = os.sep.join(paths[0:x])
        
        # 情况 1: 单文件模块 (tasks.py)
        if module in os.listdir(path):
            spec = spec_from_file_location(
                name, os.path.join(path, module)
            )
            break
        
        # 情况 2: 包模块 (tasks/__init__.py)
        elif name in os.listdir(path) and os.path.exists(
            os.path.join(path, name, "__init__.py")
        ):
            basepath = os.path.join(path, name)
            spec = spec_from_file_location(
                name,
                os.path.join(basepath, "__init__.py"),
                submodule_search_locations=[basepath],
            )
            break
    
    if spec:
        debug("Found module: {!r}".format(spec))
        return spec
    
    # 未找到
    raise CollectionNotFound(name=name, start=self.start)
```

**搜索策略**：
1. 从 `start` 目录开始
2. 逐级向上遍历父目录
3. 在每个目录检查：
   - 是否存在 `{name}.py`（如 `tasks.py`）
   - 或是否存在 `{name}/__init__.py`（如 `tasks/__init__.py`）
4. 找到则创建 `ModuleSpec`，否则抛出 `CollectionNotFound`

**示例**（搜索 `tasks`）：
```
当前目录: /home/user/project/src

搜索顺序:
1. /home/user/project/src/
   - 检查 tasks.py?
   - 检查 tasks/__init__.py?
   
2. /home/user/project/
   - 找到 tasks.py ✓
   - 返回 spec
```

### 4.4 load() - 加载模块

**关键代码位置**：`invoke/loader.py:49-96`

```python
def load(self, name: Optional[str] = None) -> Tuple[ModuleType, str]:
    # 默认名称来自配置
    if name is None:
        name = self.config.tasks.collection_name
    
    # 定位模块
    spec = self.find(name)
    
    if spec and spec.loader and spec.origin:
        # source_file: tasks.py 或 tasks/__init__.py
        source_file = Path(spec.origin)
        
        # enclosing_dir: 包含模块的目录
        # - tasks.py 的情况: tasks.py 所在的目录
        # - tasks/__init__.py 的情况: tasks/ 目录
        enclosing_dir = source_file.parent
        
        # module_parent: 项目根目录（用于查找配置文件）
        # - tasks.py 的情况: 与 enclosing_dir 相同
        # - tasks/__init__.py 的情况: enclosing_dir 的父目录
        module_parent = enclosing_dir
        if spec.parent:  # 是包的情况
            module_parent = module_parent.parent
        
        # 将 enclosing_dir 加入 sys.path，支持相对导入
        enclosing_str = str(enclosing_dir)
        if enclosing_str not in sys.path:
            sys.path.insert(0, enclosing_str)
        
        # 实际导入模块
        module = module_from_spec(spec)
        sys.modules[spec.name] = module  # 支持 'from . import xxx'
        spec.loader.exec_module(module)
        
        # 返回模块和项目根目录
        return module, str(module_parent)
    
    raise ImportError
```

**关键路径区分**：

| 模块形式 | source_file | enclosing_dir | module_parent |
|---------|-------------|---------------|---------------|
| `tasks.py` | `/project/tasks.py` | `/project` | `/project` |
| `tasks/__init__.py` | `/project/tasks/__init__.py` | `/project/tasks` | `/project` |

**为什么需要 module_parent？**
- 用于设置项目配置文件的查找位置
- 在 `Program.load_collection()` 中：
  ```python
  self.config.set_project_location(parent)
  self.config.load_project()
  ```
- 这确保了 `invoke.yaml` 等配置文件从正确的目录加载

### 4.5 Loader 流程图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FilesystemLoader.load("tasks")                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. find("tasks") - 定位模块                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  起点: self.start (默认 CWD: /home/user/project/src)                        │
│                                                                              │
│  遍历路径 (reversed):                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ x = 4: path = "/home/user/project/src"                                 │  │
│  │      - 检查 tasks.py?                                                   │  │
│  │      - 检查 tasks/__init__.py?                                         │  │
│  │      - 未找到 → 继续                                                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ x = 3: path = "/home/user/project"                                     │  │
│  │      - 检查 tasks.py? → 找到!                                          │  │
│  │      - spec_from_file_location("tasks", "/home/user/project/tasks.py") │  │
│  │      - break 退出循环                                                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  返回: spec = ModuleSpec(origin="/home/user/project/tasks.py")             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 计算关键路径                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  source_file = Path(spec.origin)                                            │
│              = Path("/home/user/project/tasks.py")                          │
│                                                                              │
│  enclosing_dir = source_file.parent                                         │
│                = Path("/home/user/project")                                 │
│                                                                              │
│  module_parent = enclosing_dir                                              │
│                = Path("/home/user/project")                                 │
│                (如果是包 tasks/__init__.py，则多向上一层)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 设置 sys.path，支持相对导入                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  enclosing_str = "/home/user/project"                                       │
│  if enclosing_str not in sys.path:                                          │
│      sys.path.insert(0, enclosing_str)                                      │
│                                                                              │
│  现在 tasks.py 中可以:                                                       │
│  - from . import utils              ✓                                       │
│  - from .submodule import task    ✓                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 实际导入模块                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  module = module_from_spec(spec)                                            │
│  sys.modules[spec.name] = module  # 支持相对导入                            │
│  spec.loader.exec_module(module)  # 执行模块代码                            │
│                                                                              │
│  此时:                                                                        │
│  - @task 装饰器执行，Task 对象创建                                          │
│  - 如果有 ns = Collection(...)，也会被创建                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 返回                                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  return module, str(module_parent)                                           │
│         │              │                                                      │
│         │              └── "/home/user/project"                            │
│         │                        (用于 config.set_project_location)          │
│         │                                                                     │
│         └── 已加载的模块对象（包含所有 Task 定义）                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Parser 解析机制

### 5.1 核心组件

```
Parser (解析器主体)
  ├── initial: ParserContext (初始上下文，核心参数)
  ├── contexts: Lexicon[ParserContext] (任务上下文)
  └── ignore_unknown: bool (未知参数处理策略)

ParseMachine (状态机，实际解析逻辑)
  └── 基于 fluidity 状态机库

ParserContext (解析上下文)
  ├── name: str (上下文名称，如 "build")
  ├── aliases: List[str] (别名列表)
  ├── args: Lexicon[Argument] (参数定义)
  ├── positional_args: List[Argument] (位置参数)
  ├── flags: Lexicon[Argument] (标志形式访问)
  └── inverse_flags: Dict[str, str] (反向标志，如 --no-xxx)

Argument (单个参数定义)
  ├── names: Tuple[str, ...] (参数名，如 ("verbose", "v"))
  ├── kind: Type (类型，如 bool, str, int, list)
  ├── default: Any (默认值)
  ├── help: str (帮助文本)
  ├── positional: bool (是否位置参数)
  ├── optional: bool (是否可选值)
  └── ...
```

### 5.2 Parser.parse_argv() 流程

**关键代码位置**：`invoke/parser/parser.py:90-202`

```python
def parse_argv(self, argv: List[str]) -> ParseResult:
    # 创建状态机
    machine = ParseMachine(
        initial=self.initial,
        contexts=self.contexts,
        ignore_unknown=self.ignore_unknown,
    )
    
    # 分离 -- 之后的 remainder
    try:
        ddash = argv.index("--")
    except ValueError:
        ddash = len(argv)
    body = argv[:ddash]
    remainder = argv[ddash:][1:]  # 跳过 -- 本身
    
    # 逐个处理 token
    for index, token in enumerate(body):
        # 处理非空格分隔的形式:
        # - --foo=bar → 拆分为 --foo 和 bar
        # - -qv → 拆分为 -q 和 -v（或处理为多标志）
        mutations = []
        orig = token
        
        if is_flag(token) and not machine.result.unparsed:
            # 情况 1: --foo=bar 形式
            if "=" in token:
                token, _, value = token.partition("=")
                mutations.append((index + 1, value))
            
            # 情况 2: -qv 多短标志形式
            elif not is_long_flag(token) and len(token) > 2:
                # 检查是多布尔标志还是带值的标志
                # 例如: -fvalue 可能是 -f + value
                # 或者: -qv 是 -q + -v
                # ... 复杂的拆分逻辑
        
        # 应用 mutations
        for index, value in mutations:
            body.insert(index, value)
        
        # 交给状态机处理
        machine.handle(token)
    
    # 完成解析
    machine.finish()
    result = machine.result
    result.remainder = " ".join(remainder)
    return result
```

### 5.3 ParseMachine 状态机

**关键代码位置**：`invoke/parser/parser.py:205-455`

#### 状态定义

```python
class ParseMachine(StateMachine):
    initial_state = "context"
    
    # 状态
    state("context", enter=["complete_flag", "complete_context"])
    state("unknown", enter=["complete_flag", "complete_context"])
    state("end", enter=["complete_flag", "complete_context"])
    
    # 转换
    transition(from_=("context", "unknown"), event="finish", to="end")
    transition(
        from_="context",
        event="see_context",
        action="switch_to_context",
        to="context",
    )
    transition(
        from_=("context", "unknown"),
        event="see_unknown",
        action="store_only",
        to="unknown",
    )
```

**状态说明**：

| 状态 | 含义 |
|------|------|
| `context` | 正常解析状态，可以识别上下文和标志 |
| `unknown` | 遇到未知输入，后续所有 token 存入 `unparsed` |
| `end` | 解析完成 |

#### handle() 核心方法

**关键代码位置**：`invoke/parser/parser.py:272-328`

```python
def handle(self, token: str) -> None:
    # 如果已经在 unknown 状态，直接存储
    if self.current_state == "unknown":
        self.see_unknown(token)
        return
    
    # 情况 1: 当前上下文的标志
    if self.context and token in self.context.flags:
        self.switch_to_flag(token)
    
    # 情况 2: 当前上下文的反向标志
    elif self.context and token in self.context.inverse_flags:
        self.switch_to_flag(token, inverse=True)
    
    # 情况 3: 当前标志需要的值
    elif self.waiting_for_flag_value:
        self.see_value(token)
    
    # 情况 4: 位置参数
    elif self.context and self.context.missing_positional_args:
        self.see_positional_arg(token)
    
    # 情况 5: 新的上下文名称（任务名）
    elif token in self.contexts:
        self.see_context(token)
    
    # 情况 6: 初始上下文的标志（在任务上下文中出现的核心参数）
    elif self.initial and token in self.initial.flags:
        # 特殊处理 --help
        if flag.name == "help":
            flag.value = self.context.name  # 保存任务名作为 help 的值
        else:
            self.switch_to_flag(token)
    
    # 情况 7: 未知
    else:
        if not self.ignore_unknown:
            self.error("No idea what {!r} is!".format(token))
        else:
            self.see_unknown(token)
```

### 5.4 完整解析示例

**命令**：
```bash
invoke build --verbose test --name=mytest
```

**argv**（已去掉程序名）：
```python
["build", "--verbose", "test", "--name=mytest"]
```

**上下文定义**：
- `initial_context`: 核心参数（如 `--help`, `--version` 等）
- `contexts["build"]`: build 任务的上下文，包含 `--verbose` 标志
- `contexts["test"]`: test 任务的上下文，包含 `--name` 参数

**解析流程**：

```
初始状态:
  - self.context = initial_context (核心参数)
  - self.current_state = "context"
  - self.result = ParseResult([initial_context])

处理 token "build":
  1. 检查: "build" in self.contexts? → Yes!
  2. see_context("build")
     - switch_to_context("build")
       - 完成当前上下文 (complete_context)
       - self.context = copy.deepcopy(self.contexts["build"])
       - 新上下文加入 result
  3. self.result = [initial_context, build_context]

处理 token "--verbose":
  1. 检查: "--verbose" in self.context.flags? → Yes!
  2. switch_to_flag("--verbose")
     - self.flag = build_context.flags["--verbose"]
     - 如果是 bool 类型，立即设置 flag.value = True

处理 token "test":
  1. 检查: "test" in self.contexts? → Yes!
  2. see_context("test")
     - complete_flag() 完成当前标志
     - complete_context() 完成 build 上下文
     - switch_to_context("test")
       - self.context = copy.deepcopy(self.contexts["test"])
  3. self.result = [initial_context, build_context, test_context]

处理 token "--name=mytest":
  1. 拆分: "--name=mytest" → "--name" 和 "mytest"
  2. 处理 "--name":
     - switch_to_flag("--name")
     - self.flag = test_context.flags["--name"]
  3. 处理 "mytest":
     - waiting_for_flag_value? → Yes!
     - see_value("mytest")
     - self.flag.value = "mytest"

finish():
  - complete_flag()
  - complete_context()
  - self.current_state = "end"

最终 result:
  - ParseResult([initial_context, build_context, test_context])
  - 其中:
    - initial_context.args: 空（没有核心参数）
    - build_context.args: {"verbose": True}
    - test_context.args: {"name": "mytest"}
```

### 5.5 解析结果到 Executor 的传递

**关键代码位置**：`invoke/program.py:750-772`

```python
def parse_tasks(self) -> None:
    # ...
    result = self.parser.parse_argv(self.core.unparsed)
    # 第一个是 core_via_tasks（任务中出现的核心参数）
    self.core_via_tasks = result.pop(0)
    # 剩余的是任务上下文
    self.tasks = result
```

**然后传入 Executor**（`invoke/program.py:582-583`）：

```python
executor = klass(self.collection, self.config, self.core)
executor.execute(*self.tasks)  # *self.tasks 解包 ParserContext 列表
```

**在 Executor.normalize() 中转换**（`invoke/executor.py:151-179`）：

```python
def normalize(self, tasks) -> List["Call"]:
    calls = []
    for task in tasks:
        # ...
        elif isinstance(task, ParserContext):
            name = task.name
            kwargs = task.as_kwargs  # 关键: .as_kwargs 转换为 dict
        
        # 通过 Collection 定位 Task
        c = Call(self.collection[name], kwargs=kwargs, called_as=name)
        calls.append(c)
    # ...
    return calls
```

**ParserContext.as_kwargs**（`invoke/parser/context.py:162-174`）：

```python
@property
def as_kwargs(self) -> Dict[str, Any]:
    ret = {}
    for arg in self.args.values():
        ret[arg.name] = arg.value  # 使用 arg.name（如 "verbose" 而非 "--verbose"）
    return ret
```

### 5.6 解析链路图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           CLI 输入: invoke build --verbose test --name=mytest               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. Program.parse_core_args()                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  parser = Parser(initial=initial_context, ignore_unknown=True)              │
│  self.core = parser.parse_argv(["build", "--verbose", "test", "--name=mytest"])│
│                                                                              │
│  由于 ignore_unknown=True:                                                   │
│    - initial_context 只包含核心参数（--help, --version 等）                 │
│    - "build", "test" 等不是核心参数，被视为 unknown                          │
│    - 结果: self.core.unparsed = ["build", "--verbose", "test", "--name=mytest"]│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. Program.parse_tasks()                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  a. 创建包含任务上下文的 Parser                                               │
│     self.parser = Parser(                                                    │
│         initial=self.initial_context,                                       │
│         contexts=self.collection.to_contexts()                              │
│                  └── [build_context, test_context, ...]                      │
│     )                                                                        │
│                                                                              │
│  b. 解析剩余参数                                                              │
│     result = self.parser.parse_argv(self.core.unparsed)                     │
│            = [initial_context, build_context, test_context]                 │
│                                                                              │
│  c. 分离 core_via_tasks 和任务                                               │
│     self.core_via_tasks = result.pop(0)  # initial_context                 │
│     self.tasks = result  # [build_context, test_context]                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. Program.execute()                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  executor = Executor(self.collection, self.config, self.core)               │
│  executor.execute(*self.tasks)                                               │
│     │              │                                                         │
│     │              └── *[build_context, test_context]                        │
│     │                                                                         │
│     ▼                                                                         │
│  Executor.execute(build_context, test_context)                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. Executor.normalize()                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  for task in [build_context, test_context]:                                 │
│      if isinstance(task, ParserContext):                                    │
│          name = task.name          # "build" / "test"                       │
│          kwargs = task.as_kwargs  # {"verbose": True} / {"name": "mytest"}  │
│                                                                              │
│      # 定位 Task 实例                                                         │
│      task_obj = self.collection[name]                                        │
│                                                                              │
│      # 包装为 Call                                                            │
│      c = Call(task_obj, kwargs=kwargs, called_as=name)                      │
│      calls.append(c)                                                         │
│                                                                              │
│  结果: calls = [                                                             │
│      Call(build_task, kwargs={"verbose": True}, called_as="build"),         │
│      Call(test_task, kwargs={"name": "mytest"}, called_as="test")          │
│  ]                                                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    后续: expand_calls → dedupe → 执行循环                   │
│                    (详见 R1 报告)                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 完整 CLI 生命周期总结

### 6.1 端到端链路

```
用户输入: invoke build --verbose test --name=mytest
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  1. sys.argv = ["invoke", "build", "--verbose", "test", "--name=mytest"]  │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  2. Program.run(argv)                                                      │
│     ├─ create_config()  → Config()                                         │
│     ├─ parse_core()                                                        │
│     │   ├─ normalize_argv(argv)                                            │
│     │   └─ parse_core_args()                                                │
│     │       └─ Parser(ignore_unknown=True)                                 │
│     │           └─ self.core = ParseResult([core_context])                │
│     │              self.core.unparsed = ["build", "--verbose", ...]       │
│     ├─ parse_collection()                                                   │
│     │   └─ load_collection()                                                │
│     │       ├─ FilesystemLoader.find("tasks")                              │
│     │       │   └─ 向上搜索找到 tasks.py                                    │
│     │       ├─ loader.load() → (module, parent)                           │
│     │       ├─ config.set_project_location(parent)                          │
│     │       └─ Collection.from_module(module) → self.collection            │
│     ├─ parse_tasks()                                                        │
│     │   ├─ _make_parser()                                                   │
│     │   │   └─ Parser(contexts=collection.to_contexts())                    │
│     │   │          └── build_context, test_context, ...                    │
│     │   ├─ parser.parse_argv(self.core.unparsed)                           │
│     │   │   └─ ParseResult([core_via, build_ctx, test_ctx])                │
│     │   └─ self.tasks = [build_ctx, test_ctx] (ParserContext 列表)        │
│     ├─ parse_cleanup() → 检查 --help, --list 等                           │
│     ├─ update_config() → 合并 CLI 参数到 Config                             │
│     └─ execute()                                                            │
│         └─ Executor.execute(*self.tasks)                                    │
│                │                                                            │
│                ▼                                                            │
│             3. Executor.execute(build_ctx, test_ctx)                       │
│                ├─ normalize()                                                │
│                │   └─ ParserContext → Call                                  │
│                │       ├─ name = ctx.name                                   │
│                │       ├─ kwargs = ctx.as_kwargs                           │
│                │       └─ Call(collection[name], kwargs, called_as=name)   │
│                ├─ expand_calls() → 展开 pre/post                            │
│                ├─ dedupe() → 去重                                           │
│                └─ 执行循环: for call in calls:                              │
│                       call.task(context, **call.kwargs)                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键数据结构流转

| 阶段 | 数据形式 | 包含内容 |
|------|---------|---------|
| CLI 输入 | `List[str]` | `["invoke", "build", "--verbose", ...]` |
| 核心解析后 | `ParseResult` | `[core_context]`, `unparsed=["build", ...]` |
| 任务解析后 | `List[ParserContext]` | `[build_context, test_context]` |
| Executor 输入 | `*ParserContext` | 解包传入 |
| normalize 后 | `List[Call]` | `[Call(build_task, {...}), Call(test_task, {...})]` |
| 执行时 | `Task` + kwargs | `call.task(context, **call.kwargs)` |

### 6.3 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| add_collection() | `invoke/collection.py` | 285-324 |
| _check_default_collision() | `invoke/collection.py` | 326-329 |
| Program.run() | `invoke/program.py` | 355-422 |
| Program.parse_core() | `invoke/program.py` | 424-452 |
| Program.parse_collection() | `invoke/program.py` | 454-489 |
| Program.load_collection() | `invoke/program.py` | 702-729 |
| Program.parse_tasks() | `invoke/program.py` | 750-772 |
| Program.execute() | `invoke/program.py` | 559-583 |
| Executor.normalize() | `invoke/executor.py` | 151-179 |
| Loader.load() | `invoke/loader.py` | 49-96 |
| FilesystemLoader.find() | `invoke/loader.py` | 123-154 |
| Parser.parse_argv() | `invoke/parser/parser.py` | 90-202 |
| ParseMachine.handle() | `invoke/parser/parser.py` | 272-328 |
| ParserContext | `invoke/parser/context.py` | 59-266 |
| ParserContext.as_kwargs | `invoke/parser/context.py` | 162-174 |

---

**分析日期**：2026-04-27
**分析版本**：invoke-7066
