# OpenCode 支持实施计划

> **致 Claude：** 必需子技能：使用 superpowers:executing-plans 逐任务实施本计划。

**目标：** 使用原生 JavaScript 插件为 OpenCode.ai 添加完整的 superpowers 支持，同时与现有 Codex 实现共享核心功能。

**架构：** 将通用技能发现/解析逻辑提取到 `lib/skills-core.js` 中，重构 Codex 以使用它，然后使用 OpenCode 原生插件 API（包含自定义工具和会话钩子）构建 OpenCode 插件。

**技术栈：** Node.js、JavaScript、OpenCode Plugin API、Git worktrees

---

## 第一阶段：创建共享核心模块

### 任务 1：提取前置元数据解析

**文件：**
- 创建：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex`（第 40-74 行）

**步骤 1：创建包含 extractFrontmatter 函数的 lib/skills-core.js**

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

/**
 * 从技能文件中提取 YAML 前置元数据。
 * 当前格式：
 * ---
 * name: skill-name
 * description: Use when [condition] - [what it does]
 * ---
 *
 * @param {string} filePath - SKILL.md 文件路径
 * @returns {{name: string, description: string}}
 */
function extractFrontmatter(filePath) {
    try {
        const content = fs.readFileSync(filePath, 'utf8');
        const lines = content.split('\n');

        let inFrontmatter = false;
        let name = '';
        let description = '';

        for (const line of lines) {
            if (line.trim() === '---') {
                if (inFrontmatter) break;
                inFrontmatter = true;
                continue;
            }

            if (inFrontmatter) {
                const match = line.match(/^(\w+):\s*(.*)$/);
                if (match) {
                    const [, key, value] = match;
                    switch (key) {
                        case 'name':
                            name = value.trim();
                            break;
                        case 'description':
                            description = value.trim();
                            break;
                    }
                }
            }
        }

        return { name, description };
    } catch (error) {
        return { name: '', description: '' };
    }
}

module.exports = {
    extractFrontmatter
};
```

**步骤 2：验证文件已创建**

运行：`ls -l lib/skills-core.js`
预期：文件存在

**步骤 3：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: create shared skills core module with frontmatter parser"
```

---

### 任务 2：提取技能发现逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex`（第 97-136 行）

**步骤 1：将 findSkillsInDir 函数添加到 skills-core.js**

在 `module.exports` 之前添加：

```javascript
/**
 * 递归查找目录中的所有 SKILL.md 文件。
 *
 * @param {string} dir - 要搜索的目录
 * @param {string} sourceType - 用于命名空间的 'personal' 或 'superpowers'
 * @param {number} maxDepth - 最大递归深度（默认：3）
 * @returns {Array<{path: string, name: string, description: string, sourceType: string}>}
 */
function findSkillsInDir(dir, sourceType, maxDepth = 3) {
    const skills = [];

    if (!fs.existsSync(dir)) return skills;

    function recurse(currentDir, depth) {
        if (depth > maxDepth) return;

        const entries = fs.readdirSync(currentDir, { withFileTypes: true });

        for (const entry of entries) {
            const fullPath = path.join(currentDir, entry.name);

            if (entry.isDirectory()) {
                // 检查此目录中是否有 SKILL.md
                const skillFile = path.join(fullPath, 'SKILL.md');
                if (fs.existsSync(skillFile)) {
                    const { name, description } = extractFrontmatter(skillFile);
                    skills.push({
                        path: fullPath,
                        skillFile: skillFile,
                        name: name || entry.name,
                        description: description || '',
                        sourceType: sourceType
                    });
                }

                // 递归进入子目录
                recurse(fullPath, depth + 1);
            }
        }
    }

    recurse(dir, 0);
    return skills;
}
```

**步骤 2：更新 module.exports**

将导出行替换为：

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出（成功）

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill discovery function to core module"
```

---

### 任务 3：提取技能解析逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex`（第 212-280 行）

