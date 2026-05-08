---
description: 进行五轴代码审查——正确性、可读性、架构、安全、性能
---

调用 agent-skills:code-review-and-quality 技能。

审查当前更改（暂存或近期提交）的所有五个轴：

1. **正确性** — 是否匹配规范？边界情况是否处理？测试是否充分？
2. **可读性** — 名称是否清晰？逻辑是否直观？组织是否良好？
3. **架构** — 是否遵循既有模式？边界是否清晰？抽象层次是否恰当？
4. **安全** — 输入是否验证？密钥是否安全？认证是否检查？（使用 security-and-hardening 技能）
5. **性能** — 是否有 N+1 查询？是否有无边界操作？（使用 performance-optimization 技能）

将发现分类为 Critical、Important 或 Suggestion。
输出结构化审查，包含具体的 file:line 引用和修复建议。
