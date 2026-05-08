# agent-skills 入门指南

agent-skills 适用于任何接受 Markdown 指令的 AI 编程代理。本指南介绍通用方法。针对特定工具的配置，请参阅对应的专用指南。

## Skill 的工作原理

每个 skill 都是一个 Markdown 文件（`SKILL.md`），描述了特定的工程工作流。当加载到代理的上下文中时，代理会遵循该工作流——包括验证步骤、需要避免的反模式以及退出标准。

**Skill 不是参考文档。** 它们是代理遵循的逐步流程。

## 快速开始（任何代理）

### 1. 克隆仓库

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

### 2. 选择一个 skill

浏览 `skills/` 目录。每个子目录包含一个 `SKILL.md`，其中包括：
- **何时使用** — 表明该 skill 适用的触发条件
- **流程** — 逐步工作流
- **验证** — 如何确认工作已完成
- **常见自我辩解** — 代理可能用来跳过步骤的借口
- **红旗信号** — 表明 skill 正在被违反的迹象

### 3. 将 skill 加载到代理中

将相关的 `SKILL.md` 内容复制到代理的系统提示、规则文件或对话中。最常见的方法：

**系统提示：** 在会话开始时粘贴 skill 内容。

**规则文件：** 将 skill 内容添加到项目的规则文件（CLAUDE.md、.cursorrules 等）中。

**对话：** 在给出指令时引用 skill："按照 test-driven-development 流程处理此变更。"

### 4. 使用 meta-skill 进行发现

首先加载 `using-agent-skills` skill。它包含一个流程图，将任务类型映射到相应的 skill。

## 推荐配置

### 最小配置（从这里开始）

将三个核心 skill 加载到规则文件中：

1. **spec-driven-development** — 用于定义要构建什么
2. **test-driven-development** — 用于证明代码可以正常工作
3. **code-review-and-quality** — 用于在合并前验证质量

这三个 skill 覆盖了 AI 辅助开发中最关键的质量缺口。

### 完整生命周期

如需全面覆盖，按阶段加载 skill：

```
启动项目:  spec-driven-development → planning-and-task-breakdown
开发阶段:  incremental-implementation + test-driven-development
合并之前:  code-review-and-quality + security-and-hardening
部署之前:  shipping-and-launch
```

### 按需上下文加载

不要一次加载所有 skill —— 这会浪费上下文。只加载与当前任务相关的 skill：

- 做 UI 工作？加载 `frontend-ui-engineering`
- 调试？加载 `debugging-and-error-recovery`
- 搭建 CI？加载 `ci-cd-and-automation`

## Skill 结构

每个 skill 遵循相同的结构：

```
YAML 前置信息（名称、描述）
├── 概述 — 此 skill 做什么
├── 何时使用 — 触发条件和适用场景
├── 核心流程 — 逐步工作流
├── 示例 — 代码示例和模式
├── 常见自我辩解 — 借口和反驳
├── 红旗信号 — 表明 skill 正在被违反的迹象
└── 验证 — 退出标准检查清单
```

完整规范请参见 [skill-anatomy.md](skill-anatomy.md)。

## 使用 Agent

`agents/` 目录包含预配置的代理角色：

| Agent | 用途 |
|-------|------|
| `code-reviewer.md` | 五维度代码审查 |
| `test-engineer.md` | 测试策略和编写 |
| `security-auditor.md` | 漏洞检测 |

当需要专业审查时加载代理定义。例如，让编程代理"使用 code-reviewer 代理角色审查此变更"并提供代理定义。

## 使用命令

`.claude/commands/` 目录包含 Claude Code 的斜杠命令：

| 命令 | 调用的 Skill |
|------|-------------|
| `/spec` | spec-driven-development |
| `/plan` | planning-and-task-breakdown |
| `/build` | incremental-implementation + test-driven-development |
| `/test` | test-driven-development |
| `/review` | code-review-and-quality |
| `/ship` | shipping-and-launch |

## 使用参考资料

`references/` 目录包含补充检查清单：

| 参考资料 | 配合使用 |
|----------|----------|
| `testing-patterns.md` | test-driven-development |
| `performance-checklist.md` | performance-optimization |
| `security-checklist.md` | security-and-hardening |
| `accessibility-checklist.md` | frontend-ui-engineering |

当需要 skill 未涵盖的详细模式时，加载参考资料。

## 规格和任务工件

`/spec` 和 `/plan` 命令会创建工作工件（`SPEC.md`、`tasks/plan.md`、`tasks/todo.md`）。在工作进行期间，将它们视为**活文档**：

- 在开发过程中将它们保留在版本控制中，以便人类和代理拥有共同的真相来源。
- 当范围或决策发生变化时更新它们。
- 如果你的仓库不需要长期保留这些文件，可以在合并前删除或将该文件夹添加到 `.gitignore` —— 工作流并不要求它们永久存在。

## 提示

1. **对于任何非简单工作，从 spec-driven-development 开始**
2. **编写代码时始终加载 test-driven-development**
3. **不要跳过验证步骤** —— 它们才是关键所在
4. **有选择地加载 skill** —— 更多上下文并不总是更好
5. **使用 agent 进行审查** —— 不同视角能发现不同问题
