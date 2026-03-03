---
name: writing-plans
description: 在有规范或多步骤任务需求时使用，在触碰代码之前执行
---

# 编写计划

## 概述

编写全面的实现计划，假设工程师对我们的代码库毫无背景知识，且品味存疑。记录他们需要知道的一切：每个任务需要修改哪些文件、代码、测试、可能需要查阅的文档，以及如何测试。将整个计划拆分为小而可操作的任务。遵循 DRY、YAGNI、TDD 原则。频繁提交。

假设他们是有经验的开发者，但对我们的工具集或问题领域几乎一无所知。假设他们对良好的测试设计了解不多。

**在开始时宣告：** "我正在使用 writing-plans 技能来创建实现计划。"

**背景：** 此技能应在专用工作树中运行（由头脑风暴技能创建）。

**计划保存位置：** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## 可操作任务的粒度

**每个步骤是一个动作（2-5 分钟）：**
- "编写失败测试"——一个步骤
- "运行它以确认其失败"——一个步骤
- "实现最少代码使测试通过"——一个步骤
- "运行测试并确认其通过"——一个步骤
- "提交"——一个步骤

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [功能名称] 实现计划

> **致 Claude：** 必要子技能：使用 superpowers:executing-plans 逐任务实现此计划。

**目标：** [一句话描述要构建的内容]

**架构：** [2-3 句话描述方法]

**技术栈：** [关键技术/库]

---
```

## 任务结构

````markdown
### 任务 N：[组件名称]

**文件：**
- 创建：`exact/path/to/file.py`
- 修改：`exact/path/to/existing.py:123-145`
- 测试：`tests/exact/path/to/test.py`

**步骤 1：编写失败测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**步骤 2：运行测试以验证其失败**

运行：`pytest tests/path/test.py::test_name -v`
预期：FAIL，提示"function not defined"

**步骤 3：编写最少实现**

```python
def function(input):
    return expected
```

**步骤 4：运行测试以验证其通过**

运行：`pytest tests/path/test.py::test_name -v`
预期：PASS

**步骤 5：提交**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 注意事项
- 始终使用精确的文件路径
- 计划中包含完整代码（不要只写"添加验证"）
- 精确的命令及预期输出
- 使用 @ 语法引用相关技能
- DRY、YAGNI、TDD，频繁提交

## 执行交接

保存计划后，提供执行选择：

**"计划已完成并保存至 `docs/plans/<filename>.md`。两种执行选项：**

**1. 子代理驱动（本会话）** - 我为每个任务派发新的子代理，任务间进行审查，快速迭代

**2. 并行会话（单独）** - 打开带有 executing-plans 的新会话，批量执行并设置检查点

**选择哪种方式？"**

**如果选择子代理驱动：**
- **必要子技能：** 使用 superpowers:subagent-driven-development
- 保持在本会话
- 每个任务一个新子代理 + 代码审查

**如果选择并行会话：**
- 引导他们在工作树中打开新会话
- **必要子技能：** 新会话使用 superpowers:executing-plans
