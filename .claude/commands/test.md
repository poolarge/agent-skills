---
description: 运行 TDD 工作流——编写失败测试、实现、验证。对于 bug，使用 Prove-It 模式。
---

调用 agent-skills:test-driven-development 技能。

对于新功能：
1. 编写描述预期行为的测试（应该失败）
2. 实现代码使其通过
3. 在保持测试通过的同时重构

对于 bug 修复（Prove-It 模式）：
1. 编写重现 bug 的测试（必须失败）
2. 确认测试失败
3. 实现修复
4. 确认测试通过
5. 运行完整测试套件检查回归

对于浏览器相关问题，同时调用 agent-skills:browser-testing-with-devtools 以用 Chrome DevTools MCP 验证。
