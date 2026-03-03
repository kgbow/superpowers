# Superpowers

Superpowers 是一套完整的软件开发工作流，专为你的编程智能体设计，构建于一组可组合的"技能（skills）"之上，并附带初始指令，确保智能体正确使用这些技能。

## 工作原理

一切从你启动编程智能体的那一刻开始。当它发现你在构建某个东西时，它*不会*直接跳进去写代码。相反，它会退一步，询问你真正想做什么。

在通过对话梳理出规格说明后，它会以足够短小、便于阅读和消化的段落形式展示给你。

在你确认设计方案后，智能体会整理出一份实现计划，清晰到足以让一位充满热情却品味欠佳、缺乏判断力、没有项目背景、又不爱测试的初级工程师照着执行。该计划强调真正的红绿灯测试驱动开发（TDD）、YAGNI（你不会需要它）和 DRY（不要重复自己）。

接下来，一旦你说"开始"，它就会启动一个*子智能体驱动开发*流程，让多个智能体逐一完成各项工程任务，检查和审查它们的工作，并持续推进。Claude 能够连续自主工作数小时而不偏离你制定的计划，这并不罕见。

系统还有很多其他功能，但这就是核心所在。由于技能会自动触发，你无需做任何额外操作。你的编程智能体就这样拥有了超能力（Superpowers）。


## 赞助

如果 Superpowers 帮助你完成了能带来收益的工作，并且你有意愿的话，我将非常感激你考虑[赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse


## 安装

**注意：** 安装方式因平台而异。Claude Code 或 Cursor 有内置插件市场。Codex 和 OpenCode 需要手动配置。


### Claude Code（通过插件市场）

在 Claude Code 中，首先注册市场：

```bash
/plugin marketplace add obra/superpowers-marketplace
```

然后从该市场安装插件：

```bash
/plugin install superpowers@superpowers-marketplace
```

### Cursor（通过插件市场）

在 Cursor 智能体聊天中，从市场安装：

```text
/plugin-add superpowers
```

### Codex

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

**详细文档：** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

告诉 OpenCode：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
```

**详细文档：** [docs/README.opencode.md](docs/README.opencode.md)

### 验证安装

在你选择的平台中开启一个新会话，并请求一个应当触发技能的操作（例如，"帮我规划这个功能"或"让我们调试这个问题"）。智能体应该会自动调用相关的 superpowers 技能。

## 基本工作流

1. **brainstorming** - 在编写代码之前激活。通过提问精炼粗糙的想法，探索替代方案，分段展示设计以供验证。保存设计文档。

2. **using-git-worktrees** - 在设计获批后激活。在新分支上创建隔离的工作空间，运行项目设置，验证测试基线是否干净。

3. **writing-plans** - 在设计获批时激活。将工作分解为小任务（每个 2-5 分钟）。每个任务包含精确的文件路径、完整代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 在计划完成时激活。为每个任务分派新的子智能体，进行两阶段审查（规格合规性检查，然后代码质量检查），或以批次方式执行并设置人工检查点。

5. **test-driven-development** - 在实现过程中激活。强制执行红-绿-重构流程：先写失败的测试，观察其失败，再写最少量的代码，观察其通过，然后提交。删除在测试之前写的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题将阻止进度推进。

7. **finishing-a-development-branch** - 在任务完成时激活。验证测试，提供选项（合并/PR/保留/丢弃），清理工作树。

**智能体在执行任何任务前都会检查相关技能。** 这是强制工作流，而非建议。

## 内容介绍

### 技能库

**测试**
- **test-driven-development** - 红-绿-重构循环（含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因分析流程（含 root-cause-tracing、defense-in-depth、condition-based-waiting 技术）
- **verification-before-completion** - 确保问题真正得到修复

**协作**
- **brainstorming** - 苏格拉底式设计精炼
- **writing-plans** - 详细的实现计划
- **executing-plans** - 带检查点的批量执行
- **dispatching-parallel-agents** - 并发子智能体工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段审查的快速迭代（规格合规性检查，然后代码质量检查）

**元技能**
- **writing-skills** - 按照最佳实践创建新技能（含测试方法）
- **using-superpowers** - 技能系统简介

## 理念

- **测试驱动开发** - 始终先写测试
- **系统化而非即兴** - 流程优于猜测
- **降低复杂度** - 以简洁为首要目标
- **证据优于断言** - 在宣布成功之前先验证

延伸阅读：[Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## 贡献

技能直接存放在本仓库中。贡献方式：

1. Fork 本仓库
2. 为你的技能创建一个分支
3. 遵循 `writing-skills` 技能来创建和测试新技能
4. 提交 PR

完整指南请参阅 `skills/writing-skills/SKILL.md`。

## 更新

在更新插件时，技能会自动更新：

```bash
/plugin update superpowers
```

## 许可证

MIT 许可证 - 详见 LICENSE 文件

## 支持

- **问题反馈**: https://github.com/obra/superpowers/issues
- **插件市场**: https://github.com/obra/superpowers-marketplace
