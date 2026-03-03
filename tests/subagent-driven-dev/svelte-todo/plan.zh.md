# Svelte 待办事项列表 - 实施计划

使用 `superpowers:subagent-driven-development` 技能执行此计划。

## 背景

使用 Svelte 构建一个待办事项列表应用。完整规格说明请参阅 `design.md`。

## 任务

### 任务 1：项目初始化

使用 Vite 创建 Svelte 项目。

**执行：**
- 运行 `npm create vite@latest . -- --template svelte-ts`
- 使用 `npm install` 安装依赖
- 验证开发服务器正常运行
- 清理 App.svelte 中的默认 Vite 模板内容

**验证：**
- `npm run dev` 启动服务器
- 应用显示最小化的 "Svelte Todos" 标题
- `npm run build` 成功执行

---

### 任务 2：待办事项状态存储

创建用于待办事项状态管理的 Svelte 存储。

**执行：**
- 创建 `src/lib/store.ts`
- 定义包含 id、text、completed 字段的 `Todo` 接口
- 创建初始为空数组的可写存储
- 导出函数：`addTodo(text)`、`toggleTodo(id)`、`deleteTodo(id)`、`clearCompleted()`
- 创建 `src/lib/store.test.ts`，为每个函数编写测试

**验证：**
- 测试通过：`npm run test`（如需要，安装 vitest）

---

### 任务 3：localStorage 持久化

为待办事项添加持久化层。

**执行：**
- 创建 `src/lib/storage.ts`
- 实现 `loadTodos(): Todo[]` 和 `saveTodos(todos: Todo[])`
- 优雅处理 JSON 解析错误（返回空数组）
- 与存储集成：初始化时加载，变更时保存
- 添加加载/保存/错误处理的测试

**验证：**
- 测试通过
- 手动测试：添加待办事项，刷新页面，待办事项持久存在

---

### 任务 4：TodoInput 组件

创建用于添加待办事项的输入组件。

**执行：**
- 创建 `src/lib/TodoInput.svelte`
- 文本输入框绑定到本地状态
- 添加按钮调用 `addTodo()` 并清空输入框
- 按下 Enter 键也可提交
- 输入框为空时禁用添加按钮
- 添加组件测试

**验证：**
- 测试通过
- 组件渲染输入框和按钮

---

### 任务 5：TodoItem 组件

创建单个待办事项组件。

**执行：**
- 创建 `src/lib/TodoItem.svelte`
- 属性：`todo: Todo`
- 复选框切换完成状态（调用 `toggleTodo`）
- 已完成时文本显示删除线
- 删除按钮（X）调用 `deleteTodo`
- 添加组件测试

**验证：**
- 测试通过
- 组件渲染复选框、文本和删除按钮

---

### 任务 6：TodoList 组件

创建列表容器组件。

**执行：**
- 创建 `src/lib/TodoList.svelte`
- 属性：`todos: Todo[]`
- 为每个待办事项渲染 TodoItem
- 为空时显示 "No todos yet"
- 添加组件测试

**验证：**
- 测试通过
- 组件渲染 TodoItem 列表

---

### 任务 7：FilterBar 组件

创建筛选器和状态栏组件。

**执行：**
- 创建 `src/lib/FilterBar.svelte`
- 属性：`todos: Todo[]`、`filter: Filter`、`onFilterChange: (f: Filter) => void`
- 显示计数："X items left"（未完成数量）
- 三个筛选按钮：All（全部）、Active（进行中）、Completed（已完成）
- 当前激活的筛选器在视觉上高亮显示
- "Clear completed" 按钮（没有已完成待办事项时隐藏）
- 添加组件测试

**验证：**
- 测试通过
- 组件渲染计数、筛选器和清除按钮

---

### 任务 8：应用集成

在 App.svelte 中将所有组件连接起来。

**执行：**
- 导入所有组件和存储
- 添加筛选器状态（默认值：'all'）
- 根据筛选器状态计算已筛选的待办事项
- 渲染：标题、TodoInput、TodoList、FilterBar
- 向每个组件传递适当的属性

**验证：**
- 应用渲染所有组件
- 添加待办事项正常工作
- 切换状态正常工作
- 删除功能正常工作

---

### 任务 9：筛选功能

确保筛选功能端到端正常工作。

**执行：**
- 验证筛选按钮改变显示的待办事项
- 'all' 显示所有待办事项
- 'active' 仅显示未完成的待办事项
- 'completed' 仅显示已完成的待办事项
- "清除已完成" 删除已完成的待办事项，必要时重置筛选器
- 添加集成测试

**验证：**
- 筛选测试通过
- 手动验证所有筛选状态

---

### 任务 10：样式与优化

添加 CSS 样式以提升可用性。

**执行：**
- 为应用设置样式以匹配设计稿
- 已完成的待办事项显示删除线和柔和颜色
- 激活的筛选按钮高亮显示
- 输入框具有聚焦样式
- 删除按钮在悬停时显示（或在移动端始终显示）
- 响应式布局

**验证：**
- 应用在视觉上可用
- 样式不影响功能

---

### 任务 11：端到端测试

添加 Playwright 测试以覆盖完整用户流程。

**执行：**
- 安装 Playwright：`npm init playwright@latest`
- 创建 `tests/todo.spec.ts`
- 测试以下流程：
  - 添加待办事项
  - 完成待办事项
  - 删除待办事项
  - 筛选待办事项
  - 清除已完成项
  - 持久化（添加、重新加载、验证）

**验证：**
- `npx playwright test` 通过

---

### 任务 12：README

记录项目文档。

**执行：**
- 创建 `README.md`，包含以下内容：
  - 项目描述
  - 环境搭建：`npm install`
  - 开发：`npm run dev`
  - 测试：`npm test` 和 `npx playwright test`
  - 构建：`npm run build`

**验证：**
- README 准确描述了该项目
- 说明中的指令实际可运行
