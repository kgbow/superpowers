# Superpowers 发布说明

## v4.3.1（2026-02-21）

### 新增

**Cursor 支持**

Superpowers 现已支持 Cursor 的插件系统。包含 `.cursor-plugin/plugin.json` 清单文件以及 README 中针对 Cursor 的安装说明。SessionStart 钩子的输出现在在现有 `hookSpecificOutput.additionalContext` 字段的基础上，新增了 `additional_context` 字段，以兼容 Cursor 的钩子机制。

### 修复

**Windows：恢复多语言包装器以确保可靠的钩子执行（#518、#504、#491、#487、#466、#440）**

Claude Code 在 Windows 上的 `.sh` 自动检测会在钩子命令前添加 `bash`，导致执行失败。修复方案：

- 将 `session-start.sh` 重命名为 `session-start`（无扩展名），避免自动检测干扰
- 恢复 `run-hook.cmd` 多语言包装器，支持多位置 bash 发现（标准 Git for Windows 路径，然后回退到 PATH）
- 如果找不到 bash，则静默退出而非报错
- 在 Unix 上，包装器通过 `exec bash` 直接运行脚本
- 使用符合 POSIX 标准的 `dirname "$0"` 路径解析（适用于 dash/sh，而不仅限于 bash）

此修复解决了 Windows 上路径含空格、缺少 WSL、`set -euo pipefail` 在 MSYS 上的脆弱性以及反斜杠被转义等问题导致的 SessionStart 失败。

## v4.3.0（2026-02-12）

此次修复应能显著提升 superpowers 技能的合规性，并减少 Claude 意外进入原生计划模式的可能性。

### 变更

**brainstorming 技能现在强制执行工作流，而非仅仅描述它**

模型曾跳过设计阶段，直接进入 frontend-design 等实现技能，或将整个头脑风暴过程压缩成一个文本块。该技能现在使用硬性关卡、强制清单和 graphviz 流程图来确保合规：

- `<HARD-GATE>`：在设计呈现并获得用户批准之前，禁止使用任何实现技能、代码或脚手架
- 明确的清单（6 项），必须作为任务创建并按顺序完成
- 带有 graphviz 流程图，以 `writing-plans` 作为唯一有效的终止状态
- 针对"这太简单了，不需要设计"这一反模式的警告——这正是模型用来跳过流程的典型借口
- 设计章节的大小根据章节复杂度而定，而非项目复杂度

**using-superpowers 工作流图拦截 EnterPlanMode**

在技能流程图中添加了 `EnterPlanMode` 拦截点。当模型即将进入 Claude 原生计划模式时，它会检查头脑风暴是否已经发生，并转而通过 brainstorming 技能来处理。计划模式永远不会被进入。

### 修复

**SessionStart 钩子现在同步运行**

将 hooks.json 中的 `async: true` 改为 `async: false`。异步时，钩子可能在模型第一轮响应前未能完成，导致 using-superpowers 的指令未能在第一条消息的上下文中生效。

## v4.2.0（2026-02-05）

### 破坏性变更

**Codex：以原生技能发现取代引导 CLI**

`superpowers-codex` 引导 CLI、Windows `.cmd` 包装器及相关引导内容文件已被移除。Codex 现在通过 `~/.agents/skills/superpowers/` 符号链接使用原生技能发现，因此旧的 `use_skill`/`find_skills` CLI 工具不再需要。

安装现在只需克隆加符号链接（详见 INSTALL.md）。不再依赖 Node.js。旧的 `~/.codex/skills/` 路径已废弃。

### 修复

**Windows：修复 Claude Code 2.1.x 钩子执行问题（#331）**

Claude Code 2.1.x 改变了 Windows 上钩子的执行方式：它现在会自动检测命令中的 `.sh` 文件并在前面添加 `bash`。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 `.cmd` 文件作为 bash 脚本执行。

修复方案：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 会自动处理 bash 调用。同时添加了 .gitattributes 以强制 shell 脚本使用 LF 换行符（修复 Windows 检出时的 CRLF 问题）。

**Windows：SessionStart 钩子异步运行以防止终端冻结（#404、#413、#414、#419）**

同步运行的 SessionStart 钩子会阻止 TUI 在 Windows 上进入原始模式，导致所有键盘输入冻结。异步运行钩子可防止冻结，同时仍能注入 superpowers 上下文。

**Windows：修复 `escape_for_json` 的 O(n²) 性能问题**

使用 `${input:$i:1}` 的逐字符循环在 bash 中由于子字符串复制开销而呈 O(n²) 复杂度。在 Windows Git Bash 上这需要花费 60 秒以上。已替换为 bash 参数替换（`${s//old/new}`），以单次 C 层级处理每个模式——在 macOS 上快 7 倍，在 Windows 上提升幅度更大。

