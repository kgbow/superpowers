# 代码质量审查者提示模板

派发代码质量审查者子智能体时使用此模板。

**目的：** 验证实现构建良好（整洁、经过测试、可维护）

**仅在规格符合性审查通过后派发。**

```
Task tool (superpowers:code-reviewer):
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [来自实现者的报告]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [任务开始前的提交]
  HEAD_SHA: [当前提交]
  DESCRIPTION: [任务摘要]
```

**代码审查者返回：** 优点、问题（严重/重要/次要）、评估结论
