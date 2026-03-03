# Svelte 待办事项列表 - 设计文档

## 概述

一个使用 Svelte 构建的简单待办事项列表应用。支持创建、完成和删除待办事项，并具备 localStorage 持久化功能。

## 功能特性

- 添加新的待办事项
- 将待办事项标记为完成/未完成
- 删除待办事项
- 按以下条件筛选：全部 / 进行中 / 已完成
- 清除所有已完成的待办事项
- 持久化到 localStorage
- 显示剩余事项数量

## 用户界面

```
┌─────────────────────────────────────────┐
│  Svelte Todos                           │
├─────────────────────────────────────────┤
│  [________________________] [Add]       │
├─────────────────────────────────────────┤
│  [ ] Buy groceries                  [x] │
│  [✓] Walk the dog                   [x] │
│  [ ] Write code                     [x] │
├─────────────────────────────────────────┤
│  2 items left                           │
│  [All] [Active] [Completed]  [Clear ✓]  │
└─────────────────────────────────────────┘
```

## 组件

```
src/
  App.svelte           # 主应用，状态管理
  lib/
    TodoInput.svelte   # 文本输入框 + 添加按钮
    TodoList.svelte    # 列表容器
    TodoItem.svelte    # 单个待办事项（含复选框、文本、删除）
    FilterBar.svelte   # 筛选按钮 + 清除已完成
    store.ts           # 待办事项的 Svelte 存储
    storage.ts         # localStorage 持久化
```

## 数据模型

```typescript
interface Todo {
  id: string;        // UUID
  text: string;      // 待办事项文本
  completed: boolean;
}

type Filter = 'all' | 'active' | 'completed';
```

## 验收标准

1. 可通过输入文字并按 Enter 键或点击添加按钮来新增待办事项
2. 可通过点击复选框切换待办事项的完成状态
3. 可通过点击 X 按钮删除待办事项
4. 筛选按钮显示正确的待办事项子集
5. "X items left" 显示未完成待办事项的数量
6. "Clear completed" 删除所有已完成的待办事项
7. 待办事项在页面刷新后持久保留（localStorage）
8. 空状态显示有帮助的提示信息
9. 所有测试通过
