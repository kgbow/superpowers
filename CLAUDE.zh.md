# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在处理本仓库代码时提供指导。

## 运行测试

```bash
# 运行快速技能测试（每个约 2 分钟）
cd tests/claude-code && ./run-skill-tests.sh

# 运行特定测试
./run-skill-tests.sh --test test-subagent-driven-development.sh

# 运行并输出详细信息
./run-skill-tests.sh --verbose

# 运行集成测试（10–30 分钟，需要 `claude` CLI）
./run-skill-tests.sh --integration
```

集成测试必须**从 superpowers 目录**运行（不能从临时目录运行），并且需要在 `~/.claude/settings.json` 中配置 `"superpowers@superpowers-dev": true`。

分析会话的令牌用量：
```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<encoded-path>/<session>.jsonl
```

## 架构

Superpowers 是一个 Claude Code 插件（同时兼容 Cursor/Codex/OpenCode），它将一套技能系统注入到编程智能体中。

### 技能加载方式

1. **SessionStart 钩子**（`hooks/session-start` 通过 `hooks/run-hook.cmd`）在每次会话时触发，并将 `skills/using-superpowers/SKILL.md` 的完整内容注入到智能体的上下文中。这一步骤引导智能体了解技能系统。
2. 智能体随后根据 `description` 字段与当前任务的匹配情况，使用 `Skill` 工具按需加载各个技能。
3. `lib/skills-core.js` 负责处理技能发现、前置元数据（frontmatter）解析和路径解析。`~/.claude/skills/` 中的个人技能会遮蔽同名的 superpowers 技能。

### 技能文件格式

每个技能位于 `skills/<name>/SKILL.md`，包含 YAML 前置元数据：

```yaml
---
name: skill-name-with-hyphens
description: Use when [triggering conditions, symptoms, context]
---
```

**`description` 字段的关键规则：**
- 必须以"Use when..."开头，且*仅描述触发条件*，不描述工作流程
- 在描述中概括工作流会导致智能体跳过阅读技能正文，而直接遵循描述中的捷径
- 前置元数据总长度最多 1024 个字符；描述保持在 500 个字符以内

支持文件（脚本、大型参考文档）放在同一技能目录下。跨技能引用只使用名称（`superpowers:skill-name`），切勿使用 `@file` 链接（该方式会强制将内容加载到上下文中）。

### 技能开发工作流（文档的测试驱动开发）

新技能遵循适配于文档的红-绿-重构流程：
1. **红**：在*没有*该技能的情况下运行压力场景，记录智能体的错误行为
2. **绿**：编写能够解决这些具体问题的最小化技能；反复测试直至合规
3. **重构**：识别智能体用于规避规则的新借口；逐一明确堵住漏洞

完整的编写指南请参阅 `writing-skills` 技能（见 `skills/writing-skills/SKILL.md`）。

### 默认工作流（注入每个智能体）

`brainstorming` → `writing-plans` → `subagent-driven-development` 或 `executing-plans`

`using-superpowers` 技能强制要求在*任何操作之前*调用相关技能。这是强制要求，而非可选项。

### 测试结构

`tests/claude-code/` 包含以无头模式（`claude -p`）运行 Claude Code 的 bash 测试，通过解析 `.jsonl` 会话记录来验证行为。`test-helpers.sh` 提供共享的断言工具（`assert_contains`、`assert_order`、`run_claude` 等）。

### 跨平台钩子兼容性

`hooks/run-hook.cmd` 是一个多语言批处理/bash 脚本，用于处理 Windows（Git for Windows bash 发现）和 Unix（`exec bash`）环境。钩子脚本使用无扩展名的文件名，以避免 Claude Code 在 Windows 上的 `.sh` 自动检测行为（该行为会自动在命令前添加 `bash`）。
