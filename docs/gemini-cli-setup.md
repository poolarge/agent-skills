# 在 Gemini CLI 中使用 agent-skills

## 配置

### 方式一：作为 Skill 安装（推荐）

Gemini CLI 拥有原生 skill 系统，会自动发现 `.gemini/skills/` 或 `.agents/skills/` 目录中的 `SKILL.md` 文件。每个 skill 在匹配你的任务时按需激活。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**或从本地克隆安装：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
gemini skills install /path/to/agent-skills/skills/
```

**仅为特定工作区安装：**

```bash
gemini skills install /path/to/agent-skills/skills/ --scope workspace
```

以工作区范围安装的 skill 会放入 `.gemini/skills/`（或 `.agents/skills/`）。用户级别的 skill 放入 `~/.gemini/skills/`。

安装后，通过以下命令验证：

```
/skills list
```

Gemini CLI 会自动将 skill 名称和描述注入到提示中。当它识别到匹配的任务时，会在加载完整指令前请求许可来激活该 skill。

### 方式二：GEMINI.md（持久上下文）

对于希望始终作为持久项目上下文加载的 skill（而非按需激活），将它们添加到项目的 `GEMINI.md` 中：

```bash
# 创建 GEMINI.md，包含核心 skill 作为持久上下文
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md > GEMINI.md
echo -e "\n---\n" >> GEMINI.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> GEMINI.md
```

你也可以通过从单独的文件导入来模块化：

```markdown
# Project Instructions

@skills/test-driven-development/SKILL.md
@skills/incremental-implementation/SKILL.md
```

使用 `/memory show` 验证已加载的上下文，使用 `/memory reload` 在变更后刷新。

> **Skill 与 GEMINI.md 的区别：** Skill 是按需的专业能力，仅在相关时激活，保持上下文窗口整洁。GEMINI.md 提供对每个提示都加载的持久上下文。将阶段专用工作流用 skill，将始终需要的项目约定用 GEMINI.md。

## 推荐配置

### 始终开启（GEMINI.md）

将这些作为持久上下文添加到每个会话：

- `incremental-implementation` — 以小的可验证切片构建
- `code-review-and-quality` — 五维度审查

### 按需使用（Skill）

将这些作为 skill 安装，使其仅在相关时激活：

- `test-driven-development` — 在实现逻辑或修复 bug 时激活
- `spec-driven-development` — 在启动新项目或功能时激活
- `frontend-ui-engineering` — 在构建 UI 时激活
- `security-and-hardening` — 在安全审查时激活
- `performance-optimization` — 在性能优化工作时激活

## 高级配置

### MCP 集成

本技能包中的许多 skill 利用 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 工具与环境交互。例如：

- `browser-testing-with-devtools` 使用 `chrome-devtools` MCP 扩展。
- `performance-optimization` 可以从性能相关的 MCP 工具中受益。

要启用这些功能，请确保在 Gemini CLI 配置（`~/.gemini/config.json`）中安装了相关的 MCP 扩展。

### 会话钩子

Gemini CLI 支持会话生命周期钩子。你可以使用这些钩子在会话开始时自动注入上下文或运行验证脚本。

要复制其他工具中的 `agent-skills` 体验，你可以配置 `SessionStart` 钩子来提醒你可用的 skill 或加载 meta-skill。

### 显式上下文加载

你可以通过在提示中使用 `@` 符号引用来显式加载任何 skill 到当前会话：

```markdown
Use the @skills/test-driven-development/SKILL.md skill to implement this fix.
```

当你想确保遵循特定工作流而不等待自动发现时，这很有用。

## 斜杠命令

仓库在 `.gemini/commands/` 下提供了 7 个斜杠命令，映射到开发生命周期。当从项目根目录运行时，Gemini CLI 会自动发现它们。

| 命令 | 功能 |
|------|------|
| `/spec` | 在编写代码前编写结构化规格说明 |
| `/planning` | 将工作分解为小的、可验证的任务 |
| `/build` | 增量实现下一个任务 |
| `/test` | 运行 TDD 工作流 — 红、绿、重构 |
| `/review` | 五维度代码审查 |
| `/code-simplify` | 在不改变行为的情况下降低复杂度 |
| `/ship` | 通过并行角色分派执行上线前检查清单 |

每个命令会自动调用相应的 skill —— 无需手动加载 skill。

> **注意：** 使用 `/planning` 而不是 `/plan` —— `/plan` 与 Gemini CLI 内部命令名冲突。

## 使用提示

1. **优先使用 skill 而非 GEMINI.md** — Skill 按需激活并保持上下文窗口聚焦。只有希望始终加载的 skill 才放入 GEMINI.md。
2. **Skill 描述很重要** — 每个 SKILL.md 在其前置信息中有一个 `description` 字段，告诉代理何时激活它。本仓库中的描述针对所有支持工具（Claude Code、Gemini CLI 等）的自动发现进行了优化，清楚地说明了 skill *做什么*以及*何时*应该触发。
3. **使用 agent 进行审查** — 在请求结构化代码审查时，复制 `agents/code-reviewer.md` 内容。
4. **结合参考资料使用** — 在处理特定质量领域（如测试或性能）时，引用 `references/` 中的检查清单。
