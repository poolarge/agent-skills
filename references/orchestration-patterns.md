# 编排模式参考

本仓库认可的 agent 编排模式目录，以及需要避免的反模式。在添加协调多个 persona 的新斜杠命令前，或在引入"包装"现有 persona 的新 persona 前，请先阅读此文档。

核心规则：**用户（或斜杠命令）是编排者。Persona 不调用其他 persona。** 技能是 persona 工作流中的必经节点。

---

## 认可的模式

### 1. 直接调用（无编排）

单个 persona、单个视角、单个产出物。默认且最经济的选项。

```
user → code-reviewer → report → user
```

**适用场景：** 工作是对单个产出物的单个视角，可以用一句话描述。

**示例：**
- "Review this PR" → `code-reviewer`
- "Find security issues in `auth.ts`" → `security-auditor`
- "What tests are missing for the checkout flow?" → `test-engineer`

**成本：** 一次往返。始终应与之比较的基线。

---

### 2. 单 persona 斜杠命令

用项目的技能包装单个 persona 的斜杠命令。免去用户每次重复解释工作流。

```
/review → code-reviewer (with code-review-and-quality skill) → report
```

**适用场景：** 同样的单 persona 调用以相同设置反复发生。

**本仓库示例：** `/review`、`/test`、`/code-simplify`。

**成本：** 同直接调用。斜杠命令只是保存的提示。

**反信号：** 如果斜杠命令的主体主要是"决定调用哪个 persona"，删除它，让用户直接调用 persona。

---

### 3. 并行扇出 + 合并

多个 persona 同时对相同输入操作，各自产出独立报告。合并步骤（在主 agent 上下文中）将它们综合为单一决策。

```
                    ┌─→ code-reviewer    ─┐
/ship → fan out  ───┼─→ security-auditor ─┤→ merge → go/no-go + rollback
                    └─→ test-engineer    ─┘
```

**适用场景：**
- 子任务确实相互独立（无共享可变状态、无顺序依赖）
- 每个子 agent 受益于拥有自己的上下文窗口
- 合并步骤足够小，能留在主上下文中
- 墙钟延迟重要

**本仓库示例：** `/ship`。

**成本：** N 个并行子 agent 上下文 + 一次合并轮。比直接调用高，但墙钟更快且报告质量更好，因为每个子 agent 专注于其单一视角。

**采纳前的验证检查清单：**
- [ ] 我能同时运行所有子 agent 而没有顺序问题吗？
- [ ] 每个 persona 产出的是不同 *类型* 的发现，而非只是同一发现的不同角度吗？
- [ ] 合并步骤能放入主 agent 的剩余上下文吗？
- [ ] 用户等待时间足够长，并行性确实有意义吗？

如果任何答案为"否"，回退到直接调用或单 persona 命令。

---

### 4. 顺序流水线——用户驱动的斜杠命令

用户按定义顺序运行斜杠命令，在步骤之间携带上下文（或提交历史）。没有编排 agent——用户就是编排者。

```
user runs:  /spec  →  /plan  →  /build  →  /test  →  /review  →  /ship
```

**适用场景：** 工作流有依赖关系（每步需要上一步的输出）且步骤之间的人类判断有价值。

**本仓库示例：** 整个 DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 生命周期。

**成本：** 每步一个子 agent 上下文。编排层免费，因为没有编排 agent。

**为什么不自动化：** 一个 LLM"生命周期编排者"会 (a) 在步骤间丢失细节，因为它必须为交接做摘要，(b) 跳过本应捕获方向错误的人类检查点，(c) 通过改写轮次使 token 成本翻倍。

---

### 5. 研究隔离（上下文保留）

当任务需要阅读大量不应污染主上下文的材料时，派生一个研究子 agent，仅返回摘要。

```
main agent → research sub-agent (reads 50 files) → digest → main agent continues
```

**适用场景：**
- 主会话需要保持专注于下游任务
- 调查结果远小于它消耗的输入
- 决策质量受益于主 agent 有思考空间

**示例：** "Find every call site of this deprecated API across the monorepo," "Summarize what these 30 ADRs say about caching."

