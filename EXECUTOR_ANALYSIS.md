# Invoke Executor 深度分析报告

## 1. 概述

本文档深入分析 invoke 库中 `Executor` 类的实现机制，包括：
- pre/post 钩子展开机制
- dedupe 去重机制
- 依赖图构建与循环依赖检测
- 并行调度与异常回滚
- Context 线程安全保证

## 2. pre/post 钩子展开机制

### 2.1 核心实现

`Executor.expand_calls()` 方法负责展开 pre/post 钩子，采用**深度优先递归**策略。

**关键代码位置**：`invoke/executor.py:201-232`

```python
def expand_calls(self, calls: List["Call"]) -> List["Call"]:
    ret = []
    for call in calls:
        # 递归展开 pre 任务
        ret.extend(self.expand_calls(call.pre))
        # 添加当前任务
        ret.append(call)
        # 递归展开 post 任务
        ret.extend(self.expand_calls(call.post))
    return ret
```

### 2.2 展开顺序

展开遵循以下顺序：
1. 先递归展开所有 **pre** 任务
2. 然后添加 **当前** 任务
3. 最后递归展开所有 **post** 任务

### 2.3 实际示例分析

以 `tests/_support/depth_first.py` 中的任务为例：

```python
@task
def clean_html(c): print("Cleaning HTML")

@task
def clean_tgz(c): print("Cleaning .tar.gz files")

@task(clean_html, clean_tgz)  # pre = [clean_html, clean_tgz]
def clean(c): print("Cleaned everything")

@task
def makedirs(c): print("Making directories")

@task(clean, makedirs)  # pre = [clean, makedirs]
def build(c): print("Building")

@task
def pretest(c): print("Preparing for testing")

@task(pretest)  # pre = [pretest]
def test(c): print("Testing")

@task(build, post=[test])  # pre = [build], post = [test]
def deploy(c): print("Deploying")
```

**展开后的执行顺序**：
1. `clean_html` (deploy → build → clean 的 pre)
2. `clean_tgz` (deploy → build → clean 的 pre)
3. `clean` (deploy → build 的 pre)
4. `makedirs` (deploy → build 的 pre)
5. `build` (deploy 的 pre)
6. `deploy` (主任务)
7. `pretest` (deploy → test 的 pre)
8. `test` (deploy 的 post)

**测试验证**：`tests/executor.py:138-151` 中的 `chaining_is_depth_first` 测试用例确认了此顺序。

### 2.4 注意事项

- **Task 到 Call 的转换**：在展开过程中，`pre`/`post` 列表中的 `Task` 对象会被转换为 `Call` 对象（`executor.py:218-219`）
- **递归展开**：pre/post 任务本身也可能有自己的 pre/post，会被递归展开

## 3. dedupe 去重机制

### 3.1 核心实现

去重逻辑在 `Executor.dedupe()` 方法中实现。

**关键代码位置**：`invoke/executor.py:181-199`

```python
def dedupe(self, calls: List["Call"]) -> List["Call"]:
    deduped = []
    for call in calls:
        if call not in deduped:
            deduped.append(call)
    return deduped
```

### 3.2 去重判断依据

去重的核心在于 `Call.__eq__()` 方法的实现：

**关键代码位置**：`invoke/tasks.py:421-429`

```python
def __eq__(self, other: object) -> bool:
    # 只比较 task、args、kwargs，不比较 called_as
    for attr in "task args kwargs".split():
        if getattr(self, attr) != getattr(other, attr):
            return False
    return True
```

**去重规则**：
- 两个 `Call` 对象相等当且仅当：
  - 它们的 `task` 属性指向同一个 `Task` 对象
  - 它们的 `args` 属性相同
  - 它们的 `kwargs` 属性相同
- **不比较** `called_as` 属性（即通过不同别名调用的相同任务会被视为相同）

### 3.3 去重时机

去重发生在 **pre/post 展开之后**，执行之前：

**关键代码位置**：`invoke/executor.py:110-118`

```python
expanded = self.expand_calls(calls)  # 1. 展开
calls = self.dedupe(expanded) if dedupe else expanded  # 2. 去重（可选）
```

### 3.4 实际示例分析

**示例 1：相邻钩子去重**

任务定义（`tests/_support/integration.py`）：
```python
@task
def foo(c): print("foo")

@task(foo)  # pre = [foo]
def bar(c): print("bar")

@task(foo, bar, post=[post1, post2])  # pre = [foo, bar]
def biz(c): print("biz")
```

**展开后的调用列表**（不去重）：
`[foo, foo, bar, biz, post1, post2, post2]`

**去重后**：
`[foo, bar, biz, post1, post2]`

**测试验证**：`tests/executor.py:156-181`

**示例 2：不同参数的相同任务**

**关键代码位置**：`tests/executor.py:252-265`