**Codex：修复 Windows/PowerShell 调用问题（#285、#243）**

- Windows 不识别 shebang，因此直接调用无扩展名的 `superpowers-codex` 脚本会触发"打开方式"对话框。所有调用现在都添加了 `node` 前缀。
- 修复了 Windows 上的 `~/` 路径展开问题——PowerShell 在将 `~` 作为参数传递给 `node` 时不会展开它。已改为 `$HOME`，在 bash 和 PowerShell 中均能正确展开。

**Codex：修复安装程序中的路径解析问题**

使用 `fileURLToPath()` 代替手动 URL 路径名解析，以在所有平台上正确处理含空格和特殊字符的路径。

**Codex：修复 writing-skills 中过时的技能路径**

将已废弃的 `~/.codex/skills/` 引用更新为 `~/.agents/skills/`，以支持原生发现。

### 改进

**工作树隔离现在是实现前的必要条件**

将 `using-git-worktrees` 添加为 `subagent-driven-development` 和 `executing-plans` 的必要技能。实现工作流现在明确要求在开始工作前设置隔离的工作树，防止意外直接在主分支上工作。

**主分支保护调整为需要明确同意**

不再完全禁止在主分支上工作，技能现在允许在用户明确同意的情况下进行。更加灵活，同时仍确保用户了解其影响。

**简化安装验证**

从验证步骤中移除了 `/help` 命令检查和特定斜杠命令列表。技能主要通过描述你想做的事来调用，而非运行特定命令。

**Codex：在引导程序中明确子智能体工具映射**

改进了 Codex 工具与 Claude Code 等效工具在子智能体工作流中的对应关系文档。

### 测试

- 为 subagent-driven-development 添加了工作树要求测试
- 添加了主分支红色警告测试
- 修复了技能识别测试断言中的大小写敏感问题

---

## v4.1.1（2026-01-23）

### 修复

**OpenCode：按照官方文档统一使用 `plugins/` 目录（#343）**

OpenCode 的官方文档使用 `~/.config/opencode/plugins/`（复数形式）。我们的文档之前使用的是 `plugin/`（单数形式）。虽然 OpenCode 两种形式都接受，但我们已统一使用官方约定以避免混淆。

变更：
- 将仓库结构中的 `.opencode/plugin/` 重命名为 `.opencode/plugins/`
- 更新了所有平台的安装文档（INSTALL.md、README.opencode.md）
- 更新了测试脚本以匹配

**OpenCode：修复符号链接说明（#339、#342）**

- 在 `ln -s` 之前添加了明确的 `rm`（修复重新安装时的"文件已存在"错误）
- 添加了 INSTALL.md 中缺失的技能符号链接步骤
- 从已废弃的 `use_skill`/`find_skills` 更新为原生 `skill` 工具引用

---

## v4.1.0（2026-01-23）

### 破坏性变更

**OpenCode：切换到原生技能系统**

Superpowers for OpenCode 现在使用 OpenCode 的原生 `skill` 工具，而非自定义的 `use_skill`/`find_skills` 工具。这是一种更简洁的集成方式，与 OpenCode 的内置技能发现机制兼容。

**需要迁移：** 技能必须符号链接到 `~/.config/opencode/skills/superpowers/`（详见更新后的安装文档）。

### 修复

**OpenCode：修复会话开始时的智能体重置问题（#226）**

之前使用 `session.prompt({ noReply: true })` 的引导注入方法会导致 OpenCode 在第一条消息时将所选智能体重置为"build"。现在改用 `experimental.chat.system.transform` 钩子，该钩子直接修改系统提示而无副作用。

**OpenCode：修复 Windows 安装问题（#232）**

- 移除了对 `skills-core.js` 的依赖（消除了文件被复制而非符号链接时的相对导入错误）
- 为 cmd.exe、PowerShell 和 Git Bash 添加了全面的 Windows 安装文档
- 记录了每个平台的正确符号链接与 junction 用法

**Claude Code：修复 Claude Code 2.1.x 的 Windows 钩子执行问题**

Claude Code 2.1.x 改变了 Windows 上钩子的执行方式：它现在会自动检测命令中的 `.sh` 文件并在前面添加 `bash `。这破坏了多语言包装器模式，因为 `bash "run-hook.cmd" session-start.sh` 会尝试将 .cmd 文件作为 bash 脚本执行。