**步骤 1：添加 resolveSkillPath 函数**

在 `module.exports` 之前添加：

```javascript
/**
 * 将技能名称解析为其文件路径，处理覆盖关系
 * （个人技能覆盖 superpowers 技能）。
 *
 * @param {string} skillName - 如 "superpowers:brainstorming" 或 "my-skill" 的名称
 * @param {string} superpowersDir - superpowers 技能目录路径
 * @param {string} personalDir - 个人技能目录路径
 * @returns {{skillFile: string, sourceType: string, skillPath: string} | null}
 */
function resolveSkillPath(skillName, superpowersDir, personalDir) {
    // 如果存在则去掉 superpowers: 前缀
    const forceSuperpowers = skillName.startsWith('superpowers:');
    const actualSkillName = forceSuperpowers ? skillName.replace(/^superpowers:/, '') : skillName;

    // 首先尝试个人技能（除非明确指定 superpowers:）
    if (!forceSuperpowers && personalDir) {
        const personalPath = path.join(personalDir, actualSkillName);
        const personalSkillFile = path.join(personalPath, 'SKILL.md');
        if (fs.existsSync(personalSkillFile)) {
            return {
                skillFile: personalSkillFile,
                sourceType: 'personal',
                skillPath: actualSkillName
            };
        }
    }

    // 尝试 superpowers 技能
    if (superpowersDir) {
        const superpowersPath = path.join(superpowersDir, actualSkillName);
        const superpowersSkillFile = path.join(superpowersPath, 'SKILL.md');
        if (fs.existsSync(superpowersSkillFile)) {
            return {
                skillFile: superpowersSkillFile,
                sourceType: 'superpowers',
                skillPath: actualSkillName
            };
        }
    }

    return null;
}
```

**步骤 2：更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill path resolution with shadowing support"
```

---

### 任务 4：提取更新检查逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex`（第 16-38 行）

**步骤 1：添加 checkForUpdates 函数**

在顶部的 require 之后添加：

```javascript
const { execSync } = require('child_process');
```

在 `module.exports` 之前添加：

```javascript
/**
 * 检查 git 仓库是否有可用的更新。
 *
 * @param {string} repoDir - git 仓库路径
 * @returns {boolean} - 如果有可用更新则为 true
 */
function checkForUpdates(repoDir) {
    try {
        // 使用 3 秒超时进行快速检查，避免网络中断时延迟
        const output = execSync('git fetch origin && git status --porcelain=v1 --branch', {
            cwd: repoDir,
            timeout: 3000,
            encoding: 'utf8',
            stdio: 'pipe'
        });

        // 解析 git status 输出以查看是否落后于远端
        const statusLines = output.split('\n');
        for (const line of statusLines) {
            if (line.startsWith('## ') && line.includes('[behind ')) {
                return true; // 我们落后于远端
            }
        }
        return false; // 已是最新
    } catch (error) {
        // 网络中断、git 错误、超时等 - 不阻塞引导
        return false;
    }
}
```

**步骤 2：更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath,
    checkForUpdates
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: add git update checking to core module"
```

---

## 第二阶段：重构 Codex 以使用共享核心

### 任务 5：更新 Codex 以导入共享核心

**文件：**
- 修改：`.codex/superpowers-codex`（在顶部添加导入）

**步骤 1：添加导入语句**

在文件顶部现有 require 之后（约第 6 行），添加：

```javascript
const skillsCore = require('../lib/skills-core');
```

**步骤 2：验证语法**

运行：`node -c .codex/superpowers-codex`
预期：无输出

**步骤 3：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: import shared skills core in codex"
```

---

### 任务 6：用核心版本替换 extractFrontmatter

**文件：**
- 修改：`.codex/superpowers-codex`（第 40-74 行）

**步骤 1：删除本地 extractFrontmatter 函数**

删除第 40-74 行（整个 extractFrontmatter 函数定义）。

**步骤 2：更新所有 extractFrontmatter 调用**