```python
t1 = Task(body)
pre = [call(t1, 5), call(t1, 7), call(t1, 5)]  # t1(5) 出现两次
t2 = Task(Mock(), pre=pre)
```

**去重结果**：
- `t1(5)` 只执行一次（第一次出现）
- `t1(7)` 执行一次
- 参数不同的相同任务会被视为不同的调用

### 3.5 配置控制

去重行为可通过配置控制：
- 默认：`dedupe = True`（启用去重）
- 可通过 `--no-dedupe` CLI 标志或 `tasks.dedupe = False` 禁用

## 4. 依赖图构建与循环依赖检测

### 4.1 依赖图构建

**重要发现**：Invoke **没有显式构建依赖图**。

当前实现采用的是**线性展开**策略，而非基于依赖图的拓扑排序：

1. 通过 `expand_calls()` 递归展开 pre/post 任务
2. 生成一个线性的调用列表
3. 按顺序执行

**这种设计的特点**：
- **简单直观**：pre 任务在当前任务之前执行，post 任务在之后
- **深度优先**：遵循递归展开的自然顺序
- **无依赖冲突解决**：不处理复杂的依赖关系（如多个任务依赖同一个任务的不同参数版本）

### 4.2 循环依赖检测

**重要发现**：Invoke **没有循环依赖检测机制**。

**问题场景**：
```python
@task
def a(c): pass

@task(a)  # b 依赖 a
def b(c): pass

# 修改：让 a 也依赖 b（循环依赖）
a.pre = [b]
```

**当前行为**：
- `expand_calls()` 会进入**无限递归**
- 最终抛出 `RecursionError`
- 没有友好的错误提示

### 4.3 与传统依赖图系统的对比

| 特性 | Invoke 当前实现 | 传统依赖图系统（如 make、tox） |
|------|----------------|-------------------------------|
| 依赖表示 | pre/post 列表 | 显式依赖边 |
| 执行顺序 | 深度优先展开 | 拓扑排序 |
| 循环依赖检测 | ❌ 无 | ✅ 有 |
| 并行执行基础 | ❌ 无依赖图 | ✅ 可基于入度调度 |

## 5. 并行调度与异常回滚机制

### 5.1 当前实现状态

**重要发现**：Invoke 核心 `Executor` **没有并行执行能力**。

**关键代码位置**：`invoke/executor.py:124-149`

```python
# 纯串行执行
for call in calls:
    context = call.make_context(config, core_parse_result=self.core)
    args = (context, *call.args)
    result = call.task(*args, **call.kwargs)
    results[call.task] = result
```

### 5.2 关于 "parallel execution" 的澄清

虽然文档和注释中提到了 "parallel execution"，但需要澄清：

1. **网站文档**（`sites/www/index.rst:82`）提到 "parallel execution"，但这是**未来计划**或**上层库（如 Fabric）的功能**，不是当前 invoke 核心的实现。

2. **THOUGHTS.rst**（第 163 行）中的注释表明并行执行是**未来规划**：
   ```python
   # NOTE: this is where parallelization would occur; probably
   # need to move into sub-method
   ```

3. **ExceptionHandlingThread**（`invoke/util.py:146-257`）：
   - 这是一个线程包装类，用于捕获线程中的异常
   - 来自 Fabric 1 的遗产
   - **未被核心 Executor 使用**
   - 测试文件 `tests/concurrency.py` 只测试了这个类本身，没有测试并行任务执行

### 5.3 异常处理与回滚

**异常处理**：
- 任务执行时抛出的异常会**直接向上传播**
- 执行会**立即停止**，后续任务不会执行
- 没有 try-except 包装（除非用户自己在任务中处理）

**回滚机制**：
- **没有回滚机制**
- 已执行的任务的副作用无法撤销
- 如果任务 A 执行成功，任务 B 执行失败，任务 A 的修改会保留

### 5.4 对比分析

| 特性 | Invoke 当前实现 | 理想的并行执行器 |
|------|----------------|-----------------|
| 执行方式 | 纯串行 for 循环 | 线程/进程池 |
| 依赖感知 | 无（线性执行） | 基于依赖图调度 |
| 异常处理 | 立即停止，无回滚 | 可选择继续/停止，支持回滚钩子 |
| 异常传播 | 直接抛出 | 收集所有异常后统一报告 |

## 6. Context 线程安全保证

### 6.1 Context 的创建与共享

**关键测试验证**：`tests/executor.py:337-354`

```python
def context_is_new_but_config_is_same(self):
    @task
    def task1(c): return c
    
    @task
    def task2(c): return c
    
    ret = Executor(collection=coll).execute("task1", "task2")
    c1 = ret[task1]
    c2 = ret[task2]
    
    assert c1 is not c2        # Context 实例不同
    assert c1.config is c2.config  # 但共享同一个 Config 对象
```