修复方案：hooks.json 现在直接调用 session-start.sh。Claude Code 2.1.x 会自动处理 bash 调用。同时添加了 .gitattributes 以强制 shell 脚本使用 LF 换行符（修复 Windows 检出时的 CRLF 问题）。

---

## v4.0.3（2025-12-26）

### 改进

**强化 using-superpowers 技能以处理明确的技能请求**

解决了一个失败模式：当用户明确按名称请求技能时（例如，"请使用 subagent-driven-development"），Claude 会跳过调用该技能。Claude 会认为"我知道那是什么意思"并直接开始工作，而不是加载技能。

变更：
- 将"规则"从"检查技能"更新为"调用相关或被请求的技能"——强调主动调用而非被动检查
- 添加"在任何响应或操作之前"——原来的措辞只提到"响应"，但 Claude 有时会在不先响应的情况下直接采取行动
- 添加了调用错误技能也没关系的说明——减少犹豫
- 添加新的红色警告："我知道那是什么意思"→了解概念 ≠ 使用技能

**添加了明确的技能请求测试**

在 `tests/explicit-skill-requests/` 中新增测试套件，验证用户按名称请求技能时 Claude 是否正确调用。包含单轮和多轮测试场景。

## v4.0.2（2025-12-23）

### 修复

**斜杠命令现在仅限用户使用**

为所有三个斜杠命令（`/brainstorm`、`/execute-plan`、`/write-plan`）添加了 `disable-model-invocation: true`。Claude 不再能通过 Skill 工具调用这些命令——它们被限制为仅供用户手动调用。

底层技能（`superpowers:brainstorming`、`superpowers:executing-plans`、`superpowers:writing-plans`）仍然可供 Claude 自主调用。此变更防止了 Claude 调用最终只是重定向到技能的命令时产生的混乱。

## v4.0.1（2025-12-23）

### 修复

**明确了在 Claude Code 中访问技能的方式**

修复了一个令人困惑的模式：Claude 会通过 Skill 工具调用技能，然后再尝试单独读取技能文件。`using-superpowers` 技能现在明确说明 Skill 工具直接加载技能内容——无需读取文件。

- 在 `using-superpowers` 中添加了"如何访问技能"章节
- 将指令中的"读取技能"改为"调用技能"
- 更新了斜杠命令以使用完全限定的技能名称（例如 `superpowers:brainstorming`）

**在 receiving-code-review 中添加了 GitHub 线程回复指引**（感谢 @ralphbean）

添加了关于在原始线程中回复内联审查评论，而非作为顶级 PR 评论的说明。

**在 writing-skills 中添加了自动化优于文档的指引**（感谢 @EthanJStark）

添加了机械性约束应通过自动化实现而非文档记录的指引——技能应保留用于需要判断的场合。

## v4.0.0（2025-12-17）

### 新功能

**subagent-driven-development 中的两阶段代码审查**

子智能体工作流现在在每个任务完成后使用两个独立的审查阶段：

1. **规格合规性审查** - 持怀疑态度的审查者验证实现是否与规格完全匹配。捕获遗漏的需求和过度构建。不信任实现者的报告——直接阅读实际代码。

2. **代码质量审查** - 仅在规格合规性通过后运行。审查代码的整洁度、测试覆盖率和可维护性。

这捕获了一种常见的失败模式：代码写得很好，但与请求不符。审查是循环而非一次性的：如果审查者发现问题，实现者修复后审查者再次检查。

其他子智能体工作流改进：
- 控制器向工作者提供完整任务文本（而非文件引用）
- 工作者可以在工作前和工作中提问澄清问题
- 在报告完成前进行自我审查清单
- 计划在开始时读取一次，提取到 TodoWrite

`skills/subagent-driven-development/` 中的新提示模板：
- `implementer-prompt.md` - 包含自我审查清单，鼓励提问
- `spec-reviewer-prompt.md` - 针对需求的持怀疑态度的验证
- `code-quality-reviewer-prompt.md` - 标准代码审查

**调试技术与工具整合**

`systematic-debugging` 现在捆绑了支持技术和工具：
- `root-cause-tracing.md` - 通过调用栈向后追踪 bug
- `defense-in-depth.md` - 在多个层级添加验证
- `condition-based-waiting.md` - 用条件轮询替代任意超时
- `find-polluter.sh` - 二分法脚本，用于找出造成污染的测试
- `condition-based-waiting-example.ts` - 来自真实调试会话的完整实现

**测试反模式参考**

`test-driven-development` 现在包含 `testing-anti-patterns.md`，涵盖：
- 测试模拟行为而非真实行为
- 向生产类添加仅供测试使用的方法
- 不理解依赖关系就进行模拟
- 隐藏结构假设的不完整模拟

