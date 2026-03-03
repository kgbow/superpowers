---
name: finishing-a-development-branch
description: 在实现完成、所有测试通过、需要决定如何整合工作时使用——通过呈现合并、创建 PR 或清理的结构化选项来指导完成开发工作
---

# 完成开发分支

## 概述

通过呈现清晰的选项并处理所选工作流程，指导完成开发工作。

**核心原则：** 验证测试 → 呈现选项 → 执行选择 → 清理。

**开始时宣布：** "我正在使用 finishing-a-development-branch 技能来完成这项工作。"

## 流程

### 第 1 步：验证测试

**在呈现选项之前，验证测试是否通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。完成之前必须修复：

[显示失败详情]

在测试通过之前无法继续合并/创建 PR。
```

停止。不要继续第 2 步。

**如果测试通过：** 继续第 2 步。

### 第 2 步：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："这个分支是从 main 分出来的——是这样吗？"

### 第 3 步：呈现选项

精确呈现以下 4 个选项：

```
实现已完成。你想怎么做？

1. 本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支现状（我稍后处理）
4. 丢弃这项工作

选择哪个选项？
```

**不要添加说明**——保持选项简洁。

### 第 4 步：执行选择

#### 选项 1：本地合并

```bash
# 切换到基础分支
git checkout <base-branch>

# 拉取最新代码
git pull

# 合并功能分支
git merge <feature-branch>

# 验证合并结果上的测试
<test command>

# 如果测试通过
git branch -d <feature-branch>
```

然后：清理工作树（第 5 步）

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

然后：清理工作树（第 5 步）

#### 选项 3：保持现状

报告："保持分支 <name>。工作树保留在 <path>。"

**不要清理工作树。**

#### 选项 4：丢弃

**先进行确认：**
```
这将永久删除：
- 分支 <name>
- 所有提交：<commit-list>
- 位于 <path> 的工作树

输入 'discard' 以确认。
```

等待精确的确认输入。

如果已确认：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

然后：清理工作树（第 5 步）

### 第 5 步：清理工作树

**针对选项 1、2、4：**

检查是否在工作树中：
```bash
git worktree list | grep $(git branch --show-current)
```

如果是：
```bash
git worktree remove <worktree-path>
```

**针对选项 3：** 保留工作树。

## 快速参考

| 选项 | 合并 | 推送 | 保留工作树 | 清理分支 |
|------|------|------|-----------|---------|
| 1. 本地合并 | ✓ | - | - | ✓ |
| 2. 创建 PR | - | ✓ | ✓ | - |
| 3. 保持现状 | - | - | ✓ | - |
| 4. 丢弃 | - | - | - | ✓（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并了有问题的代码，创建了失败的 PR
- **修复：** 在提供选项之前始终验证测试

**开放式问题**
- **问题：** "我接下来应该怎么做？" → 模糊不清
- **修复：** 精确呈现 4 个结构化选项

**自动清理工作树**
- **问题：** 在可能仍需要工作树时将其删除（选项 2、3）
- **修复：** 仅对选项 1 和 4 进行清理

**丢弃时不进行确认**
- **问题：** 意外删除工作
- **修复：** 要求输入 "discard" 进行确认

## 危险信号

**永远不要：**
- 在测试失败时继续
- 在未验证合并结果测试的情况下合并
- 在未确认的情况下删除工作
- 在未明确请求的情况下强制推送

**始终要：**
- 在提供选项之前验证测试
- 精确呈现 4 个选项
- 对选项 4 要求输入确认文字
- 仅对选项 1 和 4 清理工作树

## 集成

**被以下技能调用：**
- **subagent-driven-development**（第 7 步）——所有任务完成后
- **executing-plans**（第 5 步）——所有批次完成后

**配合使用：**
- **using-git-worktrees**——清理由该技能创建的工作树
