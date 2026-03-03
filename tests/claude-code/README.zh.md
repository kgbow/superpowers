# Claude Code 技能测试

使用 Claude Code CLI 对 superpowers 技能进行自动化测试。

## 概述

本测试套件验证技能是否正确加载，以及 Claude 是否按预期遵循这些技能。测试以无头模式（`claude -p`）调用 Claude Code，并验证其行为。

## 要求

- 已安装 Claude Code CLI 并添加到 PATH 中（`claude --version` 应可正常运行）
- 已安装本地 superpowers 插件（安装方法请参见主 README）

## 运行测试

### 运行所有快速测试（推荐）：
```bash
./run-skill-tests.sh
```

### 运行集成测试（耗时较长，需 10-30 分钟）：
```bash
./run-skill-tests.sh --integration
```

### 运行特定测试：
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 以详细模式运行：
```bash
./run-skill-tests.sh --verbose
```

### 设置自定义超时时间：
```bash
./run-skill-tests.sh --timeout 1800  # 集成测试使用 30 分钟
```

## 测试结构

### test-helpers.sh
技能测试的通用函数：
- `run_claude "prompt" [timeout]` - 使用提示词运行 Claude
- `assert_contains output pattern name` - 验证模式存在
- `assert_not_contains output pattern name` - 验证模式不存在
- `assert_count output pattern count name` - 验证精确计数
- `assert_order output pattern_a pattern_b name` - 验证顺序
- `create_test_project` - 创建临时测试目录
- `create_test_plan project_dir` - 创建示例计划文件

### 测试文件

每个测试文件：
1. 引入 `test-helpers.sh`
2. 使用特定提示词运行 Claude Code
3. 使用断言验证预期行为
4. 成功时返回 0，失败时返回非零值

## 测试示例

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# 询问 Claude 关于技能的信息
output=$(run_claude "What does the my-skill skill do?" 30)

# 验证响应
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 当前测试

### 快速测试（默认运行）

#### test-subagent-driven-development.sh
测试技能内容和要求（约 2 分钟）：
- 技能加载与可访问性
- 工作流排序（规范合规性优先于代码质量）
- 已记录自审要求
- 已记录计划读取效率
- 已记录规范合规审查员的质疑态度
- 已记录审查循环
- 已记录任务上下文提供

### 集成测试（使用 --integration 标志）

#### test-subagent-driven-development-integration.sh
完整工作流执行测试（约 10-30 分钟）：
- 创建包含 Node.js 配置的真实测试项目
- 创建包含 2 个任务的实施计划
- 使用 subagent-driven-development 执行计划
- 验证实际行为：
  - 计划在开始时读取一次（而非每个任务读取一次）
  - 在子代理提示中提供完整任务文本
  - 子代理在报告前执行自审
  - 规范合规审查在代码质量审查之前进行
  - 规范审查员独立读取代码
  - 生成可运行的实现
  - 测试通过
  - 创建正确的 git 提交

**测试内容：**
- 工作流是否能端到端正常运行
- 改进措施是否实际生效
- 子代理是否正确遵循技能
- 最终代码是否可运行且经过测试

## 添加新测试

1. 创建新测试文件：`test-<skill-name>.sh`
2. 引入 test-helpers.sh
3. 使用 `run_claude` 和断言编写测试
4. 将其添加到 `run-skill-tests.sh` 的测试列表中
5. 设置可执行权限：`chmod +x test-<skill-name>.sh`

## 超时时间注意事项

- 默认超时时间：每个测试 5 分钟
- Claude Code 可能需要一定时间才能响应
- 如有需要，使用 `--timeout` 调整
- 测试应保持精简，避免运行时间过长

## 调试失败的测试

使用 `--verbose` 可查看完整的 Claude 输出：
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

不使用 verbose 时，仅在失败时显示输出。

## CI/CD 集成

在 CI 中运行：
```bash
# 为 CI 环境指定明确的超时时间
./run-skill-tests.sh --timeout 900

# 退出码 0 表示成功，非零表示失败
```

## 注意事项

- 测试验证的是技能*指令*，而非完整执行过程
- 完整工作流测试会非常耗时
- 重点验证关键技能要求
- 测试应具有确定性
- 避免测试实现细节