**技能测试基础设施**

用于验证技能行为的三个新测试框架：

`tests/skill-triggering/` - 验证技能能从朴素提示中触发，无需明确命名。测试 6 个技能以确保仅凭描述即可触发。

`tests/claude-code/` - 使用 `claude -p` 进行无头测试的集成测试。通过会话记录（JSONL）分析验证技能使用情况。包含用于成本追踪的 `analyze-token-usage.py`。

`tests/subagent-driven-dev/` - 包含两个完整测试项目的端到端工作流验证：
- `go-fractals/` - 带 Sierpinski/Mandelbrot 的 CLI 工具（10 个任务）
- `svelte-todo/` - 带 localStorage 和 Playwright 的 CRUD 应用（12 个任务）

### 重大变更

**DOT 流程图作为可执行规范**

用 DOT/GraphViz 流程图作为权威流程定义重写了关键技能。散文内容变为辅助说明。

**描述陷阱**（记录于 `writing-skills`）：发现当描述包含工作流摘要时，技能描述会覆盖流程图内容。Claude 会遵循简短的描述，而非阅读详细的流程图。修复方案：描述必须仅作为触发条件（"在 X 时使用"），不包含任何流程细节。

**using-superpowers 中的技能优先级**

当多个技能适用时，流程技能（brainstorming、debugging）现在明确优先于实现技能。"构建 X"会先触发 brainstorming，然后才是领域技能。

**brainstorming 触发条件强化**

描述改为命令式：「在任何创意工作之前——创建功能、构建组件、添加功能或修改行为——你必须使用此技能。」

### 破坏性变更

**技能整合** - 六个独立技能被合并：
- `root-cause-tracing`、`defense-in-depth`、`condition-based-waiting` → 捆绑到 `systematic-debugging/`
- `testing-skills-with-subagents` → 捆绑到 `writing-skills/`
- `testing-anti-patterns` → 捆绑到 `test-driven-development/`
- `sharing-skills` 已移除（已过时）

### 其他改进

- **render-graphs.js** - 从技能中提取 DOT 图并渲染为 SVG 的工具
- **using-superpowers 中的理由列表** - 可扫描格式，包含新条目："我需要更多上下文"、"让我先探索一下"、"这感觉很有成效"
- **docs/testing.md** - 使用 Claude Code 集成测试来测试技能的指南

---

## v3.6.2（2025-12-03）

### 修复

- **Linux 兼容性**：修复多语言钩子包装器（`run-hook.cmd`）以使用符合 POSIX 标准的语法
  - 将第 16 行 bash 特有的 `${BASH_SOURCE[0]:-$0}` 替换为标准的 `$0`
  - 解决了 Ubuntu/Debian 系统（`/bin/sh` 为 dash）上的"Bad substitution"错误
  - 修复 #141

---

## v3.5.1（2025-11-24）

### 变更

- **OpenCode 引导程序重构**：从 `chat.message` 钩子切换到 `session.created` 事件进行引导注入
  - 引导程序现在通过 `session.prompt()` 并设置 `noReply: true` 在会话创建时注入
  - 明确告知模型 using-superpowers 已加载，防止冗余的技能加载
  - 将引导内容生成整合到共享的 `getBootstrapContent()` 助手函数中
  - 更简洁的单一实现方式（移除了回退模式）

---

## v3.5.0（2025-11-23）

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 跨上下文压缩的技能持久化消息插入模式
  - 通过 chat.message 钩子自动注入上下文
  - 在 session.compacted 事件时自动重新注入
  - 三级技能优先级：项目 > 个人 > superpowers
  - 项目本地技能支持（`.opencode/skills/`）
  - 与 Codex 代码复用的共享核心模块（`lib/skills-core.js`）
  - 带有适当隔离的自动化测试套件（`tests/opencode/`）
  - 平台特定文档（`docs/README.opencode.md`、`docs/README.codex.md`）