查找并将所有调用从 `extractFrontmatter(` 替换为 `skillsCore.extractFrontmatter(`

受影响的行大约在：90、310

**步骤 3：验证脚本仍然工作**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示技能列表

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared extractFrontmatter in codex"
```

---

### 任务 7：用核心版本替换 findSkillsInDir

**文件：**
- 修改：`.codex/superpowers-codex`（约第 97-136 行）

**步骤 1：删除本地 findSkillsInDir 函数**

删除整个 `findSkillsInDir` 函数定义（约第 97-136 行）。

**步骤 2：更新所有 findSkillsInDir 调用**

将调用从 `findSkillsInDir(` 替换为 `skillsCore.findSkillsInDir(`

**步骤 3：验证脚本仍然工作**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示技能列表

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared findSkillsInDir in codex"
```

---

### 任务 8：用核心版本替换 checkForUpdates

**文件：**
- 修改：`.codex/superpowers-codex`（约第 16-38 行）

**步骤 1：删除本地 checkForUpdates 函数**

删除整个 `checkForUpdates` 函数定义。

**步骤 2：更新所有 checkForUpdates 调用**

将调用从 `checkForUpdates(` 替换为 `skillsCore.checkForUpdates(`

**步骤 3：验证脚本仍然工作**

运行：`.codex/superpowers-codex bootstrap | head -50`
预期：显示引导内容

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared checkForUpdates in codex"
```

---

## 第三阶段：构建 OpenCode 插件

### 任务 9：创建 OpenCode 插件目录结构

**文件：**
- 创建：`.opencode/plugin/superpowers.js`

**步骤 1：创建目录**

运行：`mkdir -p .opencode/plugin`

**步骤 2：创建基本插件文件**

```javascript
#!/usr/bin/env node

/**
 * OpenCode.ai 的 Superpowers 插件
 *
 * 提供用于加载和发现技能的自定义工具，
 * 并在会话启动时自动引导。
 */

const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const os = require('os');

const homeDir = os.homedir();
const superpowersSkillsDir = path.join(homeDir, '.config/opencode/superpowers/skills');
const personalSkillsDir = path.join(homeDir, '.config/opencode/skills');

/**
 * OpenCode 插件入口点
 */
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // 自定义工具和钩子将在此处添加
  };
};
```

**步骤 3：验证文件已创建**

运行：`ls -l .opencode/plugin/superpowers.js`
预期：文件存在

**步骤 4：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: create opencode plugin scaffold"
```

---

### 任务 10：实现 use_skill 工具

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：添加 use_skill 工具实现**

将插件 return 语句替换为：

```javascript
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  // 导入 zod 用于模式验证
  const { z } = await import('zod');

  return {
    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill to guide your work. Skills contain proven workflows, mandatory processes, and expert techniques.',
        schema: z.object({
          skill_name: z.string().describe('Name of the skill to load (e.g., "superpowers:brainstorming" or "my-custom-skill")')
        }),
        execute: async ({ skill_name }) => {
          // 解析技能路径（处理覆盖：个人 > superpowers）
          const resolved = skillsCore.resolveSkillPath(
            skill_name,
            superpowersSkillsDir,
            personalSkillsDir
          );

          if (!resolved) {
            return `Error: Skill "${skill_name}" not found.\n\nRun find_skills to see available skills.`;
          }

          // 读取技能内容
          const fullContent = fs.readFileSync(resolved.skillFile, 'utf8');
          const { name, description } = skillsCore.extractFrontmatter(resolved.skillFile);

          // 提取前置元数据之后的内容
          const lines = fullContent.split('\n');
          let inFrontmatter = false;
          let frontmatterEnded = false;
          const contentLines = [];

          for (const line of lines) {
            if (line.trim() === '---') {
              if (inFrontmatter) {
                frontmatterEnded = true;
                continue;
              }
              inFrontmatter = true;
              continue;
            }

            if (frontmatterEnded || !inFrontmatter) {
              contentLines.push(line);
            }
          }

          const content = contentLines.join('\n').trim();
          const skillDirectory = path.dirname(resolved.skillFile);

          // 格式化输出，类似于 Claude Code 的 Skill 工具
          return `# ${name || skill_name}
