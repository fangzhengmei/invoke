# Invoke 配置多源层叠机制与 Context 快照隔离设计分析

## 目录

1. [配置多源层叠机制](#1-配置多源层叠机制)
2. [DataProxy 设计：点号属性与字典键访问的等价性](#2-dataproxy-设计点号属性与字典键访问的等价性)
3. [多源配置的合并时机](#3-多源配置的合并时机)
4. [Context 快照隔离设计](#4-context-快照隔离设计)
5. [配置加载器：格式解析差异与统一接口](#5-配置加载器格式解析差异与统一接口)

---

## 1. 配置多源层叠机制

### 1.1 配置来源与优先级

Invoke 的配置系统采用多层级优先级设计，共有 **10 个配置层级**，按优先级从低到高排列如下：

| 优先级 | 层级名称 | 数据来源 | 优先级常数 |
|--------|----------|----------|------------|
| 1 (最低) | defaults | 系统默认配置 | `_defaults` |
| 2 | collection | 任务集合配置 | `_collection` |
| 3 | system | 系统级配置文件 | `_system` |
| 4 | user | 用户级配置文件 | `_user` |
| 5 | project | 项目级配置文件 | `_project` |
| 6 | env | 环境变量 | `_env` |
| 7 | runtime | 运行时配置文件 | `_runtime` |
| 8 | overrides | 命令行参数 | `_overrides` |
| 9 | modifications | 用户运行时修改 | `_modifications` |
| 10 (最高) | deletions | 用户删除操作 | `_deletions` |

### 1.2 优先级层叠合并实现

合并逻辑在 `config.py:941-964` 的 `merge()` 方法中实现：

```python
def merge(self) -> None:
    self._set(_config={})
    debug("Defaults: {!r}".format(self._defaults))
    merge_dicts(self._config, self._defaults)
    debug("Collection-driven: {!r}".format(self._collection))
    merge_dicts(self._config, self._collection)
    self._merge_file("system", "System-wide")
    self._merge_file("user", "Per-user")
    self._merge_file("project", "Per-project")
    debug("Environment variable config: {!r}".format(self._env))
    merge_dicts(self._config, self._env)
    self._merge_file("runtime", "Runtime")
    debug("Overrides: {!r}".format(self._overrides))
    merge_dicts(self._config, self._overrides)
    debug("Modifications: {!r}".format(self._modifications))
    merge_dicts(self._config, self._modifications)
    debug("Deletions: {!r}".format(self._deletions))
    obliterate(self._config, self._deletions)
```

**关键设计要点：**

1. **顺序严格性**：合并按固定顺序进行，高优先级覆盖低优先级
2. **深度合并**：使用 `merge_dicts()` 进行递归深度合并，而非简单的字典覆盖
3. **删除隔离**：`deletions` 层级使用 `obliterate()` 独立处理，确保删除操作可以覆盖所有层级的设置

### 1.3 懒合并与立即合并策略

#### 立即合并触发时机

1. **初始化时**（`config.py:651-655`）：
   - 当 `lazy=False`（默认值）时，`__init__` 会自动调用：
     - `load_base_conf_files()` → 加载 system 和 user 配置
     - `merge()` → 执行一次完整合并

2. **显式加载时**：
   - 所有 `load_*` 方法默认 `merge=True`，加载后立即合并
   - 例如 `load_system()`、`load_user()`、`load_project()` 等

#### 懒合并触发时机

1. **延迟初始化**：
   - 当 `lazy=True` 时，`load_base_conf_files()` 不会被调用
   - 但 `merge()` 仍会被调用以确保 defaults 和 overrides 可用

2. **显式延迟**：
   - 调用 `load_*(merge=False)` 时，仅加载到对应层级字典，不触发合并
   - 需要手动调用 `merge()` 才会生效

#### 策略对比

| 策略 | 触发时机 | 性能特点 | 适用场景 |
|------|----------|----------|----------|
| 立即合并 | 默认行为 | 每次加载都合并，可能重复计算 | 简单场景，配置层级少 |
| 懒合并 | `lazy=True` 或 `merge=False` | 延迟到需要时合并，减少计算 | 复杂场景，多层级配置 |

### 1.4 递归深度合并算法

`merge_dicts()` 函数（`config.py:1168-1224`）实现了智能的深度合并：

```python
def merge_dicts(
    base: Dict[str, Any], updates: Dict[str, Any]
) -> Dict[str, Any]:
    for key, value in (updates or {}).items():
        if key in base:
            if isinstance(value, dict):
                if isinstance(base[key], dict):
                    merge_dicts(base[key], value)  # 递归合并
                else:
                    raise _merge_error(base[key], value)  # 类型冲突
            else:
                if isinstance(base[key], dict):
                    raise _merge_error(base[key], value)  # 类型冲突
                else:
                    base[key] = copy.copy(value)  # 浅拷贝覆盖
        else:
            if isinstance(value, dict):
                base[key] = copy_dict(value)  # 递归拷贝新字典
            else:
                base[key] = copy.copy(value)  # 浅拷贝新值
    return base
```

**核心规则：**

1. **同键递归**：如果 base 和 updates 中都有该键且都是 dict，递归合并
2. **类型冲突**：如果一个是 dict 另一个不是，抛出 `AmbiguousMergeError`
3. **值覆盖**：非 dict 值直接覆盖（使用浅拷贝避免共享引用）
4. **新键添加**：base 中没有的键直接添加（dict 类型递归拷贝）

---

## 2. DataProxy 设计：点号属性与字典键访问的等价性

### 2.1 设计目标

DataProxy 的核心目标是实现：
- **统一访问**：`config.foo.bar` 和 `config['foo']['bar']` 完全等价
- **递归包装**：嵌套字典自动转换为 DataProxy，保持访问一致性
- **修改追踪**：支持追踪用户对配置的修改和删除操作

### 2.2 核心实现机制

#### 属性访问 → 字典键访问的转换

`__getattr__` 方法（`config.py:111-129`）：

```python
def __getattr__(self, key: str) -> Any:
    try:
        return self._get(key)
    except KeyError:
        if key in self._proxies:
            return getattr(self._config, key)
        err = "No attribute or config key found for {!r}".format(key)
        # ... 构建有用的错误信息
        raise AttributeError(err)
```

**关键点：**

1. **优先级**：Python 默认属性查找优先于 `__getattr__`，因此真实属性（如方法）不会被配置键覆盖
2. **代理转发**：`_proxies` 列表中的特殊方法（如 `keys()`、`items()`）直接转发到底层 `_config` 字典
3. **错误友好**：当键不存在时，提供有效键列表和真实属性列表的错误信息

#### 属性设置 → 字典键设置的转换

`__setattr__` 方法（`config.py:131-140`）：

```python
def __setattr__(self, key: str, value: Any) -> None:
    has_real_attr = key in dir(self)
    if not has_real_attr:
        self[key] = value  # 转换为字典键访问
    else:
        super().__setattr__(key, value)  # 真实属性正常设置
```

**设计决策：**

- **真实属性优先**：使用 `key in dir(self)` 判断是否为真实属性
- **隐式转换**：非真实属性自动转换为字典键设置
- **内部设置**：类内部使用 `_set()` 方法绕过代理机制

#### 字典协议实现

DataProxy 完整实现了字典协议：

| 方法 | 实现位置 | 功能 |
|------|----------|------|
| `__getitem__` | line 167-168 | 委托给 `_get()` |
| `__setitem__` | line 163-166 | 设置值并追踪修改 |
| `__delitem__` | line 243-245 | 删除值并追踪删除 |
| `__iter__` | line 142-145 | 迭代底层字典 |
| `__len__` | line 160-161 | 返回底层字典长度 |
| `__contains__` | line 210-211 | 检查键是否存在 |

### 2.3 递归嵌套包装

#### 动态包装机制

`_get()` 方法（`config.py:170-188`）实现了递归包装：

```python
def _get(self, key: str) -> Any:
    if key in ("__setstate__",):
        raise AttributeError(key)
    value = self._config[key]
    if isinstance(value, dict):
        keypath = (key,)
        if hasattr(self, "_keypath"):
            keypath = self._keypath + keypath
        root = getattr(self, "_root", self)
        value = DataProxy.from_data(data=value, root=root, keypath=keypath)
    return value
```

**包装过程：**

1. **类型检查**：只有当值是 `dict` 类型时才进行包装
2. **路径追踪**：构建 `keypath` 元组，记录从根到当前节点的路径
3. **根引用**：保持对根对象的引用，用于修改追踪
4. **延迟创建**：只在访问时创建 DataProxy，避免预先遍历整个字典

#### from_data 工厂方法

`from_data()` 方法（`config.py:77-109`）用于创建嵌套的 DataProxy：

```python
@classmethod
def from_data(
    cls,
    data: Dict[str, Any],
    root: Optional["DataProxy"] = None,
    keypath: Tuple[str, ...] = tuple(),
) -> "DataProxy":
    obj = cls()
    obj._set(_config=data)
    obj._set(_root=root)
    obj._set(_keypath=keypath)
    return obj
```

**参数说明：**

- `data`: 该 DataProxy 包装的字典数据
- `root`: 根 DataProxy/Config 的引用，用于修改通知
- `keypath`: 从根到当前节点的键路径元组

### 2.4 修改追踪机制

#### 层级设计

Config（DataProxy 的子类）引入了三个特殊层级用于追踪运行时操作：

```python
# 最高优先级：用户修改
self._set(_modifications={})

# 独立处理：用户删除（存储为扁平结构）
self._set(_deletions={})
```

#### 修改追踪流程

当用户通过任何方式修改配置时（`config.py:163-166`）：

```python
def __setitem__(self, key: str, value: str) -> None:
    self._config[key] = value
    self._track_modification_of(key, value)
```

`_track_modification_of()` 方法（`config.py:234-241`）：

```python
def _track_modification_of(self, key: str, value: str) -> None:
    target = None
    if self._is_leaf:
        target = self._root
    elif self._is_root:
        target = self
    if target is not None:
        target._modify(getattr(self, "_keypath", tuple()), key, value)
```

**核心逻辑：**

1. **叶子/根判断**：`_is_leaf` 表示有 `_root` 引用，`_is_root` 表示有 `_modify` 方法
2. **向上委托**：嵌套 DataProxy 将修改操作委托给根 Config
3. **路径传递**：传递 `keypath` 以记录修改在哪个层级

#### 实际修改存储

`_modify()` 方法（`config.py:1102-1130`）：

```python
def _modify(self, keypath: Tuple[str, ...], key: str, value: str) -> None:
    # 1. 从 deletions 中移除（如果之前被删除过）
    excise(self._deletions, keypath + (key,))
    
    # 2. 构建嵌套路径并存储到 modifications
    data = self._modifications
    keypath_list = list(keypath)
    while keypath_list:
        subkey = keypath_list.pop(0)
        if subkey not in data:
            data[subkey] = {}
        data = data[subkey]
    data[key] = value
    
    # 3. 重新合并以生效
    self.merge()
```

#### 删除追踪流程

删除操作与修改类似，但使用独立的 `_deletions` 结构（`config.py:1132-1161`）：

```python
def _remove(self, keypath: Tuple[str, ...], key: str) -> None:
    data = self._deletions
    keypath_list = list(keypath)
    while keypath_list:
        subkey = keypath_list.pop(0)
        if subkey in data:
            data = data[subkey]
            if data is None:
                return  # 上级已被删除，无需处理
        else:
            data[subkey] = {}
            data = data[subkey]
    data[key] = None  # 标记为 None 表示删除
    self.merge()
```

**删除处理的特殊设计：**

- `_deletions` 存储为嵌套字典结构，叶子节点值为 `None`
- 在 `merge()` 最后调用 `obliterate()` 执行实际删除
- 如果上级路径已被标记为删除，下级无需重复标记

### 2.5 访问等价性验证

从测试代码（`tests/config.py:220-236`）可以看到等价性保证：

```python
def allows_dict_and_attr_access(self):
    c = Config({"foo": "bar"})
    assert c.foo == "bar"
    assert c["foo"] == "bar"

def nested_dict_values_also_allow_dual_access(self):
    c = Config({"foo": "bar", "biz": {"baz": "boz"}})
    # 顶层访问
    assert c.foo == "bar"
    assert c["foo"] == "bar"
    # 嵌套访问 - 所有组合方式
    assert c.biz.baz == "boz"
    assert c["biz"]["baz"] == "boz"
    assert c.biz["baz"] == "boz"
    assert c["biz"].baz == "boz"
```

### 2.6 真实属性与配置键的隔离

DataProxy 确保真实属性（方法、特性等）不会被配置键覆盖（`tests/config.py:432-486`）：

```python
def real_attrs_and_methods_win_over_attr_proxying(self):
    class MyConfig(Config):
        myattr = None
        def mymethod(self):
            return 7
    
    c = MyConfig({"myattr": "foo", "mymethod": "bar"})
    
    # 真实属性优先
    assert c.myattr is None  # 不是 "foo"
    assert c["myattr"] == "foo"  # 但可通过字典键访问
    
    # 方法也是真实属性
    assert callable(c.mymethod)
    assert c.mymethod() == 7
    assert c["mymethod"] == "bar"  # 字典键访问配置值
```

---

## 3. 多源配置的合并时机

### 3.1 合并时机分类

Invoke 的配置合并采用**分层加载 + 按需合并**的策略，可分为三类：

| 时机类型 | 触发阶段 | 涉及层级 | 性能影响 |
|----------|----------|----------|----------|
| 初始化合并 | `__init__` 期间 | defaults, overrides, system, user | 一次性成本 |
| 延迟加载合并 | 任务执行前 | collection, project, runtime, env | 按需触发 |
| 运行时合并 | 用户修改时 | modifications, deletions | 每次修改 |

### 3.2 初始化时完成的合并

#### __init__ 流程

`Config.__init__`（`config.py:512-655`）的核心流程：

```python
def __init__(self, ..., lazy: bool = False):
    # 1. 初始化所有层级字典
    self._set(_config={})
    self._set(_defaults=defaults if defaults else copy_dict(self.global_defaults()))
    self._set(_collection={})
    self._set(_system={})
    self._set(_user={})
    self._set(_project={})
    self._set(_env={})
    self._set(_runtime={})
    self._set(_overrides=overrides if overrides else {})
    self._set(_modifications={})
    self._set(_deletions={})
    
    # 2. 选择性加载系统和用户配置
    if not lazy:
        self.load_base_conf_files()  # load_system + load_user，merge=False
    
    # 3. 总是执行一次合并
    self.merge()
```

#### 关键点解析

1. **defaults 总是可用**：
   - 即使 `lazy=True`，`global_defaults()` 也会被复制到 `_defaults`
   - `merge()` 总是被调用，确保基本配置可用

2. **system 和 user 配置的条件加载**：
   - `lazy=False`（默认）：自动调用 `load_base_conf_files()`
   - `lazy=True`：不自动加载，需手动调用 `load_system()` 和 `load_user()`

3. **merge=False 的设计**：
   - `load_base_conf_files()` 内部调用 `load_system(merge=False)` 和 `load_user(merge=False)`
   - 这样可以先加载两个文件，最后统一 `merge()` 一次，避免重复合并

### 3.3 推迟到实际访问时执行的合并

#### 为什么需要延迟加载

某些配置层级依赖运行时信息，无法在初始化时确定：

| 层级 | 依赖信息 | 为何延迟 |
|------|----------|----------|
| collection | 当前执行的任务所属集合 | 必须知道执行哪个任务 |
| project | 任务集合所在目录 | 需先加载任务模块才能确定 |
| runtime | 用户指定的 `-c`/`--config` 参数 | 命令行解析后才知道 |
| env | 现有配置键的结构 | 需知道已有配置才能匹配环境变量 |

#### 延迟加载的执行流程

从 `executor.py:120-149` 可以看到任务执行时的配置加载顺序：

```python
def execute(self, *tasks):
    # ...
    for call in calls:
        # 1. 获取当前任务的集合配置
        collection_config = self.collection.configuration(call.called_as)
        config.load_collection(collection_config)  # merge=True
        
        # 2. 加载环境变量（必须最后，因为依赖现有配置结构）
        config.load_shell_env()
        
        # 3. 创建 Context 并执行任务
        context = call.make_context(config, core_parse_result=self.core)
        result = call.task(*args, **call.kwargs)
```

#### 各延迟层级的加载时机

**1. Project 配置**：

```python
# config.py:732-752
def load_project(self, merge: bool = True) -> None:
    self._load_file(prefix="project", merge=merge)
```

- 依赖 `_project_prefix`，由 `set_project_location()` 设置
- 通常在找到任务集合后调用

**2. Runtime 配置**：

```python
# config.py:768-784
def load_runtime(self, merge: bool = True) -> None:
    self._load_file(prefix="runtime", absolute=True, merge=merge)
```

- 依赖 `_runtime_path`，由 `set_runtime_path()` 设置
- 由 CLI 框架在解析 `--config` 参数后设置

**3. Collection 配置**：

```python
# config.py:811-826
def load_collection(
    self, data: Dict[str, Any], merge: bool = True
) -> None:
    debug("Loading collection configuration")
    self._set(_collection=data)
    if merge:
        self.merge()
```

- 数据来自 `Collection.configuration()` 方法
- 每个任务执行前重新加载，因为不同任务可能属于不同子集合

**4. 环境变量加载**：

```python
# config.py:786-809
def load_shell_env(self) -> None:
    # 强制合并以确保有最新的配置结构
    debug("Running pre-merge for shell env loading...")
    self.merge()
    debug("Done with pre-merge.")
    
    # 基于当前配置结构扫描环境变量
    loader = Environment(config=self._config, prefix=self._env_prefix)
    self._set(_env=loader.load())
    
    debug("Loaded shell environment, triggering final merge")
    self.merge()
```

**环境变量加载的特殊性：**

- **依赖现有配置**：只加载与已有配置键匹配的环境变量
- **类型转换**：根据已有配置值的类型进行转换（如 bool、int）
- **冲突检测**：检测歧义的下划线分隔键名

### 3.4 环境变量加载的深层机制

`Environment` 类（`env.py:21-123`）实现了智能的环境变量映射：

#### 键名映射规则

```python
# env.py:88-89
def _to_env_var(self, key_path: Iterable[str]) -> str:
    return "_".join(key_path).upper()
```

**映射示例：**

| 配置键路径 | 环境变量名 |
|-------------|------------|
| `run.echo` | `INVOKE_RUN_ECHO` |
| `sudo.password` | `INVOKE_SUDO_PASSWORD` |
| `foo_bar.biz` | `INVOKE_FOO_BAR_BIZ` |

#### 歧义检测

由于下划线可能表示嵌套或键名本身的一部分，需要检测歧义：

```python
# env.py:48-86
def _crawl(self, key_path, env_vars):
    new_vars = {}
    obj = self._path_get(key_path)
    
    if hasattr(obj, "keys") and callable(obj.keys):
        for key in obj.keys():
            merged_vars = dict(env_vars, **new_vars)
            merged_path = key_path + [key]
            crawled = self._crawl(merged_path, merged_vars)
            
            # 冲突检测
            for key in crawled:
                if key in new_vars:
                    err = "Found >1 source for {}"
                    raise AmbiguousEnvVar(err.format(key))
            
            new_vars.update(crawled)
    else:
        new_vars[self._to_env_var(key_path)] = key_path
    
    return new_vars
```

**歧义示例：**

```python
# 这种配置会导致 AmbiguousEnvVar
# 因为 INVOKE_FOO_BAR 可以映射到：
# - foo_bar 键
# - foo.bar 路径
c = Config(defaults={"foo_bar": "wat", "foo": {"bar": "huh"}})
c.load_shell_env()  # 抛出异常
```

#### 类型转换

```python
# env.py:111-123
def _cast(self, old: Any, new: Any) -> Any:
    if isinstance(old, bool):
        return new not in ("0", "")  # 特殊处理布尔值
    elif isinstance(old, str):
        return new
    elif old is None:
        return new
    elif isinstance(old, (list, tuple)):
        raise UncastableEnvVar(...)  # 列表/元组不可转换
    else:
        return old.__class__(new)  # 使用类构造器转换
```

**转换规则：**

| 原有类型 | 转换逻辑 | 示例 |
|----------|----------|------|
| `bool` | `"0"` 或 `""` → `False`，其他 → `True` | `"false"` → `True` |
| `str` | 直接返回 | `"hello"` → `"hello"` |
| `None` | 直接返回 | `"value"` → `"value"` |
| `int/float` | 调用构造器 | `"5"` → `5` |
| `list/tuple` | 抛出异常 | 不支持 |

### 3.5 设计对性能与一致性的影响

#### 性能考量

**优点：**

1. **按需加载**：
   - 不需要的层级不加载（如没有 project 配置就不查找）
   - 环境变量只扫描已有配置键，避免遍历整个环境

2. **合并可控**：
   - `merge=False` 允许批量加载后统一合并
   - 减少不必要的重复合并操作

**潜在成本：**

1. **频繁合并**：
   - 每个任务执行时：`load_collection()` + `load_shell_env()` → 至少 3 次合并
   - 用户每次修改配置都会触发 `merge()`

2. **重复扫描**：
   - `load_shell_env()` 每次都要重新爬取配置结构
   - 每次都重新扫描环境变量

#### 一致性考量

**优点：**

1. **任务隔离**：
   - 每个任务重新加载 `_collection` 层级
   - 确保任务看到的是自己集合的配置

2. **环境变量新鲜度**：
   - 每次执行任务都重新加载环境变量
   - 反映最新的环境状态

3. **修改追踪**：
   - `_modifications` 和 `_deletions` 独立存储
   - 每次 `merge()` 都能正确应用

**潜在风险：**

1. **时序依赖**：
   - 环境变量加载依赖之前的合并结果
   - 如果顺序错误，可能导致不一致

2. **并行问题**：
   - 多个任务共享同一个 Config 对象
   - 如果任务并行执行，可能互相干扰（见第 4 章）

---

## 4. Context 快照隔离设计

### 4.1 设计目标

Context 的核心设计目标：
1. **配置封装**：将配置与运行时状态（如当前目录、命令前缀）封装在一起
2. **任务隔离**：避免并行或顺序执行的任务通过共享配置互相污染
3. **便捷访问**：提供对配置的透明代理访问

### 4.2 Context 的核心结构

#### 类继承关系

```
DataProxy (config.py:36)
    ↓ 继承
Context (context.py:22)
    ↓ 继承
MockContext (context.py:414)
```

#### Context 数据结构

`context.py:59-80` 的 `__init__` 方法：

```python
def __init__(
    self,
    config: Optional[Config] = None,
    remainder: str = "",
) -> None:
    config = config if config is not None else Config()
    self._set(
        _config=config,  # 注意：这里 _config 存储的是 Config 对象，不是 dict！
        command_prefixes=[],
        command_cwds=[],
        remainder=remainder,
    )
```

**关键设计：**

1. **双层代理**：
   - Context 的 `_config` 是一个 Config 对象（也是 DataProxy 子类）
   - Context 本身也是 DataProxy，代理访问其 `_config`（即 Config 对象）

2. **运行时状态**：
   - `command_prefixes`: `prefix()` 上下文管理器的命令前缀栈
   - `command_cwds`: `cd()` 上下文管理器的目录栈
   - `remainder`: 命令行 `--` 后的剩余参数

#### 配置代理访问

Context 通过 `config` 属性和代理机制提供配置访问：

```python
# context.py:82-106
@property
def config(self) -> Config:
    return self._config  # 返回内部的 Config 对象

@config.setter
def config(self, value: Config) -> None:
    self._set(_config=value)
```

同时，Context 继承 DataProxy，所以：
- `context.foo` → 调用 `__getattr__` → 调用 `_get('foo')` → 访问 `self._config['foo']`
- 但 Context 的 `_config` 是 Config 对象，Config 的 `_config` 才是真正的合并后字典

### 4.3 任务执行时的配置处理

#### Executor 的执行流程

从 `executor.py:52-149` 分析任务执行的配置处理：

```python
def execute(self, *tasks):
    # 1. 规范化任务
    calls = self.normalize(tasks)
    direct = list(calls)
    expanded = self.expand_calls(calls)
    calls = self.dedupe(expanded) if dedupe else expanded
    
    results = {}
    
    for call in calls:
        # 2. 直接使用共享的 config 对象
        config = self.config  # 注意：是引用，不是拷贝！
        
        # 3. 重置任务敏感的配置层级
        # 加载当前任务的 collection 配置
        collection_config = self.collection.configuration(call.called_as)
        config.load_collection(collection_config)  # 覆盖 _collection
        
        # 加载环境变量
        config.load_shell_env()  # 覆盖 _env
        
        # 4. 创建 Context（仍共享同一个 config）
        context = call.make_context(config, core_parse_result=self.core)
        
        # 5. 执行任务
        args = (context, *call.args)
        result = call.task(*args, **call.kwargs)
        
        results[call.task] = result
    
    return results
```

#### 现有设计的问题分析

**当前实现的特点：**

1. **共享 Config 对象**：
   - 所有任务使用同一个 `self.config` 引用
   - 每个任务覆盖 `_collection` 和 `_env` 层级

2. **顺序执行安全**：
   - 单线程顺序执行时，前一个任务的修改会被后一个任务覆盖
   - 但这是预期行为（每个任务加载自己的 collection 配置）

3. **并行执行风险**：
   - 如果任务并行执行，多个任务同时修改同一个 Config 对象
   - 可能导致：
     - 任务 A 加载了自己的 collection 配置
     - 任务 B 覆盖了 `_collection`
     - 任务 A 执行时看到的是任务 B 的配置

#### 任务级别的"快照"机制

虽然 Config 对象是共享的，但通过以下机制实现了一定程度的隔离：

**1. 层级隔离**：

每个层级独立存储，合并时才组合：

```python
# 每个任务执行前：
config.load_collection(task_A_config)  # _collection = task_A_config
config.load_shell_env()                # _env = 基于当前配置的 env

# 下一个任务：
config.load_collection(task_B_config)  # _collection 被覆盖为 task_B_config
config.load_shell_env()                # _env 被重新计算
```

**2. 用户修改的持久化**：

`_modifications` 和 `_deletions` 是最高优先级，不会被 `load_collection` 等方法覆盖：

```python
# 假设用户做了修改：
context.run.echo = True  # 存入 _modifications

# 即使重新加载 collection：
config.load_collection(new_collection_config)  # 只修改 _collection
config.merge()  # _modifications 仍会覆盖其他层级

# run.echo 仍然是 True
```

**3. Collection 配置的来源**：

`Collection.configuration()` 方法为每个任务返回独立的配置：

```python
# 伪代码示意
class Collection:
    def configuration(self, task_name):
        # 返回该任务（或其子集合）特定的配置
        # 不同任务可能返回不同的配置字典
        return self._configs.get(task_name, self.default_config)
```

### 4.4 真正的隔离：Config.clone()

虽然当前执行器没有使用，但 Config 提供了 `clone()` 方法用于创建真正独立的快照：

#### clone() 方法实现

`config.py:985-1071`：

```python
def clone(self, into: Optional[Type["Config"]] = None) -> "Config":
    # 1. 创建新实例（lazy=True 避免重复加载文件）
    klass = self.__class__ if into is None else into
    new = klass(**self._clone_init_kwargs(into=into))
    
    # 2. 复制所有层级数据
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
            # 非 dict 数据浅拷贝
            new._set(name, copy.copy(my_data))
        else:
            # dict 数据使用 merge_dicts（递归拷贝）
            merge_dicts(getattr(new, name), my_data)
    
    # 3. 重新加载基础配置文件
    new.load_base_conf_files()
    
    # 4. 执行合并
    new.merge()
    
    return new
```

#### 克隆策略详解

| 数据类型 | 复制策略 | 原因 |
|----------|----------|------|
| `_defaults`, `_overrides` 等 dict 层级 | `merge_dicts()` 递归拷贝 | 避免共享嵌套字典的引用 |
| `_system_prefix`, `_user_prefix` 等字符串 | `copy.copy()` 浅拷贝 | 不可变类型，浅拷贝足够 |
| 文件路径等基础类型 | 直接赋值 | 简单类型无需拷贝 |

#### 为什么不直接用 deepcopy？

代码中的注释（`config.py:992-1002`）解释了原因：

```python
# Specifically, all dict values within the config are recursively
# recreated, with non-dict leaf values subjected to copy.copy (note:
# *not* copy.deepcopy, as this can cause issues with various objects
# such as compiled regexen or threading locks, often found buried deep
# within rich aggregates like API or DB clients).
```

**deepcopy 的问题：**

1. **对象兼容性**：某些对象（如编译的正则表达式、线程锁）不能正确 deepcopy
2. **过度拷贝**：用户可能在配置中存储了 API 客户端等大对象，deepcopy 会复制它们
3. **性能开销**：deepcopy 比选择性拷贝慢

#### clone() 的使用场景

从测试代码（`tests/config.py:951-1105`）可以看到典型使用场景：

**1. 子任务隔离**：

```python
# 假设在任务中需要修改配置但不影响全局
@task
def my_task(c):
    # 创建独立副本
    local_config = c.config.clone()
    local_config.run.echo = True  # 只影响本地副本
    
    # 使用修改后的配置执行子任务
    sub_ctx = Context(config=local_config)
    sub_task(sub_ctx)
```

**2. 配置版本对比**：

```python
original = Config()
modified = original.clone()
modified.foo = "bar"

# 对比差异
if original != modified:
    print("配置已修改")
```

**3. 并行任务隔离**：

```python
# 如果 Executor 使用 clone()，每个任务获得独立配置
def execute(self, *tasks):
    for call in calls:
        # 为每个任务创建独立快照
        task_config = self.config.clone()
        
        # 修改快照，不影响原始配置
        task_config.load_collection(...)
        task_config.load_shell_env()
        
        context = call.make_context(task_config, ...)
        result = call.task(context, ...)
```

### 4.5 Context 的运行时状态隔离

除了配置，Context 还维护运行时状态，这些状态天然隔离：

#### 目录栈（cd 上下文管理器）

```python
# context.py:359-411
@contextmanager
def cd(self, path: Union[PathLike, str]) -> Generator[None, None, None]:
    path = str(path)
    self.command_cwds.append(path)
    try:
        yield
    finally:
        self.command_cwds.pop()  # 确保退出时恢复
```

**隔离性：**

- 每个 Context 有自己的 `command_cwds` 列表
- 使用栈结构 + try/finally 确保嵌套使用安全
- 不同 Context 的目录栈互不影响

#### 命令前缀栈（prefix 上下文管理器）

```python
# context.py:280-334
@contextmanager
def prefix(self, command: str) -> Generator[None, None, None]:
    self.command_prefixes.append(command)
    try:
        yield
    finally:
        self.command_prefixes.pop()
```

**与 cd 相同的隔离特性**。

### 4.6 当前设计的权衡

#### 优点

1. **内存效率**：
   - 多个任务共享同一个 Config 对象
   - 只有层级数据，没有重复的完整配置拷贝

2. **修改持久化**：
   - 用户在一个任务中的修改（存入 `_modifications`）会影响后续任务
   - 这有时是预期行为（如任务 A 设置配置，任务 B 使用）

3. **简单性**：
   - 无需管理多个 Config 实例的生命周期
   - 执行流程直观

#### 缺点

1. **并行不安全**：
   - 多线程/多进程并行执行时，配置可能互相干扰
   - 需要外部同步或使用 `clone()`

2. **任务间隐式耦合**：
   - 前一个任务的修改会影响后一个任务
   - 可能导致难以调试的顺序依赖问题

3. **回滚困难**：
   - 如果任务执行失败，已修改的配置无法自动回滚
   - 需要手动记录和恢复

#### 改进建议

如果需要更强的隔离性，可以修改 Executor 使用 `clone()`：

```python
def execute(self, *tasks):
    # ...
    for call in calls:
        # 为每个任务创建独立的配置快照
        config = self.config.clone()
        
        # 在快照上进行修改
        collection_config = self.collection.configuration(call.called_as)
        config.load_collection(collection_config)
        config.load_shell_env()
        
        # 创建独立的 Context
        context = call.make_context(config, core_parse_result=self.core)
        
        # 执行任务，对 config 的修改不会影响其他任务
        result = call.task(context, *call.args, **call.kwargs)
    # ...
```

---

## 5. 配置加载器：格式解析差异与统一接口

### 5.1 支持的配置格式

Invoke 支持四种配置文件格式，按优先级排序：

| 格式 | 扩展名 | 加载方法 | 优先级 |
|------|--------|----------|--------|
| YAML | `.yaml` | `_load_yaml()` | 1（最高） |
| YAML | `.yml` | `_load_yml()` (同 `_load_yaml`) | 2 |
| JSON | `.json` | `_load_json()` | 3 |
| Python | `.py` | `_load_py()` | 4（最低） |

**注意**：同一目录下只加载找到的第一个文件（按优先级顺序）。

### 5.2 统一加载入口

#### _load_file 方法

所有格式通过统一的 `_load_file()` 方法加载（`config.py:850-911`）：

```python
def _load_file(
    self, prefix: str, absolute: bool = False, merge: bool = True
) -> None:
    # 1. 设置变量名
    found = "_{}_found".format(prefix)
    path = "_{}_path".format(prefix)
    data = "_{}".format(prefix)
    
    # 2. 计算文件名前缀
    midfix = self.file_prefix
    if midfix is None:
        midfix = self.prefix  # 默认 "invoke"
    
    # 3. 检查是否已加载过
    if getattr(self, found) is not None:
        return  # 避免重复加载
    
    # 4. 构建候选文件路径列表
    if absolute:
        # runtime 配置：使用绝对路径
        absolute_path = getattr(self, path)
        if absolute_path is None:
            return
        paths = [absolute_path]
    else:
        # system/user/project 配置：构建多个扩展名的候选路径
        path_prefix = getattr(self, "_{}_prefix".format(prefix))
        if path_prefix is None:
            return
        paths = [
            ".".join((path_prefix + midfix, x))
            for x in self._file_suffixes  # ("yaml", "yml", "json", "py")
        ]
    
    # 5. 按顺序尝试加载
    for filepath in paths:
        filepath = expanduser(filepath)
        try:
            # 根据扩展名选择加载器
            type_ = splitext(filepath)[1].lstrip(".")
            loader = getattr(self, "_load_{}".format(type_))
            
            # 加载数据并存储
            self._set(data, loader(filepath))
            self._set(path, filepath)
            self._set(found, True)
            break  # 找到第一个就停止
            
        except IOError as e:
            if e.errno == 2:  # 文件不存在
                debug("Didn't see any {}, skipping.".format(filepath))
            else:
                raise  # 其他 IO 错误重新抛出
    
    # 6. 处理未找到的情况
    if getattr(self, path) is None:
        self._set(found, False)
    elif merge:
        self.merge()  # 加载成功，执行合并
```

#### 核心设计要点

1. **幂等性保证**：
   - `_{prefix}_found` 标记用于防止重复加载
   - 已加载的文件不会再次读取

2. **扩展名优先级**：
   - `_file_suffixes = ("yaml", "yml", "json", "py")` 定义了尝试顺序
   - 同一目录下 `invoke.yaml` 存在时，`invoke.json` 不会被加载

3. **错误处理策略**：
   - `errno == 2`（文件不存在）：静默跳过，记录 debug 日志
   - 其他 IO 错误：向上抛出异常
   - 未知扩展名：抛出 `UnknownFileType`

### 5.3 各格式解析器实现

#### YAML 格式解析

```python
# config.py:913-917
def _load_yaml(self, path: PathLike) -> Any:
    with open(path) as fd:
        return yaml.safe_load(fd)

_load_yml = _load_yaml  # .yml 扩展名别名
```

**特点：**

1. **使用 safe_load**：
   - 避免执行任意 Python 代码（与 `_load_py` 不同）
   - 只支持标准 YAML 数据类型

2. **别名处理**：
   - `.yaml` 和 `.yml` 使用同一个加载器
   - 但在 `_file_suffixes` 中 `.yaml` 优先级更高

3. **依赖**：
   - 使用 `invoke.util.yaml`，实际是 vendored 的 PyYAML 库
   - 位于 `invoke/vendor/yaml/`

#### JSON 格式解析

```python
# config.py:919-921
def _load_json(self, path: PathLike) -> Any:
    with open(path) as fd:
        return json.load(fd)
```

**特点：**

1. **标准库**：使用 Python 标准 `json` 模块
2. **安全性**：JSON 本身不支持执行代码，天然安全
3. **限制**：不支持注释、 trailing comma 等

#### Python 格式解析

```python
# config.py:923-939
def _load_py(self, path: str) -> Dict[str, Any]:
    data = {}
    for key, value in (load_source("mod", path)).items():
        # 1. 过滤特殊成员
        if key.startswith("__"):
            continue
        
        # 2. 禁止模块类型（防止误用 tasks 文件）
        if isinstance(value, types.ModuleType):
            err = "'{}' is a module, which can't be used as a config value. (Are you perhaps giving a tasks file instead of a config file by mistake?)"
            raise UnpicklableConfigMember(err.format(key))
        
        data[key] = value
    
    return data
```

**辅助函数 `load_source`**（`config.py:26-33`）：

```python
def load_source(name: str, path: str) -> Dict[str, Any]:
    if not os.path.exists(path):
        return {}
    loader = SourceFileLoader("mod", path)
    mod = ModuleType("mod")
    mod.__spec__ = spec_from_loader("mod", loader)
    loader.exec_module(mod)
    return vars(mod)
```

**Python 格式的特殊处理：**

1. **执行模块代码**：
   - 使用 `SourceFileLoader` 执行 Python 文件
   - 这意味着配置文件中可以包含任意 Python 代码

2. **过滤特殊成员**：
   - 跳过 `__builtins__`、`__file__`、`__name__` 等特殊属性
   - 只保留用户定义的变量

3. **模块检测**：
   - 防止用户误用 tasks 文件作为配置文件
   - 如果值是模块类型，抛出明确的错误信息

4. **安全性考虑**：
   - Python 配置文件可以执行任意代码
   - 只应该加载可信的配置文件

### 5.4 三种格式的对比

| 特性 | YAML | JSON | Python |
|------|------|------|--------|
| **语法简洁性** | ⭐⭐⭐ 非常简洁 | ⭐⭐ 一般 | ⭐⭐⭐ 最灵活 |
| **注释支持** | ✅ 支持 | ❌ 不支持 | ✅ 支持 |
| **安全性** | ⭐⭐⭐ 安全（safe_load） | ⭐⭐⭐ 天然安全 | ⭐ 需谨慎（可执行代码） |
| **数据类型** | 标准 YAML 类型 | 标准 JSON 类型 | 任意 Python 类型 |
| **动态计算** | ❌ 不支持 | ❌ 不支持 | ✅ 支持（条件、循环等） |
| **优先级** | 最高 | 中等 | 最低 |

### 5.5 格式无关的使用方式

无论使用哪种格式，用户的使用方式完全一致：

```python
# 1. 初始化时自动加载 system 和 user 配置
config = Config()

# 2. 显式加载其他层级
config.load_project()  # 自动查找项目目录下的配置文件
config.load_runtime()   # 加载 --config 指定的文件

# 3. 统一的访问方式
value = config.some_key
value = config['some_key']
nested = config.section.subsection
nested = config['section']['subsection']
```

### 5.6 配置文件定位策略

不同层级的配置文件有不同的定位方式：

#### System 级配置

```python
# config.py:593-595
if system_prefix is None and not WINDOWS:
    system_prefix = "/etc/"
# 最终路径：/etc/invoke.yaml (或 .yml/.json/.py)
```

- **Unix/Linux**: `/etc/invoke.{yaml,yml,json,py}`
- **Windows**: 无默认系统级配置

#### User 级配置

```python
# config.py:606-608
if user_prefix is None:
    user_prefix = "~/."
# 最终路径：~/.invoke.{yaml,yml,json,py}
```

- 路径：`~/.invoke.{yaml,yml,json,py}`
- 使用 `expanduser()` 解析 `~`

#### Project 级配置

```python
# config.py:828-848
def set_project_location(self, path: Union[PathLike, str, None]) -> None:
    project_prefix = None
    if path is not None:
        project_prefix = join(path, "")  # 确保以路径分隔符结尾
    self._set(_project_prefix=project_prefix)
    # ...
# 最终路径：{project_dir}/invoke.{yaml,yml,json,py}
```

- 依赖 `project_location` 参数，通常是任务集合所在目录
- 由 Loader 在找到任务模块后设置

#### Runtime 级配置

```python
# config.py:754-766
def set_runtime_path(self, path: Optional[PathLike]) -> None:
    self._set(_runtime_path=path)
    self._set(_runtime={})
    self._set(_runtime_found=None)
```

- 由用户通过 `--config` 或 `-c` 命令行参数指定
- 必须是完整的文件路径（不是目录）

### 5.7 实际示例

#### YAML 配置示例

```yaml
# invoke.yaml
run:
  echo: true
  pty: false

sudo:
  prompt: "Password: "

tasks:
  auto_dash_names: true
```

#### JSON 配置示例

```json
{
  "run": {
    "echo": true,
    "pty": false
  },
  "sudo": {
    "prompt": "Password: "
  },
  "tasks": {
    "auto_dash_names": true
  }
}
```

#### Python 配置示例

```python
# invoke.py
import os

# 可以使用动态计算
run = {
    "echo": True,
    "pty": False,
    "shell": os.environ.get("SHELL", "/bin/bash")  # 动态值
}

# 可以使用条件逻辑
if os.environ.get("DEBUG"):
    run["echo"] = True
else:
    run["echo"] = False

sudo = {
    "prompt": "Password: "
}

tasks = {
    "auto_dash_names": True
}
```

### 5.8 加载器的设计优点

1. **格式透明**：
   - 用户无需关心使用哪种格式
   - 统一的 Config API 隐藏了格式差异

2. **优先级灵活**：
   - 同一目录下可选择优先级高的格式覆盖低优先级格式
   - 便于在不同环境使用不同配置策略

3. **错误友好**：
   - 文件不存在静默跳过（便于可选配置）
   - 格式错误抛出明确异常
   - Python 格式检测常见误用（如 tasks 文件）

4. **可扩展性**：
   - 通过 `_file_suffixes` 可以添加新格式
   - 通过添加 `_load_{type}()` 方法支持新格式

---

## 总结

### 核心设计模式

Invoke 的配置与 Context 系统综合运用了多种设计模式：

1. **层级模式（Layered Architecture）**：
   - 10 个配置层级按优先级层叠
   - 每层独立存储，合并时才组合

2. **代理模式（Proxy Pattern）**：
   - DataProxy 统一属性访问和字典访问
   - Context 代理 Config 提供便捷访问

3. **工厂模式（Factory Pattern）**：
   - `DataProxy.from_data()` 动态创建嵌套代理
   - `Config.clone()` 创建独立副本

4. **模板方法模式（Template Method）**：
   - `merge()` 定义固定的合并顺序
   - 各层级加载遵循相似的模板

### 关键设计决策

| 决策点 | 选择 | 权衡 |
|--------|------|------|
| 合并时机 | 延迟合并 + 按需触发 | 内存效率 vs 实时一致性 |
| 任务隔离 | 共享 Config + 层级覆盖 | 简单性 vs 并行安全性 |
| 格式支持 | 多种格式 + 优先级 | 灵活性 vs 复杂度 |
| 修改追踪 | 独立层级存储 | 可追溯性 vs 内存开销 |

### 扩展建议

1. **并行执行支持**：
   - 考虑在 Executor 中使用 `config.clone()` 为每个任务创建独立快照
   - 或引入配置版本号机制检测并发修改

2. **性能优化**：
   - 缓存环境变量扫描结果（如果配置结构未变化）
   - 实现增量合并，只合并变化的层级

3. **新格式支持**：
   - 可通过继承 Config 并覆盖 `_file_suffixes` 和添加 `_load_*` 方法支持 TOML 等新格式

---

*报告生成时间：2026-04-29*

*基于 Invoke 版本：从代码结构分析为 2.x 系列*