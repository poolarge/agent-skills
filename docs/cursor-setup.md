# 在 Cursor 中使用 agent-skills

## 配置

### 方式一：规则目录（推荐）

Cursor 支持 `.cursor/rules/` 目录用于项目特定规则：

```bash
# 创建规则目录
mkdir -p .cursor/rules

# 将需要的 skill 复制为规则
cp /path/to/agent-skills/skills/test-driven-development/SKILL.md .cursor/rules/test-driven-development.md
cp /path/to/agent-skills/skills/code-review-and-quality/SKILL.md .cursor/rules/code-review-and-quality.md
cp /path/to/agent-skills/skills/incremental-implementation/SKILL.md .cursor/rules/incremental-implementation.md
```

此目录中的规则会自动加载到 Cursor 的上下文中。

### 方式二：.cursorrules 文件

在项目根目录创建 `.cursorrules` 文件，内联写入核心 skill：

```bash
# 生成合并后的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .cursorrules
echo "\n---\n" >> .cursorrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .cursorrules
```

## 推荐配置

### 核心 Skill（始终加载）

将这些添加到 `.cursor/rules/`：

1. `test-driven-development.md` — TDD 工作流和 Prove-It 模式
2. `code-review-and-quality.md` — 五维度审查
3. `incremental-implementation.md` — 以小的可验证切片构建

### 阶段专用 Skill（按需加载）

针对阶段性工作，根据需要创建额外的规则文件：

- `spec-development.md` -> `spec-driven-development/SKILL.md`
- `frontend-ui.md` -> `frontend-ui-engineering/SKILL.md`
- `security.md` -> `security-and-hardening/SKILL.md`
- `performance.md` -> `performance-optimization/SKILL.md`

在处理相关任务时将这些文件添加到 `.cursor/rules/`，完成后移除以管理上下文限制。

## 使用提示

1. **不要一次加载所有 skill** — Cursor 有上下文限制。加载 2-3 个核心 skill 作为规则，根据需要添加阶段专用 skill。
2. **显式引用 skill** — 告诉 Cursor "按照 test-driven-development 规则处理此变更"，以确保它读取已加载的规则。
3. **使用 agent 进行审查** — 复制 `agents/code-reviewer.md` 内容，让 Cursor "使用此代码审查框架审查此 diff。"
4. **按需加载参考资料** — 在处理性能问题时，将 `performance.md` 添加到 `.cursor/rules/` 或直接粘贴检查清单内容。
