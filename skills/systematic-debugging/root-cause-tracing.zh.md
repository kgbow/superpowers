# 根本原因追踪

## 概述

漏洞往往在调用堆栈的深处显现（git init 在错误目录中运行、文件创建在错误位置、数据库以错误路径打开）。你的本能是在错误出现的地方修复，但那只是在处理症状。

**核心原则：** 沿调用链反向追踪，直到找到最初的触发点，然后在根源处修复。

## 适用场景

```dot
digraph when_to_use {
    "漏洞出现在堆栈深处？" [shape=diamond];
    "能够反向追踪？" [shape=diamond];
    "在症状点修复" [shape=box];
    "追踪到最初触发点" [shape=box];
    "更好的做法：同时添加纵深防御" [shape=box];

    "漏洞出现在堆栈深处？" -> "能够反向追踪？" [label="是"];
    "能够反向追踪？" -> "追踪到最初触发点" [label="是"];
    "能够反向追踪？" -> "在症状点修复" [label="否——死路"];
    "追踪到最初触发点" -> "更好的做法：同时添加纵深防御";
}
```

**适用于以下情况：**
- 错误发生在执行深处（而非入口点）
- 堆栈跟踪显示较长的调用链
- 不清楚无效数据来自何处
- 需要找出是哪个测试/代码触发了问题

## 追踪流程

### 1. 观察症状
```
Error: git init failed in /Users/jesse/project/packages/core
```

### 2. 找到直接原因
**哪段代码直接导致了这个问题？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 追问：是什么调用了这里？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 由 Session.initializeWorkspace() 调用
  → 由 Session.create() 调用
  → 由测试中的 Project.create() 调用
```

### 4. 持续向上追踪
**传入的是什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 会解析为 `process.cwd()`
- 那是源代码目录！

### 5. 找到最初触发点
**空字符串从哪里来？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前访问！
```

## 添加堆栈跟踪

当无法手动追踪时，添加探针：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（而非 logger——logger 可能不会显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别规律（同一个测试？同一个参数？）

## 查找导致污染的测试

如果某个问题在测试期间出现，但不知道是哪个测试引起的：

使用本目录中的二分查找脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐一运行测试，在第一个污染测试处停止。参见脚本说明以了解用法。

## 实际案例：空 projectDir

**症状：** `.git` 创建在 `packages/core/`（源代码目录）

**追踪链：**
1. `git init` 在 `process.cwd()` 中运行 ← cwd 参数为空
2. WorktreeManager 以空 projectDir 被调用
3. Session.create() 传入了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始时返回 `{ tempDir: '' }`

**根本原因：** 顶层变量初始化时访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**同时添加了纵深防御：**
- 第一层：Project.create() 验证目录
- 第二层：WorkspaceManager 验证非空
- 第三层：NODE_ENV 守卫，拒绝在 tmpdir 之外执行 git init
- 第四层：git init 之前的堆栈跟踪日志

## 关键原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "能再向上追踪一层？" [shape=diamond];
    "向上回溯" [shape=box];
    "这是根源吗？" [shape=diamond];
    "在根源处修复" [shape=box];
    "在每一层添加验证" [shape=box];
    "漏洞变得不可能出现" [shape=doublecircle];
    "绝不只修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "能再向上追踪一层？";
    "能再向上追踪一层？" -> "向上回溯" [label="是"];
    "能再向上追踪一层？" -> "绝不只修复症状" [label="否"];
    "向上回溯" -> "这是根源吗？";
    "这是根源吗？" -> "向上回溯" [label="否——继续追踪"];
    "这是根源吗？" -> "在根源处修复" [label="是"];
    "在根源处修复" -> "在每一层添加验证";
    "在每一层添加验证" -> "漏洞变得不可能出现";
}
```

**绝不只修复错误出现的地方。** 反向追踪以找到最初的触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而非 logger——logger 可能被抑制
**在操作之前：** 在危险操作之前记录日志，而非在失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 实际效果

来自调试会话（2025-10-03）：
- 通过 5 层追踪找到根本原因
- 在根源处修复（getter 验证）
- 添加了 4 层防御
- 1847 个测试通过，零污染