**成本：** 一个隔离的子 agent 上下文。只要替代方案是将数百文件加载到主上下文，就值得做。

**在 Claude Code 上，使用内置 `Explore` 子 agent** 而非定义自定义研究 persona。`Explore` 在 Haiku 上运行，被禁止 write/edit 工具，专为此模式构建。仅在 `Explore` 不适合时定义自定义研究子 agent（例如你需要模型无法推断的领域特定系统提示）。

---

## Claude Code 兼容性

本目录是 harness 无关的，但大多数读者会在 Claude Code 上运行。以下是各模式如何映射到 Claude Code 原语——以及平台在哪里为我们强制执行规则。

### Persona 存放位置

插件子 agent 放在插件根目录的 `agents/` 中。本仓库是插件（`.claude-plugin/plugin.json`），所以 `agents/code-reviewer.md`、`agents/security-auditor.md` 和 `agents/test-engineer.md` 在插件启用时自动发现。无需路径配置。

### 子 agent 与 Agent Teams

Claude Code 有两个并行原语。模式 3（并行扇出 + 合并）映射到 **子 agent**。如果你需要互相对话的队友，使用 **Agent Teams**。

| | 子 agent | Agent Teams |
|--|-----------|-------------|
| 协调 | 主 agent 扇出，子 agent 仅回报 | 队友互相发消息，共享任务列表 |
| 上下文 | 每个子 agent 有自己的上下文窗口 | 每个队友有自己的上下文窗口 |
| 适用场景 | 产出报告的独立任务 | 需要讨论的协作工作 |
| 状态 | 稳定 | 实验——需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| 成本 | 较低 | 较高——每个队友是一个独立的 Claude 实例 |

**本仓库的 persona 在两种模式下都工作。** 作为子 agent 派生时（例如 `/ship`），它们向主会话报告发现。作为队友派生时（`Spawn a teammate using the security-auditor agent type…`），它们可以直接挑战彼此的发现。Persona 定义相同；只有派生上下文不同。

一个细微之处：persona 的 `skills` 和 `mcpServers` frontmatter 字段在作为子 agent 运行时生效，但 **作为队友运行时被忽略**——队友从你的项目和用户设置加载技能和 MCP 服务器，与常规会话相同。如果 persona 依赖特定技能或 MCP 服务器，在会话级别配置，使其在两种模式下都可用。

### 平台强制规则

本目录中有两条规则不仅是惯例——Claude Code 会强制执行：

- **"子 agent 不能派生其他子 agent"**（文档原文）。反模式 B（persona 调用 persona）和反模式 D（深层 persona 树）在 Claude Code 上无法存在，因为构造上不允许。
- **"无嵌套团队"**——队友不能派生自己的团队。同样的反模式在团队级别也被阻止。

这意味着你可以采纳本目录的模式，无需担心贡献者意外构建反模式。它们会直接加载失败。

### 需要了解的内置子 agent

在定义自定义子 agent 前，检查以下内置角色是否覆盖：

| 内置 | 目的 |
|----------|---------|
| `Explore` | 只读代码库搜索和分析。用于模式 5（研究隔离）。 |
| `Plan` | 规划模式期间的只读研究。 |
| `general-purpose` | 需要探索和修改的多步任务。 |

不要重新定义这些。在它们之上叠加你的专家 persona（code-reviewer、security-auditor、test-engineer）。

### 插件 agent 的 frontmatter 限制

插件子 agent **不支持** `hooks`、`mcpServers` 或 `permissionMode` frontmatter 字段——这些会被静默忽略。如果未来的 persona 需要这些字段中的任何一个，用户必须将文件复制到 `.claude/agents/` 或 `~/.claude/agents/`。

插件 agent 中 **可用** 的字段：`name`、`description`、`tools`、`disallowedTools`、`model`、`maxTurns`、`skills`、`memory`、`background`、`effort`、`isolation`、`color`、`initialPrompt`。如果想优化成本，可按 persona 使用 `model`（例如 Haiku 用于 `test-engineer` 覆盖扫描，Sonnet 用于 `code-reviewer`，Opus 用于 `security-auditor`）。

