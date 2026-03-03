# 为 OpenCode 安装 Superpowers

## 前提条件

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Git

## 安装步骤

### 1. 克隆 Superpowers

```bash
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 注册插件

创建符号链接，使 OpenCode 能够发现插件：

```bash
mkdir -p ~/.config/opencode/plugins
rm -f ~/.config/opencode/plugins/superpowers.js
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js ~/.config/opencode/plugins/superpowers.js
```

### 3. 创建技能符号链接

创建符号链接，使 OpenCode 的原生技能工具能够发现 superpowers 技能：

```bash
mkdir -p ~/.config/opencode/skills
rm -rf ~/.config/opencode/skills/superpowers
ln -s ~/.config/opencode/superpowers/skills ~/.config/opencode/skills/superpowers
```

### 4. 重启 OpenCode

重启 OpenCode。插件将自动注入 superpowers 上下文。

通过询问以下内容来验证："do you have superpowers?"

## 使用方法

### 查找技能

使用 OpenCode 的原生 `skill` 工具列出可用技能：

```
use skill tool to list skills
```

### 加载技能

使用 OpenCode 的原生 `skill` 工具加载特定技能：

```
use skill tool to load superpowers/brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目技能

在项目的 `.opencode/skills/` 目录中创建特定于项目的技能。

**技能优先级：** 项目技能 > 个人技能 > Superpowers 技能

## 更新

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 故障排查

### 插件未加载

1. 检查插件符号链接：`ls -l ~/.config/opencode/plugins/superpowers.js`
2. 检查源文件是否存在：`ls ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`
3. 查看 OpenCode 日志中的错误信息

### 找不到技能

1. 检查技能符号链接：`ls -l ~/.config/opencode/skills/superpowers`
2. 验证其指向：`~/.config/opencode/superpowers/skills`
3. 使用 `skill` 工具列出已发现的技能

### 工具映射

当技能引用 Claude Code 工具时：
- `TodoWrite` → `update_plan`
- 带子代理的 `Task` → `@mention` 语法
- `Skill` 工具 → OpenCode 的原生 `skill` 工具
- 文件操作 → 你的原生工具

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 完整文档：https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
