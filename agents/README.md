# Agent 人格

专精人格只扮演单一角色、持单一视角。每个 persona 是一个 Markdown 文件，由你的运行框架（Claude Code、Cursor、Copilot 等）作为系统提示词消费。

| 人格 | 角色 | 最适合 |
|---------|------|----------|
| [code-reviewer](code-reviewer.md) | 高级主任工程师 | 合并前的五维度审查 |
| [security-auditor](security-auditor.md) | 安全工程师 | 漏洞检测、OWASP 风格审计 |
| [test-engineer](test-engineer.md) | QA 工程师 | 测试策略、覆盖率分析、Prove-It 模式 |

## 人格与技能、命令的关系

三层结构，各司其职：

| 层级 | 是什么 | 示例 | 组合角色 |
|-------|-----------|---------|------------------|
| **Skill（技能）** | 包含步骤和退出条件的工作流 | `code-review-and-quality` | *怎么做* — 在人格或命令内部调用 |
| **Persona（人格）** | 带有视角和输出格式的角色 | `code-reviewer` | *谁来做* — 采用特定视角，产出报告 |
| **Command（命令）** | 面向用户的入口 | `/review`, `/ship` | *何时做* — 组合人格和技能 |

用户（或斜杠命令）是编排者。**人格不会调用其他人格。** 技能是人格工作流中的必要环节。

## 何时使用哪个

### 直接调用人格
当你需要对当前变更获取单一视角、且用户在参与时，选择此方式。

- "Review this PR" → 直接调用 `code-reviewer`
- "Are there security issues in `auth.ts`?" → 直接调用 `security-auditor`
- "What tests are missing for the checkout flow?" → 直接调用 `test-engineer`

### 斜杠命令（单人格封装）
当你有一个可重复的工作流、且不想每次重新解释时，选择此方式。

- `/review` → 封装 `code-reviewer` 与项目的审查技能
- `/test` → 封装 `test-engineer` 与 TDD 技能

### 斜杠命令（编排者 — 扇出）
仅当**独立**的调查可以并行运行、并产生由单个 agent 合并的报告时，才选择此方式。

- `/ship` → 并行扇出到 `code-reviewer` + `security-auditor` + `test-engineer`，然后将它们的报告综合为通过/不通过决策

这是本仓库唯一认可的编排模式。完整模式目录和反模式参见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

## 决策矩阵

```
工作是否是对单一产物的单一视角？
├── 是 → 直接调用人格
└── 否 → 子任务是否独立（无共享可变状态，无顺序依赖）？
         ├── 是 → 带并行扇出的斜杠命令（如 /ship）
         └── 否 → 由用户依次运行的斜杠命令（/spec → /plan → /build → /test → /review）
```

## 实例：有效的编排

`/ship` 是本仓库中典型的扇出编排器：

```
/ship
  ├── (并行) code-reviewer    → 审查报告
  ├── (并行) security-auditor → 审计报告
  └── (并行) test-engineer    → 覆盖率报告
                  ↓
        合并阶段（主 agent）
                  ↓
        通过/不通过决策 + 回滚计划
```

为什么有效：
- 每个子 agent 操作同一个 diff，但产出**不同的视角**
- 它们之间没有依赖 → 真正的并行，真实的挂钟时间节省
- 每个在新的上下文窗口中运行 → 主会话保持整洁
- 合并步骤很小且受益于完整上下文，因此留在主 agent 中

## 实例：无效的编排（不要这样构建）

一个 `meta-orchestrator` 人格，其职责是"决定调用哪个人格"：

```
/work-on-pr → meta-orchestrator
                  ↓ (决定"这需要审查")
              code-reviewer
                  ↓ (返回)
              meta-orchestrator (转述结果)
                  ↓
              用户
```

为什么失败：
- 纯路由层，没有领域价值
- 增加了两次转述环节 → 信息丢失 + 2 倍 token 成本
- 用户已经知道自己想要审查；让他们直接调用 `/review`
- 复制了斜杠命令和 `AGENTS.md` 意图映射已经在做的工作

## 人格规则

1. 人格是单一角色、单一输出格式。如果你发现自己要加入第二个角色，请创建第二个人格。
2. **人格不会调用其他人格。** 组合是斜杠命令或用户的职责。在 Claude Code 上这也是硬性平台约束——*"subagents cannot spawn other subagents"*——所以这条规则自然生效。
3. 人格可以调用技能（*怎么做*）。
4. 每个人格文件末尾都有"Composition"块，说明其适用位置。

## Claude Code 互操作

本仓库中的人格设计为无需修改即可作为 Claude Code 子 agent 和 Agent Teams 队友使用：

- **作为子 agent：** 启用此插件时自动发现（无需路径配置）。使用 Agent 工具并指定 `subagent_type: code-reviewer`（或 `security-auditor`、`test-engineer`）。`/ship` 是典型示例。
- **作为 Agent Teams 队友**（实验性，需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）：在生成队友时引用相同的人格名称。人格正文会**追加到**队友的系统提示词中作为额外指令（而非替换），因此你的人格文本叠加在主 agent 安装的团队协调指令（SendMessage、task-list 工具等）之上。

子 agent 只将结果报告回主 agent。Agent Teams 允许队友之间直接消息。当报告足够时使用子 agent；当子 agent 需要相互质疑发现时（例如竞争假设调试）使用 Agent Teams。完整映射参见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

插件 agent 不支持 `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些字段会被静默忽略。在此编写新人格时不要依赖它们。

## 添加新人格

1. 创建 `agents/<role>.md`，使用与现有人格相同的 frontmatter 格式。
2. 定义角色、范围、输出格式和规则。
3. 在底部添加 **Composition** 块（何时直接调用 / 通过什么命令调用 / 不要从其他人格调用）。
4. 将人格添加到本文件顶部的表格中。
5. 如果该人格启用了新的编排模式，在 `references/orchestration-patterns.md` 中记录，而不是在人格文件本身中发明模式。