### 并行派生多个子 agent

在 Claude Code 中，并行扇出（模式 3）需要在 **单次助手轮次中发出多个 Agent 工具调用**。顺序轮次会序列化执行。`/ship` 明确指出了这一点。任何新的编排命令也应如此。

---

## 完整示例：用 Agent Teams 进行竞争假说调试

此示例展示何时应使用 **Agent Teams** 而非 `/ship` 的子 agent 扇出。两种模式从远处看相似——都派生同样的三个 persona——但价值来源不同。

### 场景

> *Checkout 偶尔在完成前挂起约 30 秒。大约每 50 个 session 发生一次。日志中没有错误。上周发布后开始出现。*

合理的根因（互相排斥，都符合症状）：

1. 新支付确认流程中的竞态条件
2. 一个偶尔落入慢速同步网络调用的认证检查
3. 一个随购物车大小增长的查询缺少索引
4. 一个 SDK 在超时前静默重试的脆弱第三方 API

单个 agent 会选择第一个合理理论然后停止调查。`/ship` 式的子 agent 扇出会让每个 persona 独立报告——但它们的报告永远不会交汇，因此无法排除错误理论。

这正是 Agent Teams 文档描述的场景：*"有多个独立调查者积极尝试推翻彼此，存活下来的理论更可能是真正的根因。"*

### 为什么这 *不是* `/ship` 的工作

| | `/ship`（子 agent） | Agent Teams |
|--|--------------------|-------------|
| 子 agent 看到 | 相同 diff，不同视角 | 共享任务列表，彼此的消息 |
| 输出 | 三份独立报告 → 合并 | 对抗性辩论 → 共识根因 |
| 适用时机 | 你要对已知产出物做出裁决 | 你要在假说中 *寻找* 产出物 |

`/ship` 是裁决；Agent Teams 是调查。

### 设置（一次性，按环境）

Agent Teams 是实验性的。在 `~/.claude/settings.json` 中：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

需要 Claude Code v2.1.32 或更高版本。本仓库的 persona 会自动被识别——无需手动编写团队配置文件。

### 触发提示

在主导会话中输入自然语言：

```
用户报告结账间歇性挂起约 30 秒，始于上周
发布之后。日志中无错误。

创建一个 agent 团队用竞争假设来调试此问题。使用
已有的 agent 类型派生三个队友：

  - code-reviewer  — 调查结账代码路径中的竞态条件和阻塞调用
  - security-auditor — 调查认证检查、会话处理，
                       以及最近添加的任何同步网络调用
  - test-engineer  — 提出能够区分各假设的测试，
                     并检查结账的覆盖缺口

让他们直接互发消息来挑战彼此的理论。
随着共识出现更新发现。只有当两个队友同意
能反驳其他人的理论时才收敛。
```

主导会话派生三个队友，引用已有的 persona 名称。Persona 主体被 **追加** 到每个队友的系统提示作为额外指令（叠加在主导会话安装的团队协调指令之上）；上面的触发提示成为它们的任务。

### 发生过程

1. 每个队友在自己的上下文窗口中运行，从自己的视角探索代码库。
2. 队友使用 `message` 直接向彼此发送发现。主导会话无需转发。
3. 共享任务列表显示谁在调查什么——随时可通过 `Ctrl+T`（进程内模式）或 tmux 面板（分屏模式）查看。
4. 当 `code-reviewer` 发现一个应该是顺序的 `Promise.all` 时，它向 `security-auditor` 发消息确认认证调用不是竞态的一部分。`security-auditor` 检查后回复——确认竞态是真正问题或提供反证。
5. `test-engineer` 为正在胜出的理论提出聚焦的集成测试，团队在声明共识前用其验证。
6. 主导会话综合收敛的发现并向你呈现。

你可以随时通过 `Shift+Down` 切换到某个队友并输入——有助于重新引导走偏方向的调查者。

### 清理时机

当调查确定根因后，告诉主导会话：

```
清理团队
```

始终通过主导会话清理，而非队友（文档说明：队友缺乏完整的团队上下文来进行清理）。

### 成本预期

