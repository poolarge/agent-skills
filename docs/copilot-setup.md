# 在 GitHub Copilot 中使用 agent-skills

## 配置

### Copilot 指令

Copilot 支持通过仓库中的 `.github/skills`、`.claude/skills` 或 `.agents/skills` 目录创建 agent skill。

```bash
mkdir -p .github

# 创建核心 skill 文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .github/skills/test-driven-development/SKILL.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md > .github/skills/code-review-and-quality/SKILL.md
```

更多详情，请参阅 [Creating agent skills for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills)。

### 代理角色（agents.md）

Copilot 支持专用代理角色。使用 agent-skills 的代理：

```bash
# 复制代理定义
cp /path/to/agent-skills/agents/code-reviewer.md .github/agents/code-reviewer.md
cp /path/to/agent-skills/agents/test-engineer.md .github/agents/test-engineer.md
cp /path/to/agent-skills/agents/security-auditor.md .github/agents/security-auditor.md
```

在 Copilot Chat 中调用代理：
- `@code-reviewer Review this PR`
- `@test-engineer Analyze test coverage for this module`
- `@security-auditor Check this endpoint for vulnerabilities`

### 自定义指令（用户级别）

对于需要在所有仓库中使用的 skill：

1. 打开 VS Code → 设置 → GitHub Copilot → Custom Instructions
2. 添加你最常用的 skill 摘要

## 推荐配置

### .github/copilot-instructions.md

GitHub Copilot 通过 `.github/copilot-instructions.md` 支持项目级指令。

```markdown
# 项目编码标准

## 测试
- 先写测试再写代码（TDD）
- 对于 bug：先写一个失败的测试，然后修复（Prove-It 模式）
- 测试层级：单元测试 > 集成测试 > 端到端测试（使用能捕获行为的最低层级）
- 每次变更后运行 `npm test`

## 代码质量
- 从五个维度审查：正确性、可读性、架构、安全性、性能
- 每个 PR 必须通过：lint、类型检查、测试、构建
- 代码或版本控制中不得包含密钥

## 实现
- 以小的、可验证的增量构建
- 每个增量：实现 → 测试 → 验证 → 提交
- 永远不要将格式变更与行为变更混在一起

## 边界
- 始终：提交前运行测试、验证用户输入
- 先询问：数据库 schema 变更、新依赖
- 永远不：提交密钥、删除失败测试、跳过验证
```

### 专用代理

在 Copilot Chat 中使用代理进行针对性审查工作流。

## 使用提示

1. **保持指令简洁** — Copilot 指令在聚焦时效果最佳。总结关键规则，而不是包含完整的 skill 文件。
2. **使用 agent 进行审查** — code-reviewer、test-engineer 和 security-auditor 代理专为 Copilot 的代理模型设计。
3. **在聊天中引用** — 在处理特定阶段时，将相关的 skill 内容粘贴到 Copilot Chat 中作为上下文。
4. **结合 PR 审查** — 设置 Copilot 使用 code-reviewer 代理角色审查 PR。
