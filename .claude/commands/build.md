---
description: 逐步实现下一个任务——构建、测试、验证、提交
---

调用 agent-skills:incremental-implementation 技能，配合 agent-skills:test-driven-development。

从计划中选取下一个待办任务。对每个任务：

1. 读取任务的验收标准
2. 加载相关上下文（现有代码、模式、类型）
3. 为预期行为编写失败的测试（RED）
4. 实现最小代码使测试通过（GREEN）
5. 运行完整测试套件检查回归
6. 运行构建验证编译
7. 以描述性消息提交
8. 标记任务完成并转向下一个

如果任何步骤失败，遵循 agent-skills:debugging-and-error-recovery 技能。