三个 Sonnet 队友运行约 10–15 分钟的调查，成本明显高于 `/ship` 以子 agent 方式派生同样的三个 persona。理由是 *结论质量*——对于生产调试，错误的修复代价高昂，额外 token 是划算的。对于常规 PR 审查，坚持 `/ship`。

### 此场景中的反模式

**不要**将此重构为扇出子 agent 的 `/debug` 斜杠命令。子 agent 不能互相发消息——你会失去使模式生效的对抗性辩论。如果一个工作流反复出现，将上面的触发提示记录为片段，而非包装成滥用子 agent 的斜杠命令。

### 何时 *不* 使用 Agent Teams

- 对已知 diff 的生产发布裁决 → 使用 `/ship`（子 agent）。
- 对单个产出物的单个专家视角 → 直接 persona 调用。
- 顺序生命周期（spec → plan → build）→ 用户驱动的斜杠命令（模式 4）。
- 读密集型研究且摘要小 → 内置 `Explore` 子 agent。

仅在队友 **需要** 挑战彼此以产出正确答案时才使用 Agent Teams。

---

## 反模式

### A. 路由器 persona（"元编排者"）

一个 persona 的职责是决定调用哪个其他 persona。

```
/work → router-persona → "this needs a review" → code-reviewer → router (paraphrases) → user
```

**失败原因：**
- 纯路由层，无领域价值
- 增加两个改写轮次 → 信息丢失 + 大约 2× token 成本
- 用户本来就知道需要审查；他们可以直接调用 `/review`
- 重复了斜杠命令和 `AGENTS.md` 中意图映射已做的事

**替代做法：** 添加或改进斜杠命令。在 `AGENTS.md` 中记录意图 → 命令映射。

---

### B. 调用其他 persona 的 persona

一个 `code-reviewer` 在看到认证代码时内部调用 `security-auditor`。

**失败原因：**
- Persona 被设计为产出单一视角；串联它们违背了这一点
- 调用 persona 传递的摘要丢失了被调用 persona 需要的上下文
- 失败模式倍增（谁的输出格式优先？谁的规则适用？）
- 对用户隐藏了成本

**替代做法：** 让调用 persona 在其报告中 *推荐* 后续审计。用户或斜杠命令运行第二轮。

---

### C. 改写式顺序编排者

一个 agent 替用户依次调用 `/spec`、`/plan`、`/build` 等。

**失败原因：**
- 丢失了本应捕获方向错误的人类检查点
- 每次交接会摘要上下文——长流水线中累积漂移
- token 成本翻倍：每步都是编排者轮次 + 子 agent 轮次
- 在判断最关键的节点上剥夺了用户的主动权

**替代做法：** 保持用户作为编排者。在 `README.md` 中记录推荐顺序，让用户自行调用。

---

### D. 深层 persona 树

`/ship` 调用 `pre-ship-coordinator`，它调用 `quality-coordinator`，它又调用 `code-reviewer`。

**失败原因：**
- 每层增加延迟和 token，但无决策价值
- 调试变成多级调查
- 叶 persona 因多次摘要丢失上下文

**替代做法：** 编排深度最多为 1（斜杠命令 → persona）。合并在主 agent 中进行。

---

## 决策流程

考虑新编排工作流时，走此流程：

```
工作是否是对单个产出物的单一视角？
├── 是 → 直接调用。结束。
└── 否  → 同样的组合是否会重复？
         ├── 否  → 直接调用，临时。结束。
         └── 是 → 子任务是否独立？
                  ├── 否  → 用户运行的顺序斜杠命令（模式 4）。
                  └── 是 → 并行扇出 + 合并（模式 3）。
                           对照上述检查清单验证。
                           任何检查失败 → 回退到单 persona 命令（模式 2）。
```

---

## 何时向此目录添加新模式

仅在以下条件满足后添加新条目：

1. 你在实际工作中至少使用了该模式两次
2. 你能指出本仓库中演示该模式的具体产出物
3. 你能解释为什么现有模式不适用
4. 你能描述其反模式影子（人们会错误地构建什么）

过早的目录条目会成为没人遵循的愿景文档。