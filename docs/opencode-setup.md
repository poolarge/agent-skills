# OpenCode 配置

本指南说明如何在 OpenCode 中使用 Agent Skills，其体验尽可能接近 Claude Code（自动 skill 选择、生命周期驱动的工作流、严格的流程执行）。

## 概述

OpenCode 支持自定义 `/commands`，但没有像 Claude Code 那样的原生插件系统或自动 skill 路由。

相反，我们通过以下方式实现同等体验：

- 强大的系统提示（`AGENTS.md`）
- 内置的 `skill` 工具
- 从 `/skills` 目录进行一致的 skill 发现

这创建了一个**代理驱动的工作流**，其中 skill 被自动选择和执行。

虽然可以在 OpenCode 中重建 `/spec`、`/plan` 等命令，但此集成有意采用代理驱动的方式：

- Skill 根据意图自动选择
- 工作流通过 `AGENTS.md` 强制执行
- 无需手动调用命令

这更接近 Claude Code 的实际行为，即 skill 是自动触发而非手动触发的。

---

## 安装

1. 克隆仓库：

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

2. 在 OpenCode 中打开项目。

3. 确保工作区中存在以下文件：

- `AGENTS.md`（根目录）
- `skills/` 目录

无需额外安装。

---

## 工作原理

### 1. Skill 发现

所有 skill 位于：

```
skills/<skill-name>/SKILL.md
```

OpenCode 代理通过 `AGENTS.md` 被指示：

- 检测何时有 skill 适用
- 调用 `skill` 工具
- 严格遵循 skill

### 2. 自动 Skill 调用

代理评估每个请求并将其映射到适当的 skill。

示例：

- "构建一个功能" → `incremental-implementation` + `test-driven-development`
- "设计一个系统" → `spec-driven-development`
- "修复一个 bug" → `debugging-and-error-recovery`
- "审查这段代码" → `code-review-and-quality`

用户**不需要**显式请求 skill。

### 3. 生命周期映射（隐式命令）

开发生命周期以隐式方式编码：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

这替代了 `/spec`、`/plan` 等斜杠命令。

---

## 使用示例

### 示例 1：功能开发

用户：
```
为此应用添加认证功能
```

代理行为：
- 检测到功能开发任务
- 调用 `spec-driven-development`
- 在编写代码前生成规格说明
- 转入规划和实现 skill

---

### 示例 2：Bug 修复

用户：
```
此端点返回 500 错误
```

代理行为：
- 调用 `debugging-and-error-recovery`
- 复现 → 定位 → 修复 → 添加防护

---

### 示例 3：代码审查

用户：
```
审查此 PR
```

代理行为：
- 调用 `code-review-and-quality`
- 应用结构化审查（正确性、设计、可读性等）

---

## 代理期望（关键）

为了让 OpenCode 正常工作，代理必须遵循以下规则：

- 在行动前始终检查是否有 skill 适用
- 如果有 skill 适用，必须使用它
- 永远不跳过必需的工作流（规格、计划、测试等）
- 不要直接跳到实现

这些规则通过 `AGENTS.md` 强制执行。

---

## 限制

- 没有原生斜杠命令（通过意图映射处理）
- 没有插件系统（通过提示 + 结构处理）
- Skill 调用依赖模型遵从度

尽管如此，工作流在实际使用中仍与 Claude Code 高度一致。

---

## 推荐工作流

只需使用自然语言：

- "设计一个功能"
- "规划此变更"
- "实现这个"
- "修复此 bug"
- "审查这个"

代理会自动选择并执行正确的 skill。

---

## 总结

OpenCode 集成通过组合以下方式工作：

- 结构化 skill（本仓库）
- 强大的代理规则（`AGENTS.md`）
- 通过推理自动调用 skill

这产生了一个**完全代理驱动的、生产级工程工作流**，无需插件或手动命令。