### 变更

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` ES 模块
  - 消除了 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一可信来源
  - Codex 通过 Node.js 互操作成功加载 ES 模块

- **改进文档**：重写 README 以清晰说明问题/解决方案
  - 移除重复章节和冲突信息
  - 添加完整工作流描述（头脑风暴 → 计划 → 执行 → 完成）
  - 简化平台安装说明
  - 强调技能检查协议而非自动激活声明

---

## v3.4.1（2025-10-31）

### 改进

- 优化 superpowers 引导程序以消除冗余的技能执行。`using-superpowers` 技能内容现在直接在会话上下文中提供，并附有仅对其他技能使用 Skill 工具的明确指引。这减少了开销，并防止了智能体尽管在会话开始时已有内容，却仍手动执行 `using-superpowers` 的令人困惑的循环。

## v3.4.0（2025-10-30）

### 改进

- 简化了 `brainstorming` 技能，回归到最初的对话式愿景。移除了包含正式清单的重量级 6 阶段流程，改为自然对话：一次一个问题，然后以 200-300 字的章节呈现设计并进行验证。保留了文档和实现交接功能。

## v3.3.1（2025-10-28）

### 改进

- 更新了 `brainstorming` 技能，要求在提问前先进行自主侦查，鼓励以推荐驱动决策，并防止智能体将优先级排序委托回给人类。
- 按照 Strunk《风格的要素》原则对 `brainstorming` 技能进行了写作清晰度改进（删除多余词语，将否定形式转换为肯定形式，改善平行结构）。

### 错误修复

- 明确了 `writing-skills` 指引，使其指向正确的智能体特定个人技能目录（Claude Code 的 `~/.claude/skills`，Codex 的 `~/.codex/skills`）。

## v3.3.0（2025-10-28）

### 新功能

**实验性 Codex 支持**
- 添加了统一的 `superpowers-codex` 脚本，包含 bootstrap/use-skill/find-skills 命令
- 跨平台 Node.js 实现（适用于 Windows、macOS、Linux）
- 命名空间技能：superpowers 技能使用 `superpowers:skill-name`，个人技能使用 `skill-name`
- 名称匹配时个人技能覆盖 superpowers 技能
- 整洁的技能展示：显示名称/描述而不显示原始 frontmatter
- 有用的上下文：显示每个技能的支持文件目录
- Codex 的工具映射：TodoWrite→update_plan，子智能体→手动回退等
- 与最小 AGENTS.md 的引导集成，实现自动启动
- 完整的安装指南和针对 Codex 的引导说明

**与 Claude Code 集成的主要区别：**
- 单一统一脚本而非独立工具
- 针对 Codex 特定等效工具的工具替换系统
- 简化的子智能体处理（手动工作而非委托）
- 更新的术语："Superpowers 技能"而非"核心技能"

### 新增文件
- `.codex/INSTALL.md` - Codex 用户安装指南
- `.codex/superpowers-bootstrap.md` - 包含 Codex 适配的引导说明
- `.codex/superpowers-codex` - 包含所有功能的统一 Node.js 可执行文件

**注意：** Codex 支持为实验性功能。该集成提供了核心 superpowers 功能，但可能需要根据用户反馈进行改进。

## v3.2.3（2025-10-23）

### 改进

**更新 using-superpowers 技能以使用 Skill 工具而非 Read 工具**
- 将技能调用说明从 Read 工具改为 Skill 工具
- 更新描述："使用 Read 工具" → "使用 Skill 工具"
- 更新步骤 3："使用 Read 工具" → "使用 Skill 工具读取并运行"
- 更新理由列表："读取当前版本" → "运行当前版本"

Skill 工具是在 Claude Code 中调用技能的正确机制。此次更新修正了引导说明，引导智能体使用正确的工具。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 将工具引用从 Read 改为 Skill

## v3.2.2（2025-10-21）

### 改进

**强化 using-superpowers 技能以对抗智能体的自我合理化**
- 添加了包含绝对语言的 EXTREMELY-IMPORTANT 块，关于强制技能检查
  - "即使有 1% 的概率某个技能适用，你也必须读取它"
  - "你没有选择。你无法用理由绕过它。"
- 添加了强制的首次响应协议清单
  - 智能体在任何响应之前必须完成的 5 步流程
  - 明确的"不按此响应 = 失败"后果
- 添加了常见自我合理化部分，列举 8 种具体的规避模式
  - "这只是一个简单的问题" → 错误
  - "我可以快速检查文件" → 错误
  - "让我先收集信息" → 错误
  - 以及在智能体行为中观察到的另外 5 种常见模式

这些变更针对观察到的智能体行为——尽管有明确指令，它们仍会用理由绕过技能使用。强硬的语言和预防性的反驳旨在使不合规行为更难发生。

### 变更文件
- 更新：`skills/using-superpowers/SKILL.md` - 添加三层执行机制以防止技能跳过的合理化

## v3.2.1（2025-10-20）

### 新功能

**代码审查智能体现在包含在插件中**
- 在插件的 `agents/` 目录中添加了 `superpowers:code-reviewer` 智能体
- 智能体根据计划和编码标准提供系统化代码审查
- 之前需要用户拥有个人智能体配置
- 所有技能引用已更新为使用命名空间 `superpowers:code-reviewer`
- 修复 #55

### 变更文件
- 新增：`agents/code-reviewer.md` - 带有审查清单和输出格式的智能体定义
- 更新：`skills/requesting-code-review/SKILL.md` - 引用 `superpowers:code-reviewer`
- 更新：`skills/subagent-driven-development/SKILL.md` - 引用 `superpowers:code-reviewer`

## v3.2.0（2025-10-18）

### 新功能

**头脑风暴工作流中的设计文档**
- 在 brainstorming 技能中添加了第 4 阶段：设计文档
- 设计文档现在在实现前写入 `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 恢复了在技能转换过程中丢失的原始头脑风暴命令的功能
- 文档在工作树设置和实现计划之前写入
- 通过子智能体测试以验证在时间压力下的合规性

