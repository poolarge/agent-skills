# AGENTS.md

本文件为 AI 编程代理（Claude Code、Cursor、Copilot、Antigravity 等）在使用本仓库代码时提供指导。

## 仓库概述

面向高级软件工程师的 Claude.ai 和 Claude Code 技能集合。技能是打包的指令和脚本，用于扩展 Claude 和你的编程代理的能力。

## OpenCode 集成

OpenCode 使用由 `skill` 工具和本仓库 `/skills` 目录驱动的**技能驱动执行模型**。

### 核心规则

- 如果任务匹配某个技能，你必须调用它
- 技能位于 `skills/<skill-name>/SKILL.md`
- 如果有适用的技能，永远不要直接实现
- 始终严格按照技能指令执行（不要部分应用）

### 意图 → 技能映射

代理应自动将用户意图映射到技能：

- 功能 / 新功能 → `spec-driven-development`，然后 `incremental-implementation`、`test-driven-development`
- 规划 / 分解 → `planning-and-task-breakdown`
- Bug / 故障 / 异常行为 → `debugging-and-error-recovery`
- 代码审查 → `code-review-and-quality`
- 重构 / 简化 → `code-simplification`
- API 或接口设计 → `api-and-interface-design`
- UI 工作 → `frontend-ui-engineering`

### 生命周期映射（隐式命令）

OpenCode 不支持 `/spec` 或 `/plan` 这样的斜杠命令。

相反，代理必须在内部遵循此生命周期：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

### 执行模型

对于每个请求：

1. 判断是否有任何技能适用（哪怕只有 1% 的可能）
2. 使用 `skill` 工具调用适当的技能
3. 严格遵循技能工作流
4. 只有在必需步骤（规格、计划等）完成后才能进入实现阶段

### 反自我辩解

以下想法是错误的，必须忽略：

- "这太小了，不需要用技能"
- "我可以直接快速实现"
- "我先收集一下上下文"

正确行为：

- 始终先检查并使用技能

这确保 OpenCode 的行为与 Claude Code 类似，具有完整的工作流强制执行。

## 编排：角色、技能和命令

本仓库有三个可组合的层。它们有不同的职责，不应混淆：

- **技能**（`skills/<name>/SKILL.md`）— 包含步骤和退出标准的工作流。是"怎么做"。当意图匹配时必须经过的环节。
- **角色**（`agents/<role>.md`）— 具有特定视角和输出格式的角色。是"谁来做"。
- **斜杠命令**（`.claude/commands/*.md`）— 面向用户的入口。是"何时做"。编排层。

组合规则：**用户（或斜杠命令）是编排者。角色不会调用其他角色。** 一个角色可以调用技能。

本仓库认可的唯一多角色编排模式是**并行扇出加合并步骤**——由 `/ship` 使用，并发运行 `code-reviewer`、`security-auditor` 和 `test-engineer` 并综合它们的报告。不要构建一个决定调用哪个其他角色的"路由"角色；那是斜杠命令和意图映射的工作。

参见 [agents/README.md](agents/README.md) 了解决策矩阵，参见 [references/orchestration-patterns.md](references/orchestration-patterns.md) 了解完整的模式目录。

**Claude Code 互操作：** `agents/` 中的角色可作为 Claude Code 子代理（从此插件的 `agents/` 目录自动发现）和代理团队成员（按名称引用进行生成）。两个平台约束与我们的规则一致：子代理不能生成其他子代理，团队不能嵌套。插件代理会静默忽略 `hooks`、`mcpServers` 和 `permissionMode` frontmatter 字段。

## 创建新技能

### 目录结构

```
skills/
  {skill-name}/           # kebab-case 目录名
    SKILL.md              # 必需：技能定义
    scripts/              # 必需：可执行脚本
      {script-name}.sh    # Bash 脚本（推荐）
  {skill-name}.zip        # 必需：用于分发的打包文件
```

### 命名约定

- **技能目录**：`kebab-case`（如 `web-quality`）
- **SKILL.md**：始终大写，始终使用此确切文件名
- **脚本**：`kebab-case.sh`（如 `deploy.sh`、`fetch-logs.sh`）
- **Zip 文件**：必须与目录名完全匹配：`{skill-name}.zip`

### SKILL.md 格式

```markdown
---
name: {skill-name}
description: {一句话描述何时使用此技能。包含触发短语如"Deploy my app"、"Check logs"等。}
---

# {技能标题}

{简要描述技能的作用。}

## 工作原理

{编号列表解释技能的工作流}

## 使用方法

```bash
bash /mnt/skills/user/{skill-name}/scripts/{script}.sh [args]
```

**参数：**
- `arg1` - 描述（默认为 X）

**示例：**
{展示 2-3 个常见用法模式}

## 输出

{展示用户将看到的示例输出}

## 向用户展示结果

{Claude 向用户展示结果时的格式模板}

## 故障排除

{常见问题和解决方案，特别是网络/权限错误}
```

### 上下文效率最佳实践

技能按需加载——启动时仅加载技能名称和描述。完整的 `SKILL.md` 仅在代理判断技能相关时才加载到上下文中。为最小化上下文使用：

- **保持 SKILL.md 在 500 行以内** — 将详细参考材料放在单独的文件中
- **编写具体的描述** — 帮助代理准确知道何时激活技能
- **使用渐进式展示** — 引用支持文件，仅在需要时才读取
- **优先使用脚本而非内联代码** — 脚本执行不消耗上下文（仅输出消耗）
- **文件引用只深入一层** — 从 SKILL.md 直接链接到支持文件

### 脚本要求

- 使用 `#!/bin/bash` shebang
- 使用 `set -e` 实现快速失败
- 将状态消息写入 stderr：`echo "Message" >&2`
- 将机器可读的输出（JSON）写入 stdout
- 包含临时文件的清理 trap
- 脚本路径引用为 `/mnt/skills/user/{skill-name}/scripts/{script}.sh`

### 创建 Zip 包

创建或更新技能后：

```bash
cd skills
zip -r {skill-name}.zip {skill-name}/
```

### 最终用户安装

为用户记录以下两种安装方法：

**Claude Code：**
```bash
cp -r skills/{skill-name} ~/.claude/skills/
```

**claude.ai：**
将技能添加到项目知识库，或将 SKILL.md 内容粘贴到对话中。

如果技能需要网络访问，请指导用户在 `claude.ai/settings/capabilities` 添加所需域名。