# ${description || ''}
# Supporting tools and docs are in ${skillDirectory}
# ============================================

${content}`;
        }
      }
    ]
  };
};
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement use_skill tool for opencode"
```

---

### 任务 11：实现 find_skills 工具

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：将 find_skills 工具添加到 tools 数组**

在 use_skill 工具定义之后、关闭 tools 数组之前添加：

```javascript
      {
        name: 'find_skills',
        description: 'List all available skills in the superpowers and personal skill libraries.',
        schema: z.object({}),
        execute: async () => {
          // 在两个目录中查找技能
          const superpowersSkills = skillsCore.findSkillsInDir(
            superpowersSkillsDir,
            'superpowers',
            3
          );
          const personalSkills = skillsCore.findSkillsInDir(
            personalSkillsDir,
            'personal',
            3
          );

          // 合并并格式化技能列表
          const allSkills = [...personalSkills, ...superpowersSkills];

          if (allSkills.length === 0) {
            return 'No skills found. Install superpowers skills to ~/.config/opencode/superpowers/skills/';
          }

          let output = 'Available skills:\n\n';

          for (const skill of allSkills) {
            const namespace = skill.sourceType === 'personal' ? '' : 'superpowers:';
            const skillName = skill.name || path.basename(skill.path);

            output += `${namespace}${skillName}\n`;
            if (skill.description) {
              output += `  ${skill.description}\n`;
            }
            output += `  Directory: ${skill.path}\n\n`;
          }

          return output;
        }
      }
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement find_skills tool for opencode"
```

---

### 任务 12：实现会话启动钩子

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：在 tools 数组之后添加 session.started 钩子**

```javascript
    'session.started': async () => {
      // 读取 using-superpowers 技能内容
      const usingSuperpowersPath = skillsCore.resolveSkillPath(
        'using-superpowers',
        superpowersSkillsDir,
        personalSkillsDir
      );

      let usingSuperpowersContent = '';
      if (usingSuperpowersPath) {
        const fullContent = fs.readFileSync(usingSuperpowersPath.skillFile, 'utf8');
        // 去掉前置元数据
        const lines = fullContent.split('\n');
        let inFrontmatter = false;
        let frontmatterEnded = false;
        const contentLines = [];

        for (const line of lines) {
          if (line.trim() === '---') {
            if (inFrontmatter) {
              frontmatterEnded = true;
              continue;
            }
            inFrontmatter = true;
            continue;
          }

          if (frontmatterEnded || !inFrontmatter) {
            contentLines.push(line);
          }
        }

        usingSuperpowersContent = contentLines.join('\n').trim();
      }

      // 工具映射指令
      const toolMapping = `
**Tool Mapping for OpenCode:**
When skills reference tools you don't have, substitute OpenCode equivalents:
- \`TodoWrite\` → \`update_plan\` (your planning/task tracking tool)
- \`Task\` tool with subagents → Use OpenCode's subagent system (@mention syntax or automatic dispatch)
- \`Skill\` tool → \`use_skill\` custom tool (already available)
- \`Read\`, \`Write\`, \`Edit\`, \`Bash\` → Use your native tools

**Skill directories contain supporting files:**
- Scripts you can run with bash tool
- Additional documentation you can read
- Utilities and helpers specific to that skill

**Skills naming:**
- Superpowers skills: \`superpowers:skill-name\` (from ~/.config/opencode/superpowers/skills/)
- Personal skills: \`skill-name\` (from ~/.config/opencode/skills/)
- Personal skills override superpowers skills when names match
`;

      // 检查更新（非阻塞）
      const hasUpdates = skillsCore.checkForUpdates(
        path.join(homeDir, '.config/opencode/superpowers')
      );

      const updateNotice = hasUpdates ?
        '\n\n⚠️ **Updates available!** Run `cd ~/.config/opencode/superpowers && git pull` to update superpowers.' :
        '';

      // 返回要注入会话的上下文
      return {
        context: `<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'use_skill' tool:**

${usingSuperpowersContent}

${toolMapping}${updateNotice}
</EXTREMELY_IMPORTANT>`
      };
    }
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement session.started hook for opencode"
```

---

## 第四阶段：文档

### 任务 13：创建 OpenCode 安装指南

**文件：**
- 创建：`.opencode/INSTALL.md`

**步骤 1：创建安装指南**

```markdown
# 为 OpenCode 安装 Superpowers