### 6.2 创建机制

**关键代码位置**：`invoke/executor.py:141`

```python
context = call.make_context(config, core_parse_result=self.core)
```

**关键代码位置**：`invoke/tasks.py:431-443`

```python
def make_context(self, config: "Config", core_parse_result: "ParseResult") -> Context:
    return Context(config=config, remainder=core_parse_result.remainder)
```

**流程**：
1. 每个任务执行时，调用 `call.make_context()`
2. 传入**同一个** `config` 对象引用
3. 创建**新的** `Context` 实例

### 6.3 共享状态分析

| 对象 | 是否共享 | 说明 |
|------|---------|------|
| `Context` 实例 | ❌ 不共享 | 每个任务创建新实例 |
| `Context.config` | ✅ 共享 | 所有 Context 引用同一个 Config 对象 |
| `Context.command_prefixes` | ❌ 不共享 | 每个 Context 有自己的列表 |
| `Context.command_cwds` | ❌ 不共享 | 每个 Context 有自己的列表 |

### 6.4 配置状态的持久化

**关键测试验证**：`tests/executor.py:356-406`

```python
def new_config_data_is_preserved_between_tasks(self):
    @task
    def task1(c):
        c.foo = "bar"  # 修改 config
        return c
    
    @task
    def task2(c):
        return c
    
    ret = Executor(collection=coll).execute("task1", "task2")
    c2 = ret[task2]
    
    assert "foo" in c2.config  # task1 的修改被 task2 看到
    assert c2.foo == "bar"
```

**这意味着**：
- 虽然 `Context` 实例是新的，但 `config` 是共享的
- 一个任务对 `config` 的修改会影响后续所有任务
- 这在串行执行中是可预测的行为，但在并行执行中会导致竞态条件

### 6.5 线程安全分析

**Context 类**（`invoke/context.py:22-411`）：
- 没有使用任何锁（`threading.Lock`、`RLock` 等）
- 没有使用线程局部存储（`threading.local`）
- `DataProxy` 基类也没有线程安全机制

**Config 类**（`invoke/config.py`）：
- 同样没有线程安全机制
- 配置的读写是直接的字典操作
- 没有原子性保证

### 6.6 并行执行的风险

如果在并行环境中使用当前实现，会面临以下风险：

1. **竞态条件**：多个任务同时修改共享的 `config` 对象
2. **非原子操作**：配置的读写可能被中断
3. **不可预测的行为**：任务执行顺序影响配置状态

**示例风险场景**：
```python
@task
def task_a(c):
    c.config.some_key = "value_a"  # 并行时可能被覆盖
    # ... 使用 some_key ...

@task
def task_b(c):
    c.config.some_key = "value_b"  # 可能覆盖 task_a 的修改
    # ... 使用 some_key ...
```

## 7. 总结与建议

### 7.1 核心发现汇总

| 模块 | 现状 | 限制/问题 |
|------|------|-----------|
| pre/post 展开 | 深度优先递归 | 无 |
| dedupe 去重 | 基于 Call 相等性 | 参数不同的相同任务不会去重 |
| 依赖图 | 无线性展开，无依赖图 | 无循环依赖检测，无法并行调度 |
| 并行执行 | 纯串行 | 无并行能力 |
| 异常回滚 | 无 | 失败后无法回滚已执行任务 |
| Context 线程安全 | 无内置保障 | 共享 Config 有竞态条件风险 |

### 7.2 架构建议

如果需要增强 invoke 的能力，建议考虑以下改进：

1. **引入显式依赖图**：
   - 构建任务依赖图（DAG）
   - 实现拓扑排序
   - 添加循环依赖检测

2. **实现并行执行器**：
   - 基于依赖图的入度调度
   - 使用线程池或进程池
   - 参考 `concurrent.futures` 模块

3. **增强异常处理**：
   - 收集所有并行任务的异常
   - 提供回滚钩子机制
   - 可选的失败继续模式

4. **线程安全改进**：
   - 为 Config 添加读写锁
   - 或考虑任务级别的 Config 隔离
   - 提供清晰的线程安全文档

### 7.3 代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| Executor 主类 | `invoke/executor.py` | 13-232 |
| execute 方法 | `invoke/executor.py` | 52-149 |
| expand_calls 方法 | `invoke/executor.py` | 201-232 |
| dedupe 方法 | `invoke/executor.py` | 181-199 |
| Call 类 | `invoke/tasks.py` | 363-490 |
| Call.__eq__ | `invoke/tasks.py` | 421-429 |
| Context 类 | `invoke/context.py` | 22-411 |
| ExceptionHandlingThread | `invoke/util.py` | 146-257 |

---

**分析日期**：2026-04-27
**分析版本**：invoke-7066
