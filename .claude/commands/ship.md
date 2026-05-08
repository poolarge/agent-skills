---
description: 通过并行扇出到专家 persona 运行发布前检查清单，然后综合出 go/no-go 决策
---

调用 agent-skills:shipping-and-launch 技能。

`/ship` 是一个 **扇出编排器**。它并行运行三个专家 persona 对当前变更进行审查，然后将它们的报告合并为单一 go/no-go 决策及回滚计划。各 persona 独立运行——无共享状态、无顺序依赖——这正是并行执行在此安全且有用的原因。

## 阶段 A — 并行扇出

使用 Agent 工具并发派生三个子 agent。**在单次助手轮次中发出所有三个 Agent 工具调用以使它们并行执行**——顺序调用会违背此命令的目的。

在 Claude Code 中，每次调用传递与 persona `name` 字段匹配的 `subagent_type`：

1. **`code-reviewer`** — 对暂存更改或近期提交进行五轴审查（正确性、可读性、架构、安全、性能）。输出标准审查模板。
2. **`security-auditor`** — 运行漏洞和威胁模型审查。检查 OWASP Top 10、密钥处理、认证/授权、依赖 CVE。输出标准审计报告。
3. **`test-engineer`** — 分析变更的测试覆盖。识别正常路径、边界情况、错误路径和并发场景的覆盖缺口。输出标准覆盖分析。

在没有 Agent 工具的其他 harness 中，顺序调用各 persona 的系统提示，将它们的输出视为并行返回——合并阶段仍然有效。

约束（来自 Claude Code 的子 agent 模型）：
- 子 agent 不能派生其他子 agent——不要让一个 persona 向另一个委托。
- 每个子 agent 有自己的上下文窗口，仅向此主会话返回报告。
- 如果你需要互相对话而非仅回报的队友，使用 Claude Code Agent Teams 并引用这些 persona 作为队友类型（参见 `references/orchestration-patterns.md`）。

**Persona 解析。** 如果你在 `.claude/agents/` 或 `~/.claude/agents/` 中定义了自己的 `code-reviewer`、`security-auditor` 或 `test-engineer`，它们优先于本插件的版本——`/ship` 自动使用你的自定义。这是有意设计：插件子 agent 在 Claude Code 范围优先表底部，用户级定义按设计胜出。

## 阶段 B — 在主上下文中合并

三个报告都回来后，主 agent（非子 persona）综合它们：

1. **代码质量** — 聚合 `code-reviewer` 的 Critical/Important 发现及任何失败的测试、lint 或构建输出。消除审查者之间的重复。
2. **安全** — 将 `security-auditor` 的任何 Critical/High 发现提升为发布阻塞项。与 `code-reviewer` 的安全轴交叉对照。
3. **性能** — 从 `code-reviewer` 的性能轴提取；如适用交叉检查 Core Web Vitals。
4. **无障碍** — 验证键盘导航、屏幕阅读器支持、对比度（三个 persona 未覆盖——直接在此处理，或调用无障碍检查清单）。
5. **基础设施** — 环境变量、迁移、监控、功能标志。直接验证。
6. **文档** — README、ADR、changelog。直接验证。

## 阶段 C — 决策和回滚

产出单一输出：

```markdown
## Ship Decision: GO | NO-GO

### Blockers (必须在发布前修复)
- [来源 persona: Critical 发现 + file:line]

### Recommended fixes (应在发布前修复)
- [来源 persona: Important 发现 + file:line]

### Acknowledged risks (仍然发布)
- [风险 + 缓解措施]

### Rollback plan
- Trigger conditions: [会触发回滚的信号]
- Rollback procedure: [确切步骤]
- Recovery time objective: [目标]

### Specialist reports (完整)
- [code-reviewer report]
- [security-auditor report]
- [test-engineer report]
```

## 规则

1. 阶段 A 的三个 persona 并行运行——绝不顺序。
2. Persona 不互相调用。主 agent 在阶段 B 合并。
3. 回滚计划在任何 GO 决策前是必须的。
4. 如果任何 persona 返回 Critical 发现，默认裁决为 NO-GO，除非用户明确接受风险。
5. **仅在以下条件全部满足时跳过扇出：** 变更涉及 2 个文件或更少、diff 不到 50 行、且不涉及认证、支付、数据访问或配置/环境变量。否则默认扇出。`/ship` 专为面向生产的变更设计——当影响范围不可忽略时，即使 diff 看起来很小，也要运行并行审查。