### 破坏性变更

**技能引用命名空间标准化**
- 所有内部技能引用现在使用 `superpowers:` 命名空间前缀
- 更新格式：`superpowers:test-driven-development`（之前只是 `test-driven-development`）
- 影响所有 REQUIRED SUB-SKILL、RECOMMENDED SUB-SKILL 和 REQUIRED BACKGROUND 引用
- 与使用 Skill 工具调用技能的方式保持一致
- 已更新文件：brainstorming、executing-plans、subagent-driven-development、systematic-debugging、testing-skills-with-subagents、writing-plans、writing-skills

### 改进

**设计文档与实现计划命名区分**
- 设计文档使用 `-design.md` 后缀以防止文件名冲突
- 实现计划继续使用现有的 `YYYY-MM-DD-<feature-name>.md` 格式
- 两者都存储在 `docs/plans/` 目录中，命名清晰区分

## v3.1.1（2025-10-17）

### 错误修复

- **修复 README 中的命令语法**（#44）- 更新所有命令引用以使用正确的命名空间语法（`/superpowers:brainstorm` 而非 `/brainstorm`）。插件提供的命令会被 Claude Code 自动添加命名空间以避免插件间冲突。

## v3.1.0（2025-10-17）

### 破坏性变更

**技能名称标准化为小写**
- 所有技能 frontmatter 的 `name:` 字段现在使用小写短横线命名法（kebab-case），与目录名称匹配
- 示例：`brainstorming`、`test-driven-development`、`using-git-worktrees`
- 所有技能公告和交叉引用更新为小写格式
- 确保目录名称、frontmatter 和文档之间的命名一致性

### 新功能

**增强的 brainstorming 技能**
- 添加了显示阶段、活动和工具使用的快速参考表
- 添加了用于追踪进度的可复制工作流清单
- 添加了关于何时重访早期阶段的决策流程图
- 添加了包含具体示例的全面 AskUserQuestion 工具指引
- 添加了"问题模式"部分，解释何时使用结构化问题与开放式问题
- 将关键原则重构为可扫描的表格

**Anthropic 最佳实践集成**
- 添加了 `skills/writing-skills/anthropic-best-practices.md` - Anthropic 官方技能编写指南
- 在 writing-skills SKILL.md 中引用以提供全面指导
- 提供渐进式披露、工作流和评估的模式

### 改进

**技能交叉引用清晰化**
- 所有技能引用现在使用明确的要求标记：
  - `**REQUIRED BACKGROUND:**` - 你必须了解的先决条件
  - `**REQUIRED SUB-SKILL:**` - 必须在工作流中使用的技能
  - `**Complementary skills:**` - 可选但有帮助的相关技能
- 移除了旧路径格式（`skills/collaboration/X` → 直接用 `X`）
- 更新了集成部分，包含分类关系（必需 vs 互补）
- 更新了带有最佳实践的交叉引用文档

**与 Anthropic 最佳实践对齐**
- 修复了描述语法和语态（完全第三人称）
- 添加了用于扫描的快速参考表
- 添加了 Claude 可以复制和追踪的工作流清单
- 对非显而易见的决策点适当使用流程图
- 改进了可扫描的表格格式
- 所有技能都远低于 500 行推荐上限

### 错误修复

- **重新添加了缺失的命令重定向** - 恢复了在 v3.0 迁移中意外删除的 `commands/brainstorm.md` 和 `commands/write-plan.md`
- 修复了 `defense-in-depth` 名称不匹配问题（曾为 `Defense-in-Depth-Validation`）
- 修复了 `receiving-code-review` 名称不匹配问题（曾为 `Code-Review-Reception`）
- 修复了 `commands/brainstorm.md` 中对正确技能名称的引用
- 移除了对不存在的相关技能的引用

### 文档

**writing-skills 改进**
- 更新了带有明确要求标记的交叉引用指南
- 添加了对 Anthropic 官方最佳实践的引用
- 改进了显示正确技能引用格式的示例

