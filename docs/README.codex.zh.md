# Codex 的超能力

通过原生技能发现，配合 OpenAI Codex 使用 Superpowers 的指南。

## 快速安装

告诉 Codex：

```
Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.codex/INSTALL.md
```

## 手动安装

### 前提条件

- OpenAI Codex CLI
- Git

### 步骤

1. 克隆仓库：
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. 创建技能符号链接：
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

3. 重启 Codex。

### Windows

使用联接代替符号链接（无需开发者模式）：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
```

## 工作原理

Codex 具有原生技能发现能力——它在启动时扫描 `~/.agents/skills/`，解析 SKILL.md 前置元数据，并按需加载技能。Superpowers 技能通过单个符号链接变得可见：

```
~/.agents/skills/superpowers/ → ~/.codex/superpowers/skills/
```

`using-superpowers` 技能会被自动发现，并强制执行技能使用规范——无需额外配置。

## 使用方法

技能会被自动发现。Codex 在以下情况下激活技能：
- 您按名称提及某个技能（例如："use brainstorming"）
- 任务与技能的描述匹配
- `using-superpowers` 技能指示 Codex 使用某个技能

### 个人技能

在 `~/.agents/skills/` 中创建您自己的技能：

```bash
mkdir -p ~/.agents/skills/my-skill
```

创建 `~/.agents/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

`description` 字段决定 Codex 何时自动激活技能——请将其写成清晰的触发条件。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能通过符号链接即时更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

**Windows（PowerShell）：**
```powershell
Remove-Item "$env:USERPROFILE\.agents\skills\superpowers"
```

可选择删除克隆目录：`rm -rf ~/.codex/superpowers`（Windows：`Remove-Item -Recurse -Force "$env:USERPROFILE\.codex\superpowers"`）。

## 故障排除

### 技能未显示

1. 验证符号链接：`ls -la ~/.agents/skills/superpowers`
2. 检查技能是否存在：`ls ~/.codex/superpowers/skills`
3. 重启 Codex——技能在启动时发现

### Windows 联接问题

联接通常无需特殊权限即可工作。如果创建失败，请尝试以管理员身份运行 PowerShell。

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
