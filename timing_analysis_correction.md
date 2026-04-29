# FailingResponder 时序行为纠正分析报告

## 目录
1. [执行顺序的关键发现](#1-执行顺序的关键发现)
2. [首次调用中的实际行为](#2-首次调用中的实际行为)
3. [哨兵索引推进的后果](#3-哨兵索引推进的后果)
4. [纠正后的场景矩阵](#4-纠正后的场景矩阵)
5. [与之前报告的差异总结](#5-与之前报告的差异总结)

---

## 1. 执行顺序的关键发现

### 1.1 精确执行顺序分析

```python
# watchers.py:130-145
def submit(self, stream: str) -> Generator[str, None, None]:
    # 步骤 1: 调用父类，返回生成器
    response = super().submit(stream)
    
    # 步骤 2: 检查失败哨兵（pattern_matches 直接调用！）
    failed = self.pattern_matches(stream, self.sentinel, "failure_index")
    
    # 步骤 3: 失败检查（使用当前的 tried 值！）
    if self.tried and failed:
        raise ResponseNotAccepted(...)
    
    # 步骤 4: 设置 tried 标志（在检查之后！）
    if response:
        self.tried = True
    
    # 步骤 5: 返回生成器
    return response
```

### 1.2 关键洞察

| 发现 | 说明 |
|------|------|
| **失败检查在设置 `tried` 之前** | 步骤 3 在步骤 4 之前执行 |
| **sentinel 检查同步执行** | `pattern_matches(sentinel)` 在 `submit()` 返回前完成 |
| **pattern 检查延迟执行** | `pattern_matches(pattern)` 在生成器体中，Runner 迭代时才执行 |
| **生成器总是 truthy** | `if response:` 总是为 `True`，`tried` 总是被设置 |

### 1.3 不对称的调用时机

```
FailingResponder.submit(stream) 调用流程：

┌─────────────────────────────────────────────────────────────┐
│  步骤 1: response = super().submit(stream)                   │
│           ↓                                                    │
│           ┌─────────────────────────────────────────────┐   │
│           │ Responder.submit() 创建生成器                 │   │
│           │ 生成器体：                                      │   │
│           │   for _ in pattern_matches(pattern):        │   │
│           │       yield response                          │   │
│           │                                               │   │
│           │ ⚠️  pattern_matches 还没有被调用！            │   │
│           │ ⚠️  生成器只是被创建，还没有执行                │   │
│           └─────────────────────────────────────────────┘   │
│           ↓                                                    │
│           response = <generator object>                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 2: failed = pattern_matches(sentinel, failure_index)  │
│           ↓                                                    │
│           ┌─────────────────────────────────────────────┐   │
│           │ ✅ pattern_matches 被直接调用！               │   │
│           │    - 获取 failure_index                       │   │
│           │    - 扫描新增内容                              │   │
│           │    - 如果匹配，推进 failure_index              │   │
│           │    - 返回 matches                              │   │
│           └─────────────────────────────────────────────┘   │
│           ↓                                                    │
│           failed = True/False                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 3: if self.tried and failed:                           │
│           ↓                                                    │
│           ⚠️  使用检查前的 tried 值！                          │
│           ⚠️  首次调用时 self.tried = False！                  │
│           ↓                                                    │
│           False and failed = False                            │
│           ↓                                                    │
│           ❌ 不抛出异常！                                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 4: if response:                                         │
│           ↓                                                    │
│           response 是生成器对象                                │
│           生成器对象总是 truthy                                 │
│           ↓                                                    │
│           self.tried = True ⚠️ 在检查之后才设置！              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 5: return response                                      │
│           ↓                                                    │
│           返回生成器（还没有执行 pattern 检查！）                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 6: Runner 迭代生成器（在 respond() 中）                 │
│           ↓                                                    │
│           for response in watcher.submit(stream):             │
│               self.write_proc_stdin(response)                 │
│           ↓                                                    │
│           ┌─────────────────────────────────────────────┐   │
│           │ ⚠️ 现在生成器体才开始执行！                     │   │
│           │                                                 │   │
│           │ Responder.submit() 的生成器体：                 │   │
│           │   for _ in pattern_matches(pattern, index):  │   │
│           │       yield response                           │   │
│           │                                                 │   │
│           │ ✅ pattern_matches(pattern) 现在才被调用！       │   │
│           │ ✅ index 现在才可能被推进！                       │   │
│           └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.4 不对称性总结

| 操作 | pattern 检查 | sentinel 检查 |
|------|-------------|---------------|
| 调用时机 | 生成器迭代时（步骤 6） | `submit()` 返回前（步骤 2） |
| 索引推进时机 | 生成器迭代时 | `submit()` 返回前 |
| 对失败检查的影响 | 失败检查时还未执行 | 失败检查前已执行 |

---

## 2. 首次调用中的实际行为

### 2.1 场景 1：首次调用中只有 sentinel（没有 pattern）

```
配置：
- pattern = "Password:"
- sentinel = "Sorry, try again"
- 首次调用 stream = "Sorry, try again"

初始状态：
- tried = False
- index = 0
- failure_index = 0
```

**执行追踪：**

```
步骤 1: response = super().submit("Sorry, try again")
   → 创建生成器 G
   → 生成器体未执行
   → pattern_matches(pattern) 未调用
   → index 保持 0
   → response = <generator object>

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → failure_index = 0
   → new = "Sorry, try again"[0:] = "Sorry, try again"
   → re.findall("Sorry, try again", "Sorry, try again")
   → matches = ["Sorry, try again"]（非空）
   → matches 非空 → 推进索引
   → failure_index = 0 + len("Sorry, try again") = 18
   → failed = True（matches 非空）

步骤 3: if self.tried and failed:
   → self.tried = False（注意：步骤 4 还未执行！）
   → False and True = False
   → ❌ 不抛出异常！

步骤 4: if response:
   → response 是生成器对象
   → 生成器对象总是 truthy
   → self.tried = True

步骤 5: return response
   → 返回生成器 G

步骤 6: Runner 迭代 G
   → 生成器体开始执行
   → Responder.submit() 的生成器体：
   →   for _ in pattern_matches("Password:", "Sorry, try again", "index"):
   →       yield response
   →
   → pattern_matches(pattern, index) 被调用：
   →   index = 0
   →   new = "Sorry, try again"
   →   不匹配 "Password:"
   →   matches = []
   →   index 保持 0
   →
   → for 循环没有迭代
   → 没有 yield
   → 生成器结束
```

**最终状态：**
- `tried = True` ✅（已设置）
- `index = 0` ✅（未匹配，未推进）
- `failure_index = 18` ✅（已匹配，已推进）
- **没有抛出异常** ❌

### 2.2 场景 2：首次调用中 pattern 和 sentinel 都出现

```
配置：
- pattern = "Password:"
- sentinel = "Sorry, try again"
- 首次调用 stream = "Password: Sorry, try again"

初始状态：
- tried = False
- index = 0
- failure_index = 0
```

**执行追踪：**

```
步骤 1: response = super().submit("Password: Sorry, try again")
   → 创建生成器 G
   → 生成器体未执行
   → index 保持 0
   → response = <generator object>

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → failure_index = 0
   → new = "Password: Sorry, try again"
   → 匹配 "Sorry, try again"
   → matches = ["Sorry, try again"]
   → failure_index 推进到 len(stream) = 28
   → failed = True

步骤 3: if self.tried and failed:
   → self.tried = False
   → False and True = False
   → ❌ 不抛出异常！

步骤 4: if response:
   → self.tried = True

步骤 5: return G

步骤 6: Runner 迭代 G
   → 生成器体执行
   → pattern_matches("Password:", stream, "index"):
   →   index = 0
   →   new = "Password: Sorry, try again"
   →   匹配 "Password:"
   →   matches = ["Password:"]
   →   index 推进到 28
   → for 循环迭代一次
   → yield "secret\n"
   → Runner 写入 stdin
```

**最终状态：**
- `tried = True` ✅
- `index = 28` ✅
- `failure_index = 28` ✅
- **没有抛出异常** ❌

**关键问题：**
- 这是一个**密码错误**的场景！
- 用户输入了错误的密码
- 程序输出了 "Sorry, try again"
- 但 `FailingResponder` **没有抛出异常**！
- 这是一个**静默失效**的 bug！

### 2.3 场景 3：首次调用中只有 pattern（没有 sentinel）

```
配置：
- pattern = "Password:"
- sentinel = "Sorry, try again"
- 首次调用 stream = "Password: "

初始状态：
- tried = False
- index = 0
- failure_index = 0
```

**执行追踪：**

```
步骤 1: response = super().submit("Password: ")
   → 创建生成器 G
   → index 保持 0

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → new = "Password: "
   → 不匹配 "Sorry, try again"
   → matches = []
   → failure_index 保持 0
   → failed = False

步骤 3: if self.tried and failed:
   → self.tried = False
   → False and False = False
   → 不抛出异常

步骤 4: if response:
   → self.tried = True

步骤 5: return G

步骤 6: Runner 迭代 G
   → pattern_matches("Password:", "Password: ", "index"):
   →   匹配成功
   →   index 推进到 10
   → yield "secret\n"
   → 写入 stdin
```

**最终状态：**
- `tried = True` ✅
- `index = 10` ✅
- `failure_index = 0` ✅（未匹配，未推进）
- 没有抛出异常 ✅（预期行为）

---

## 3. 哨兵索引推进的后果

### 3.1 pattern_matches 的索引更新规则

```python
# watchers.py:103-104
if matches:  # ⚠️ 只有匹配时才更新索引！
    setattr(self, index_attr, index + len(new))
```

**规则：**
1. 只扫描 `stream[index:]`（新增内容）
2. 如果匹配成功，将 `index` 推进到 `index + len(new)`（流末尾）
3. 如果匹配失败，`index` 保持不变

### 3.2 首次调用中哨兵已推进的后果

**场景：首次调用中只有 sentinel，没有 pattern**

```
首次调用后：
- tried = True（误判）
- failure_index = 18（已推进到流末尾）

后续调用：
- stream = "Sorry, try again"（缓冲区内容不变）
- 或者 stream = "Sorry, try again" + "一些新增内容"
```

**子场景 3.2.1：后续调用中没有新增内容**

```
第二次调用 stream = "Sorry, try again"：

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → failure_index = 18
   → new = "Sorry, try again"[18:] = ""
   → re.findall("Sorry, try again", "")
   → matches = []
   → matches 为空 → 不推进索引
   → failed = False

步骤 3: if self.tried and failed:
   → self.tried = True
   → True and False = False
   → 不抛出异常！
```

**结果：**
- `failure_index` 已推进到流末尾
- 后续调用中 `new = ""`（空字符串）
- 永远不会匹配到 sentinel
- **永远不会抛出异常！静默失效！**

**子场景 3.2.2：后续调用中有新增内容，但不包含 sentinel**

```
第二次调用 stream = "Sorry, try again Some other output"：

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → failure_index = 18
   → new = " Some other output"
   → 不匹配 "Sorry, try again"
   → matches = []
   → failed = False

步骤 3: if self.tried and failed:
   → True and False = False
   → 不抛出异常
```

**结果：** 仍然静默失效

**子场景 3.2.3：后续调用中有新增内容，且包含 sentinel**

```
第二次调用 stream = "Sorry, try again Sorry, try again"：

步骤 2: failed = pattern_matches(sentinel, failure_index)
   → failure_index = 18
   → new = " Sorry, try again"
   → 匹配 "Sorry, try again"
   → matches = ["Sorry, try again"]
   → failure_index 推进到 len(stream)
   → failed = True

步骤 3: if self.tried and failed:
   → self.tried = True（首次调用后已设置）
   → True and True = True
   → ✅ 抛出 ResponseNotAccepted！
```

**结果：** 只有在这种场景下才会抛出异常！

### 3.3 结论：误报路径的实际可达性

**误报路径（之前报告中认为的）：**
> 无匹配的调用 → tried 被误设为 True → 后续调用中 sentinel → 误报

**实际可达性分析：**

| 场景 | 首次调用 | 后续调用 | 结果 |
|------|---------|---------|------|
| 场景 A | 只有 sentinel | 无新增内容 | ❌ 静默失效（索引已推进） |
| 场景 B | 只有 sentinel | 新增内容无 sentinel | ❌ 静默失效 |
| 场景 C | 只有 sentinel | 新增内容有 sentinel | ⚠️ **误报！** |
| 场景 D | pattern 匹配成功 | 新增内容有 sentinel | ✅ **正确报错！** |

**关键发现：**

1. **误报路径只有在特定条件下才可达：**
   - 首次调用中 sentinel 出现（触发 `failure_index` 推进）
   - 首次调用中没有 pattern 匹配（或匹配失败）
   - 后续调用中**新增内容**再次出现 sentinel

2. **如果 sentinel 只在首次调用中出现，后续调用中没有：**
   - `failure_index` 已推进
   - 永远不会再次匹配
   - **静默失效！**

3. **如果 pattern 在首次调用中匹配成功，后续调用中 sentinel 出现在新增内容：**
   - 这实际上是**正确的报错路径**！
   - 因为确实提交了响应，然后出现了错误提示

---

## 4. 纠正后的场景矩阵

### 4.1 核心问题重述

**原设计意图：**
```
期望行为：
1. 匹配 pattern → 提交响应 → 标记 tried = True
2. 如果 tried = True 且检测到 sentinel → 抛出异常

期望顺序：
submit1: 匹配 pattern → 提交响应 → tried = True
submit2: 检测 sentinel → tried=True → 抛出异常
```

**实际行为（当前实现）：**
```
实际顺序：
submit:
  1. 检测 sentinel（同步执行）
  2. 检查 tried and failed（tried 还是旧值！）
  3. 设置 tried = True（因为生成器 truthy）
  4. 返回生成器
  5. Runner 迭代生成器 → 检测 pattern（延迟执行）
```

### 4.2 完整场景矩阵

#### 场景 1：单流，pattern 和 sentinel 分两次调用

```
配置：
- 流：stdout（单流）
- pattern = "Password:"
- sentinel = "Sorry, try again"

时间线：
第 1 次 submit: "Password:"
第 2 次 submit: "Password: Sorry, try again"
```

**第 1 次 submit("Password:")：**
```
步骤 2: pattern_matches(sentinel)
   → 不匹配 "Sorry, try again"
   → failed = False
   → failure_index = 0

步骤 3: if self.tried and failed:
   → tried = False
   → False and False = False
   → 不抛出

步骤 4: if response:
   → 生成器 truthy
   → tried = True

步骤 6: 迭代生成器
   → pattern_matches(pattern)
   → 匹配 "Password:"
   → index = 10
   → yield 密码
   → 写入 stdin
```

**第 2 次 submit("Password: Sorry, try again")：**
```
步骤 2: pattern_matches(sentinel, failure_index=0)
   → new = "Password: Sorry, try again"
   → 匹配 "Sorry, try again"
   → failure_index 推进到 28
   → failed = True

步骤 3: if self.tried and failed:
   → tried = True（第 1 次后设置）
   → True and True = True
   → ✅ 抛出 ResponseNotAccepted！
```

**结果：** ✅ **正常工作**（预期行为）

---

#### 场景 2：单流，pattern 和 sentinel 在同一次调用

```
配置：
- 流：stdout（单流）
- pattern = "Password:"
- sentinel = "Sorry, try again"

时间线：
第 1 次 submit: "Password: Sorry, try again"
```

**第 1 次 submit：**
```
步骤 2: pattern_matches(sentinel)
   → 匹配 "Sorry, try again"
   → failure_index 推进到 28
   → failed = True

步骤 3: if self.tried and failed:
   → tried = False（还没设置！）
   → False and True = False
   → ❌ 不抛出异常！

步骤 4: if response:
   → tried = True

步骤 6: 迭代生成器
   → pattern_matches(pattern)
   → 匹配 "Password:"
   → index 推进到 28
   → yield 密码
   → 写入 stdin
```

**结果：** ❌ **静默失效！**（密码错误但不报错）

---

#### 场景 3：跨流，pattern 在 stdout，sentinel 在 stderr（首次调用）

```
配置：
- pty = False（双流）
- pattern = "Password:" (stdout)
- sentinel = "Sorry, try again" (stderr)

时间线：
stdout 线程 submit: "Password:"
stderr 线程 submit: "Sorry, try again"
```

**stdout 线程 submit("Password:")：**
```
步骤 2: pattern_matches(sentinel)
   → 不匹配
   → failed = False
   → failure_index (stdout副本) = 0

步骤 3: if self.tried and failed:
   → tried (stdout副本) = False
   → 不抛出

步骤 4: if response:
   → tried (stdout副本) = True

步骤 6: 迭代生成器
   → 匹配 pattern
   → index (stdout副本) 推进
   → yield 密码
   → 写入 stdin
```

**stderr 线程 submit("Sorry, try again")：**
```
步骤 2: pattern_matches(sentinel)
   → 匹配
   → failure_index (stderr副本) 推进到 18
   → failed = True

步骤 3: if self.tried and failed:
   → tried (stderr副本) = False ⚠️ 隔离！
   → False and True = False
   → ❌ 不抛出异常！

步骤 4: if response:
   → 生成器 truthy
   → tried (stderr副本) = True

步骤 6: 迭代生成器
   → 不匹配 pattern
   → 没有 yield
```

**结果：** ❌ **静默失效！**（跨流隔离 + 首次调用时序）

---

#### 场景 4：跨流，pattern 在 stdout，sentinel 在 stderr（后续调用）

```
配置：
- pty = False（双流）
- pattern = "Password:" (stdout)
- sentinel = "Sorry, try again" (stderr)

时间线：
第 1 轮：
  stdout 线程 submit: "Password:"
  stderr 线程 submit: "System info..." （无 sentinel）

第 2 轮：
  stdout 线程 submit: "Password: " （缓冲区不变或新增）
  stderr 线程 submit: "System info... Sorry, try again" （新增 sentinel）
```

**第 1 轮 stderr 线程 submit("System info...")：**
```
步骤 2: pattern_matches(sentinel)
   → 不匹配
   → failed = False
   → failure_index (stderr副本) = 0

步骤 4: if response:
   → 生成器 truthy
   → tried (stderr副本) = True ⚠️ 误判！
```

**第 2 轮 stderr 线程 submit("System info... Sorry, try again")：**
```
步骤 2: pattern_matches(sentinel, failure_index=0)
   → new = "System info... Sorry, try again"
   → 匹配
   → failure_index (stderr副本) 推进
   → failed = True

步骤 3: if self.tried and failed:
   → tried (stderr副本) = True（第 1 轮后误判设置）
   → True and True = True
   → ⚠️ **误报！** 抛出 ResponseNotAccepted！
```

**结果：** ⚠️ **误报！**（跨流隔离 + tried 误判）

---

#### 场景 5：pty 模式（单流）

```
配置：
- pty = True（单流，stdout + stderr 合并）
- pattern = "Password:"
- sentinel = "Sorry, try again"

时间线：
第 1 次 submit: "Password:"
第 2 次 submit: "Password: Sorry, try again"
```

**pty 模式特点：**
- 只有一个输出流
- 只有一个线程调用 `submit()`
- `threading.local` 的隔离特性无法触发

**行为与场景 1 相同：**
- 如果 pattern 和 sentinel 分两次调用 → ✅ 正常工作
- 如果 pattern 和 sentinel 在同一次调用 → ❌ 静默失效

---

### 4.3 场景矩阵总结

| 场景 | 模式 | 调用时序 | 结果 | 原因 |
|------|------|---------|------|------|
| 1 | 单流 | pattern 和 sentinel 分两次调用 | ✅ 正常 | 第 2 次时 tried=True |
| 2 | 单流 | pattern 和 sentinel 同一次调用 | ❌ 静默失效 | 检查时 tried=False |
| 3 | 跨流 | pattern 在 stdout，sentinel 在 stderr | ❌ 静默失效 | 跨流隔离 + 首次调用时序 |
| 4 | 跨流 | 后续调用中 sentinel 出现在 stderr | ⚠️ 误报 | 跨流隔离 + tried 误判 |
| 5 | pty 模式 | 分两次调用 | ✅ 正常 | 单流 + 时序正确 |
| 6 | pty 模式 | 同一次调用 | ❌ 静默失效 | 检查时 tried=False |

---

## 5. 与之前报告的差异总结

### 5.1 关键纠正

| 问题 | 之前报告（错误） | 本次分析（正确） |
|------|-----------------|------------------|
| **失败检查位置** | 认为在设置 `tried` 之后 | **实际在设置 `tried` 之前** |
| **首次调用行为** | 认为可能误报 | **首次调用永远不会抛出异常！** |
| **静默失效场景** | 认为只有跨流场景 | **同流同次调用也是静默失效！** |
| **误报可达性** | 认为较容易触发 | **只有特定时序下才触发** |
| **pattern_matches 时机** | 认为都是同步的 | **pattern 检查在生成器迭代时，sentinel 检查同步** |

### 5.2 核心 Bug 的重新定义

**之前认为的核心 Bug：**
1. `if response:` 中生成器 truthy 问题 → `tried` 总是被设置
2. `threading.local` 跨流隔离问题

**实际核心 Bug：**

```
⭐ Bug 1: 失败检查在设置 tried 之前执行

当前顺序：
  1. 检测 sentinel
  2. 检查 tried and failed  ← tried 还是旧值！
  3. 设置 tried = True
  4. 检测 pattern（延迟）

正确顺序应该是：
  1. 检测 pattern，提交响应
  2. 如果提交了响应，设置 tried = True
  3. 检测 sentinel
  4. 检查 tried and failed
```

```
⭐ Bug 2: pattern 和 sentinel 检查时机不对称

- sentinel 检查在 submit() 返回前同步执行
- pattern 检查在生成器迭代时延迟执行

这导致：
- 失败检查时，pattern 可能还没有被检测
- tried 可能还没有被正确设置（即使 pattern 会匹配）
```

### 5.3 误报路径的实际可达性

**之前认为：**
> 无匹配的调用 → tried 被误设为 True → 后续调用中 sentinel → 误报

**实际情况：**

| 条件 | 说明 |
|------|------|
| 必要条件 1 | 某线程的 `submit()` 中没有 pattern 匹配（或不关心） |
| 必要条件 2 | 该线程的 `submit()` 被调用至少一次（设置 `tried = True`） |
| 必要条件 3 | 后续调用中，**新增内容**包含 sentinel |
| 必要条件 4 | 该线程之前没有检测过这个 sentinel（或 `failure_index` 没有覆盖它） |

**最可能触发误报的场景：**

```
跨流场景：
- stdout 线程：匹配 pattern → 提交密码 → tried=True
- stderr 线程：第 1 次 submit("System info...")
  → 没有 pattern 匹配
  → 生成器 truthy → tried=True ⚠️ 误判
  → 没有 sentinel → failure_index 保持 0
- stderr 线程：第 2 次 submit("System info... Sorry, try again")
  → 新增内容包含 sentinel
  → failure_index=0 → 匹配整个流
  → failed=True
  → tried=True ⚠️ 误判的值
  → 抛出异常 ⚠️ 误报！
```

### 5.4 最严重的问题：静默失效

**最严重的 Bug 是静默失效，不是误报：**

| 场景 | 发生概率 | 影响 |
|------|---------|------|
| 同流同次调用（密码错误） | 中等 | ❌ 静默失效，密码错误不报错 |
| 跨流场景（pattern 在 stdout，sentinel 在 stderr） | 高（取决于 sudo 实现） | ❌ 静默失效 |
| 误报场景 | 低 | ⚠️ 误报，但至少报错了 |

**静默失效的后果：**
- 密码错误时，程序继续执行
- 用户不知道密码错误
- 可能导致安全问题或数据损坏

---

## 附录：关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| FailingResponder.submit() 完整实现 | watchers.py | 130-145 |
| 失败检查 | watchers.py | 136-140 |
| tried 标志设置 | watchers.py | 142-143 |
| pattern_matches 索引更新 | watchers.py | 103-104 |
| Runner.respond() 迭代生成器 | runners.py | 932-957 |
| Responder.submit() 生成器 | watchers.py | 107-110 |

---

## 总结

### 关键纠正

1. **失败检查在设置 `tried` 之前执行！**
   - 这是最关键的发现
   - 意味着**首次调用中永远不会抛出异常**
   - 即使 pattern 和 sentinel 同时出现，也不会报错

2. **pattern 和 sentinel 检查时机不对称：**
   - `sentinel` 检查在 `submit()` 返回前同步执行
   - `pattern` 检查在生成器迭代时延迟执行
   - 这加剧了时序问题

3. **静默失效是更严重的问题：**
   - 同流同次调用：静默失效
   - 跨流场景：静默失效
   - 这些是实际使用中最可能遇到的场景

4. **误报路径实际可达性较低：**
   - 需要特定的时序条件
   - 至少会抛出异常（虽然是误报）
   - 比静默失效好一些

### 修复建议

**核心修复：调整执行顺序**

```python
# 当前实现（错误）：
def submit(self, stream):
    response = super().submit(stream)      # 1. 创建生成器
    failed = pattern_matches(sentinel)     # 2. 检查 sentinel
    if self.tried and failed:               # 3. 检查（tried 还是旧值）
        raise ResponseNotAccepted
    if response:                            # 4. 设置 tried
        self.tried = True
    return response                         # 5. 返回生成器

# 修复后：
def submit(self, stream):
    # 先执行 pattern 检查，获取实际响应
    responses = list(super().submit(stream))  # 1. 消耗生成器，执行 pattern 检查
    
    # 如果有实际响应，设置 tried
    if responses:                             # 2. 检查实际响应，不是生成器
        self.tried = True
    
    # 然后检查 sentinel
    failed = pattern_matches(sentinel)       # 3. 检查 sentinel
    
    # 最后检查失败条件
    if self.tried and failed:                 # 4. 检查（tried 已正确设置）
        raise ResponseNotAccepted
    
    # 返回响应
    yield from responses
```

这个修复：
1. ✅ 解决了时序问题（先设置 tried，再检查失败）
2. ✅ 解决了生成器 truthy 问题（检查实际响应列表）
3. ✅ 解决了检查时机不对称问题（都同步执行）
