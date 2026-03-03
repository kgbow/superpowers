# 纵深防御验证

## 概述

当你修复一个由无效数据引起的漏洞时，在一个地方添加验证看起来已经足够。但这单一的检查可能被不同的代码路径、重构或 mock 绕过。

**核心原则：** 在数据流经的每一层都进行验证。使漏洞在结构上变得不可能出现。

## 为什么需要多层防御

单一验证："我们修复了这个漏洞"
多层防御："我们使这个漏洞变得不可能出现"

不同层级捕获不同类型的问题：
- 入口验证捕获大多数漏洞
- 业务逻辑捕获边界情况
- 环境守卫防止特定上下文中的危险操作
- 调试日志在其他层级失效时提供帮助

## 四个层级

### 第一层：入口点验证
**目的：** 在 API 边界拒绝明显无效的输入

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... 继续处理
}
```

### 第二层：业务逻辑验证
**目的：** 确保数据对于此操作合理

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... 继续处理
}
```

### 第三层：环境守卫
**目的：** 在特定上下文中防止危险操作

```typescript
async function gitInit(directory: string) {
  // 在测试中，拒绝在临时目录之外执行 git init
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... 继续处理
}
```

### 第四层：调试探针
**目的：** 捕获上下文以供取证分析

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 继续处理
}
```

## 应用此模式

发现漏洞时：

1. **追踪数据流** ——错误值从哪里产生？在哪里被使用？
2. **列出所有检查点** ——列出数据流经的每一个节点
3. **在每一层添加验证** ——入口、业务逻辑、环境、调试
4. **测试每一层** ——尝试绕过第一层，验证第二层能否捕获

## 来自实际会话的案例

漏洞：空 `projectDir` 导致 `git init` 在源代码中运行

**数据流：**
1. 测试设置 → 空字符串
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` 在 `process.cwd()` 中运行

**添加的四个层级：**
- 第一层：`Project.create()` 验证非空/存在/可写
- 第二层：`WorkspaceManager` 验证 projectDir 非空
- 第三层：`WorktreeManager` 在测试中拒绝在 tmpdir 之外执行 git init
- 第四层：git init 之前的堆栈跟踪日志

**结果：** 全部 1847 个测试通过，漏洞无法复现

## 关键洞察

四个层级全部是必要的。在测试期间，每个层级都捕获了其他层级遗漏的漏洞：
- 不同的代码路径绕过了入口验证
- Mock 绕过了业务逻辑检查
- 不同平台上的边界情况需要环境守卫
- 调试日志识别了结构性误用

**不要只在一个验证点停下来。** 在每一层都添加检查。
