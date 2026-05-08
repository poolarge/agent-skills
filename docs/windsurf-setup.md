# 在 Windsurf 中使用 agent-skills

## 配置

### 项目规则

Windsurf 使用 `.windsurfrules` 存放项目特定的代理指令：

```bash
# 从最重要的 skill 创建合并后的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md >> .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .windsurfrules
```

### 全局规则

对于需要在所有项目中使用的 skill，将它们添加到 Windsurf 的全局规则中：

1. 打开 Windsurf → 设置 → AI → Global Rules
2. 粘贴你最常用的 skill 内容

## 推荐配置

保持 `.windsurfrules` 聚焦于 2-3 个核心 skill，以控制在上下文限制内：

```
# .windsurfrules
# 本项目的核心 agent-skills

[Paste test-driven-development SKILL.md]

---

[Paste incremental-implementation SKILL.md]

---

[Paste code-review-and-quality SKILL.md]
```

## 使用提示

1. **有选择地加载** — Windsurf 的上下文是有限的。选择解决你最大质量缺口的 skill。
2. **在对话中引用** — 在处理特定阶段时，将额外的 skill 内容粘贴到聊天中（例如，在构建认证功能时粘贴 `security-and-hardening`）。
3. **将参考资料用作检查清单** — 粘贴 `references/security-checklist.md` 并让 Windsurf 逐项验证。