## 前提条件

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Node.js
- 已安装 Git

## 安装步骤

### 1. 安装 Superpowers 技能

```bash
# 将 superpowers 技能克隆到 OpenCode 配置目录
mkdir -p ~/.config/opencode/superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 安装插件

插件包含在您刚刚克隆的 superpowers 仓库中。

OpenCode 会从以下位置自动发现它：
- `~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`

或者您可以将其链接到项目本地的插件目录：

```bash
# 在您的 OpenCode 项目中
mkdir -p .opencode/plugin
ln -s ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js .opencode/plugin/superpowers.js
```

### 3. 重启 OpenCode

重启 OpenCode 以加载插件。在下一个会话中，您应该会看到：

```
You have superpowers.
```

## 使用方法

### 查找技能

使用 `find_skills` 工具列出所有可用技能：

```
use find_skills tool
```

### 加载技能

使用 `use_skill` 工具加载特定技能：

```
use use_skill tool with skill_name: "superpowers:brainstorming"
```

### 个人技能

在 `~/.config/opencode/skills/` 中创建您自己的技能：

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

个人技能会覆盖同名的 superpowers 技能。

## 更新

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 故障排除

### 插件未加载

1. 检查插件文件是否存在：`ls ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`
2. 检查 OpenCode 日志中的错误
3. 验证 Node.js 已安装：`node --version`

### 找不到技能

1. 验证技能目录是否存在：`ls ~/.config/opencode/superpowers/skills`
2. 使用 `find_skills` 工具查看已发现的内容
3. 检查文件结构：每个技能应有一个 `SKILL.md` 文件

### 工具映射问题

当技能引用您没有的 Claude Code 工具时：
- `TodoWrite` → 使用 `update_plan`
- 带子代理的 `Task` → 使用 `@mention` 语法调用 OpenCode 子代理
- `Skill` → 使用 `use_skill` 工具
- 文件操作 → 使用您的原生工具

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 文档：https://github.com/obra/superpowers
```

**步骤 2：验证文件已创建**

运行：`ls -l .opencode/INSTALL.md`
预期：文件存在

**步骤 3：提交**

```bash
git add .opencode/INSTALL.md
git commit -m "docs: add opencode installation guide"
```

---

### 任务 14：更新主 README

**文件：**
- 修改：`README.md`

**步骤 1：添加 OpenCode 章节**

找到关于支持平台的章节（在文件中搜索"Codex"），并在其后添加：

```markdown
### OpenCode

Superpowers 通过原生 JavaScript 插件支持 [OpenCode.ai](https://opencode.ai)。

**安装：** 参见 [.opencode/INSTALL.md](.opencode/INSTALL.md)

**特性：**
- 自定义工具：`use_skill` 和 `find_skills`
- 自动会话引导
- 支持覆盖的个人技能
- 支持文件和脚本访问
```

**步骤 2：验证格式**

运行：`grep -A 10 "### OpenCode" README.md`
预期：显示您添加的章节

**步骤 3：提交**

```bash
git add README.md
git commit -m "docs: add opencode support to readme"
```

---

### 任务 15：更新发布说明

**文件：**
- 修改：`RELEASE-NOTES.md`