## v3.0.1（2025-10-16）

### 变更

我们现在使用 Anthropic 的第一方技能系统！

## v2.0.2（2025-10-12）

### 错误修复

- **修复了本地技能仓库领先于上游时的误报警告** - 初始化脚本在本地仓库有提交领先于上游时会错误地警告"上游有新技能可用"。逻辑现在正确区分三种 git 状态：本地落后（应更新）、本地领先（不警告）和已分叉（应警告）。

## v2.0.1（2025-10-12）

### 错误修复

- **修复了插件上下文中的 session-start 钩子执行问题**（#8，PR #9）- 钩子因"Plugin hook error"静默失败，导致技能上下文无法加载。修复方案：
  - 在 Claude Code 的执行上下文中 BASH_SOURCE 未绑定时，使用 `${BASH_SOURCE[0]:-$0}` 回退
  - 添加 `|| true` 以在过滤状态标志时优雅处理空的 grep 结果

---

# Superpowers v2.0.0 发布说明

## 概述

Superpowers v2.0 通过重大架构调整，使技能更易于访问、维护，并由社区驱动。

最重要的变更是**技能仓库分离**：所有技能、脚本和文档已从插件移入专用仓库（[obra/superpowers-skills](https://github.com/obra/superpowers-skills)）。这将 superpowers 从单体插件转变为一个管理技能仓库本地克隆的轻量级 shim。技能在会话开始时自动更新。用户通过标准 git 工作流 fork 并贡献改进。技能库独立于插件进行版本控制。

除基础设施外，此版本还新增了九个专注于问题解决、研究和架构的新技能。我们用命令式语气和更清晰的结构重写了核心 **using-skills** 文档，使 Claude 更容易理解何时以及如何使用技能。**find-skills** 现在输出可以直接粘贴到 Read 工具的路径，消除了技能发现工作流中的摩擦。

用户体验无缝流畅：插件自动处理克隆、fork 和更新。贡献者发现新架构使改进和分享技能变得轻而易举。此版本为技能作为社区资源快速演进奠定了基础。

## 破坏性变更

### 技能仓库分离

**最重大的变更：** 技能不再存放在插件中。它们已移至 [obra/superpowers-skills](https://github.com/obra/superpowers-skills) 的独立仓库。

**这对你意味着什么：**

- **首次安装：** 插件自动将技能克隆到 `~/.config/superpowers/skills/`
- **Fork：** 设置过程中，如果安装了 `gh`，你将获得 fork 技能仓库的选项
- **更新：** 技能在会话开始时自动更新（尽可能快速合并）
- **贡献：** 在分支上工作，本地提交，向上游提交 PR
- **不再有遮蔽：** 旧的双层系统（个人/核心）被单仓库分支工作流取代

**迁移：**

如果你有现有安装：
1. 你的旧 `~/.config/superpowers/.git` 将备份到 `~/.config/superpowers/.git.bak`
2. 旧技能将备份到 `~/.config/superpowers/skills.bak`
3. 将在 `~/.config/superpowers/skills/` 创建 obra/superpowers-skills 的新克隆

### 已移除功能

- **个人 superpowers 覆盖系统** - 被 git 分支工作流取代
- **setup-personal-superpowers 钩子** - 被 initialize-skills.sh 取代

## 新功能

### 技能仓库基础设施

**自动克隆和设置**（`lib/initialize-skills.sh`）
- 首次运行时克隆 obra/superpowers-skills
- 如果安装了 GitHub CLI，提供 fork 创建选项
- 正确设置 upstream/origin 远程
- 处理从旧安装的迁移

**自动更新**
- 每次会话开始时从追踪远程拉取
- 可以时自动快速合并
- 当需要手动同步时通知（分支已分叉）
- 使用 pulling-updates-from-skills-repository 技能进行手动同步

### 新技能

**问题解决技能**（`skills/problem-solving/`）
- **collision-zone-thinking** - 将不相关概念强行结合以产生新见解
- **inversion-exercise** - 翻转假设以揭示隐藏约束
- **meta-pattern-recognition** - 发现跨领域的通用原则
- **scale-game** - 在极端情况下测试以揭露基本事实
- **simplification-cascades** - 寻找能消除多个组件的见解
- **when-stuck** - 分派到正确的问题解决技术

**研究技能**（`skills/research/`）
- **tracing-knowledge-lineages** - 了解思想随时间的演变

**架构技能**（`skills/architecture/`）
- **preserving-productive-tensions** - 保留多种有效方法，而非强制过早解决

### 技能改进

**using-skills（原 getting-started）**
- 从 getting-started 重命名为 using-skills
- 以命令式语气完全重写（v4.0.0）
- 前置关键规则
- 为所有工作流添加了"原因"解释
- 引用中始终包含 /SKILL.md 后缀
- 更清晰地区分严格规则和灵活模式

**writing-skills**
- 从 using-skills 移入交叉引用指南
- 添加了令牌效率部分（字数目标）
- 改进了 CSO（Claude 搜索优化）指南

**sharing-skills**
- 更新为新的分支和 PR 工作流（v2.0.0）
- 移除了个人/核心分离的引用

**pulling-updates-from-skills-repository**（新增）
- 与上游同步的完整工作流
- 取代旧的"updating-skills"技能

### 工具改进

**find-skills**
- 现在输出带有 /SKILL.md 后缀的完整路径
- 使路径可直接用于 Read 工具
- 更新了帮助文本

**skill-run**
- 从 scripts/ 移至 skills/using-skills/
- 改进了文档

### 插件基础设施

**Session Start 钩子**
- 现在从技能仓库位置加载
- 在会话开始时显示完整技能列表
- 打印技能位置信息
- 显示更新状态（更新成功/落后于上游）
- 将"技能落后"警告移至输出末尾

**环境变量**
- `SUPERPOWERS_SKILLS_ROOT` 设置为 `~/.config/superpowers/skills`
- 在所有路径中一致使用

## 错误修复

- 修复了 fork 时重复添加 upstream 远程的问题
- 修复了 find-skills 在输出中双倍显示"skills/"前缀的问题
- 从 session-start 中移除了过时的 setup-personal-superpowers 调用
- 修复了钩子和命令中的路径引用

## 文档

### README
- 更新为新的技能仓库架构
- 添加了 superpowers-skills 仓库的显著链接
- 更新了自动更新描述
- 修复了技能名称和引用
- 更新了元技能列表

### 测试文档
- 添加了全面的测试清单（`docs/TESTING-CHECKLIST.md`）
- 创建了用于测试的本地市场配置
- 记录了手动测试场景

## 技术细节

### 文件变更

**新增：**
- `lib/initialize-skills.sh` - 技能仓库初始化和自动更新
- `docs/TESTING-CHECKLIST.md` - 手动测试场景
- `.claude-plugin/marketplace.json` - 本地测试配置

**删除：**
- `skills/` 目录（82 个文件）- 现在在 obra/superpowers-skills
- `scripts/` 目录 - 现在在 obra/superpowers-skills/skills/using-skills/
- `hooks/setup-personal-superpowers.sh` - 已过时

**修改：**
- `hooks/session-start.sh` - 使用来自 ~/.config/superpowers/skills 的技能
- `commands/brainstorm.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/write-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `commands/execute-plan.md` - 更新路径为 SUPERPOWERS_SKILLS_ROOT
- `README.md` - 针对新架构完全重写

### 提交历史

此版本包含：
- 20 余次提交用于技能仓库分离
- PR #1：Amplifier 启发的问题解决和研究技能
- PR #2：个人 superpowers 覆盖系统（后来被取代）
- 多次技能改进和文档改进

## 升级说明

### 全新安装

```bash
# 在 Claude Code 中
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

插件会自动处理一切。

### 从 v1.x 升级

1. **备份你的个人技能**（如果有的话）：
   ```bash
   cp -r ~/.config/superpowers/skills ~/superpowers-skills-backup
   ```

2. **更新插件：**
   ```bash
   /plugin update superpowers
   ```

3. **在下次会话开始时：**
   - 旧安装将自动备份
   - 将克隆新的技能仓库
   - 如果你有 GitHub CLI，将提供 fork 选项

4. **迁移个人技能**（如果有的话）：
   - 在你的本地技能仓库中创建分支
   - 从备份中复制你的个人技能
   - 提交并推送到你的 fork
   - 考虑通过 PR 贡献回社区

## 下一步计划

### 对于用户

- 探索新的问题解决技能
- 尝试基于分支的技能改进工作流
- 向社区贡献技能

### 对于贡献者

- 技能仓库现在位于 https://github.com/obra/superpowers-skills
- Fork → 分支 → PR 工作流
- TDD 文档方法见 skills/meta/writing-skills/SKILL.md

## 已知问题

目前无已知问题。

## 致谢

- 问题解决技能受 Amplifier 模式启发
- 社区贡献和反馈
- 对技能有效性进行了广泛测试和迭代

---

**完整变更日志：** https://github.com/obra/superpowers/compare/dd013f6...main
**技能仓库：** https://github.com/obra/superpowers-skills
**问题反馈：** https://github.com/obra/superpowers/issues