**步骤 1：添加 OpenCode 支持条目**

在文件顶部（标题之后），添加：

```markdown
## [未发布]

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 带工具映射指令的自动会话引导
  - 共享核心模块（`lib/skills-core.js`）用于代码复用
  - `.opencode/INSTALL.md` 中的安装指南

### 变更

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` 模块
  - 消除 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一事实来源

---

```

**步骤 2：验证格式**

运行：`head -30 RELEASE-NOTES.md`
预期：显示您的新章节

**步骤 3：提交**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: add opencode support to release notes"
```

---

## 第五阶段：最终验证

### 任务 16：测试 Codex 仍然工作

**文件：**
- 测试：`.codex/superpowers-codex`

**步骤 1：测试 find-skills 命令**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示包含名称和描述的技能列表

**步骤 2：测试 use-skill 命令**

运行：`.codex/superpowers-codex use-skill superpowers:brainstorming | head -20`
预期：显示 brainstorming 技能内容

**步骤 3：测试 bootstrap 命令**

运行：`.codex/superpowers-codex bootstrap | head -30`
预期：显示包含说明的引导内容

**步骤 4：如果所有测试通过，记录成功**

无需提交——这仅是验证。

---

### 任务 17：验证文件结构

**文件：**
- 检查：所有新文件是否存在

**步骤 1：验证所有文件已创建**

运行：
```bash
ls -l lib/skills-core.js
ls -l .opencode/plugin/superpowers.js
ls -l .opencode/INSTALL.md
```

预期：所有文件存在

**步骤 2：验证目录结构**

运行：`tree -L 2 .opencode/`（如果 tree 不可用则用 `find .opencode -type f`）
预期：
```
.opencode/
├── INSTALL.md
└── plugin/
    └── superpowers.js
```

**步骤 3：如果结构正确，继续**

无需提交——这仅是验证。

---

### 任务 18：最终提交和总结

**文件：**
- 检查：`git status`

**步骤 1：检查 git 状态**

运行：`git status`
预期：工作树干净，所有变更已提交

**步骤 2：审查提交日志**

运行：`git log --oneline -20`
预期：显示此次实施的所有提交

**步骤 3：创建总结文档**

创建完成总结，显示：
- 已创建的提交总数
- 已创建的文件：`lib/skills-core.js`、`.opencode/plugin/superpowers.js`、`.opencode/INSTALL.md`
- 已修改的文件：`.codex/superpowers-codex`、`README.md`、`RELEASE-NOTES.md`
- 已执行的测试：已验证 Codex 命令
- 准备进行：使用实际 OpenCode 安装进行测试

**步骤 4：汇报完成**

向用户展示总结，并提供：
1. 推送到远端
2. 创建拉取请求
3. 使用真实 OpenCode 安装进行测试（需要已安装 OpenCode）

---

## 测试指南（手动——需要 OpenCode）

以下步骤需要安装 OpenCode，不属于自动化实施的一部分：

1. **安装技能**：遵循 `.opencode/INSTALL.md`
2. **启动 OpenCode 会话**：验证引导内容是否出现
3. **测试 find_skills**：应列出所有可用技能
4. **测试 use_skill**：加载技能并验证内容是否出现
5. **测试支持文件**：验证技能目录路径是否可访问
6. **测试个人技能**：创建个人技能并验证其覆盖核心技能
7. **测试工具映射**：验证 TodoWrite → update_plan 的映射是否有效

## 成功标准

- [ ] 创建了包含所有核心函数的 `lib/skills-core.js`
- [ ] `.codex/superpowers-codex` 重构为使用共享核心
- [ ] Codex 命令仍然工作（find-skills、use-skill、bootstrap）
- [ ] 创建了包含工具和钩子的 `.opencode/plugin/superpowers.js`
- [ ] 安装指南已创建
- [ ] README 和 RELEASE-NOTES 已更新
- [ ] 所有变更已提交
- [ ] 工作树干净
