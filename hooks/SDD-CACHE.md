# sdd-cache hook

[`source-driven-development`](../skills/source-driven-development/SKILL.md) 的跨会话引用缓存。跳过冗余的 `WebFetch` 调用，而不削弱该技能"对照当前文档验证"的保证。

## 原因

`source-driven-development` 为每个框架相关决策获取官方文档。跨会话在同一项目上工作意味着反复获取相同页面。将内容缓存为本地记忆会与技能矛盾——文档会变化，过期缓存会掩盖这一点。

此 hook 将获取的内容缓存在磁盘上，但 **每次重用时通过 HTTP `If-None-Match` / `If-Modified-Since` 向源服务器重新验证**。内容仅在服务器响应 `304 Not Modified` 时才从缓存提供——这是新鲜验证，而非记忆读取。

## 设置

1. 将 hooks 添加到 `.claude/settings.json`（或 `.claude/settings.local.json` 用于个人使用）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "WebFetch",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PROJECT_DIR}/hooks/sdd-cache-pre.sh",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "WebFetch",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PROJECT_DIR}/hooks/sdd-cache-post.sh",
            "async": true,
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

   `${CLAUDE_PROJECT_DIR}` 解析为你启动 Claude Code 的目录。上面的片段适用于 hooks 在同一项目内的情况。如果你在其他位置安装了 `agent-skills`（例如作为 `~/agent-skills` 下的共享插件），将 `${CLAUDE_PROJECT_DIR}/hooks/...` 替换为每个脚本的绝对路径。

2. 确保 `.claude/sdd-cache/` 在你的 `.gitignore` 中（本仓库已包含）。

3. 正常使用 `/source-driven-development`（或该技能）。技能或 agent 的工作流无需更改——缓存是透明的。

## 心智模型

以 URL 为键的 HTTP 资源缓存。新鲜度通过 `ETag` / `Last-Modified` 委托给源服务器；没有 TTL，没有提示词在键中。

存储的正文不是原始 HTML——`WebFetch` 通过模型使用调用者的提示词对每个响应进行后处理，所以我们缓存的是一个 agent 对页面的解读。键仅保留 URL 以便跨会话复用读取；原始提示词作为元数据保留并在命中消息中展示，以便下一个 agent 判断之前的解读是否适用。

## 工作原理

每个 URL 一个缓存条目，以 JSON 存储在 `.claude/sdd-cache/<sha>.json`：

| 事件 | 动作 |
|---|---|
| `PreToolUse WebFetch` | 如果条目存在，发送带 `If-None-Match` / `If-Modified-Since` 的 `HEAD` 请求。收到 `304` 时，阻止获取并通过 stderr 将缓存内容返回给 agent，原始提示词作为元数据展示。否则允许获取。 |
| `PostToolUse WebFetch` | 捕获响应，发送 `HEAD` 请求记录当前 `ETag` / `Last-Modified`，存储 `{url, prompt, etag, last_modified, content, fetched_at}`。 |

**新鲜度规则：**

- 条目仅在源服务器确认 `304 Not Modified` 时才被提供。
- 没有 `ETag` 或 `Last-Modified` 头的条目不会被缓存——没有验证器，hook 无法在后续验证新鲜度，缓存就意味着信任记忆。
- 缓存键是 `sha256(url)`。同一 URL 用不同提示词请求命中同一条目；缓存正文反映首次获取时使用的提示词，该提示词在命中时一并展示，以便 agent 决定是复用还是手动重新获取。

**Agent 看到的内容：**

- 缓存命中：`WebFetch` 通过退出码 2 被阻止。Claude Code 将 hook 的 stderr payload 作为工具错误返回给 agent——这是缓存命中的预期信号，不是失败。payload 以 `[sdd-cache] Cache hit for <url>` 为前缀，缓存正文包裹在 `----- BEGIN CACHED CONTENT -----` / `----- END CACHED CONTENT -----` 标记之间，agent 可以像 `WebFetch` 刚返回的那样使用它。
- 缓存未命中或过期：`WebFetch` 正常运行；结果为下次存储。

技能本身不变。它继续遵循 `DETECT → FETCH → IMPLEMENT → CITE`。hook 仅改变 `FETCH` 运行时底层发生的事。

## 本地测试

### 1. 直接冒烟测试脚本

```bash
# 模拟 PostToolUse payload：缓存一个页面
echo '{
  "tool_input": {
    "url": "https://react.dev/reference/react/useActionState",
    "prompt": "extract the signature"
  },
  "tool_response": "useActionState(action, initialState) returns [state, formAction, isPending]"
}' | bash hooks/sdd-cache-post.sh

# 查看存储的条目
ls .claude/sdd-cache/
cat .claude/sdd-cache/*.json | jq .

# 模拟同一 URL + 提示词的下次 PreToolUse
echo '{
  "tool_input": {
    "url": "https://react.dev/reference/react/useActionState",
    "prompt": "extract the signature"
  }
}' | bash hooks/sdd-cache-pre.sh
echo "exit=$?"
```

预期：

- 第一个命令在 `.claude/sdd-cache/` 下创建一个文件（仅当服务器返回 `ETag` 或 `Last-Modified` 时）。
- 第二个命令在源服务器回复 `304` 时退出码 `2` 并在 stderr 上输出缓存内容，否则退出 `0` 且无输出。

### 2. 在真实会话中端到端测试

1. 在 `.claude/settings.local.json` 中注册 hooks（如上所示）。
2. 在此仓库中启动 Claude Code 会话。
3. 让 agent 获取一个文档页面（例如 "fetch `https://react.dev/reference/react/useActionState` and summarize"）。
4. 验证 `.claude/sdd-cache/` 下出现文件。
5. 让 agent 用相同提示词再次获取同一页面。
6. 验证第二次 `WebFetch` 被阻止并返回缓存内容（在会话转录中可见为带 `[sdd-cache]` 前缀的工具错误）。

### 3. 新鲜度验证

要确认文档变化时缓存会失效，强制制造 `ETag` 不匹配。选择一个具体条目——`*.json` 在缓存持有多个文件时不安全：

```bash
# 选择你要损坏的条目（替换为实际文件名）
ENTRY=.claude/sdd-cache/e49c9f378670cfbb1d7d871b6dee16d9.json

# 将其 ETag 改为源服务器不会识别的值
jq '.etag = "W/\"stale-etag-forced\""' "$ENTRY" > "$ENTRY.tmp" && mv "$ENTRY.tmp" "$ENTRY"

# 下次 PreToolUse 应未命中（服务器返回 200，而非 304）
echo '{"tool_input":{"url":"...", "prompt":"..."}}' | bash hooks/sdd-cache-pre.sh
echo "exit=$?"   # 预期 0（允许获取通过）
```

### 4. 调试

两个 hooks 在调试模式开启时会将带时间戳的事件写入 `.claude/sdd-cache/.debug.log`。通过以下任一方式启用：

```bash
# 方式 A：环境变量（按会话）
SDD_CACHE_DEBUG=1 claude

# 方式 B：标记文件（持久）
mkdir -p .claude/sdd-cache && touch .claude/sdd-cache/.debug
# …禁用：rm .claude/sdd-cache/.debug
```

日志记录 URL、检测到的 `tool_response` 形状、HEAD 状态和每次调用命中或未命中的原因。在缓存未命中看起来不正常时有用（通常是：源服务器停止发出验证器）。

## 已知限制

- **正文由提示词塑造。** 命中返回之前 agent 对页面的解读，原始提示词一并展示以便当前 agent 判断其是否适用。如果不适用，删除 `.claude/sdd-cache/` 下的文件以强制重新获取。
- **每次缓存写入需要额外一次 HEAD。** Claude Code 不暴露 `WebFetch` 已接收的响应头，所以 post hook 重新查询源服务器以捕获 `ETag` / `Last-Modified`。每次未命中多一次往返——保持此为纯 hook 而不修改核心的代价。
- **没有 `ETag` 或 `Last-Modified` 的服务器不会被缓存。** 大多数官方文档站点（react.dev、docs.djangoproject.com、developer.mozilla.org）发出验证器。不发出的站点总是重新获取。
- **行为异常的服务器可能返回错误的 `304`。** 这是需要诊断的服务器 bug，不是缓存不变量需要防御的；我们不通过 TTL 来掩盖它。发现过期条目时删除它。
- **缓存是本地且项目级别的。** 没有团队共享缓存。添加一个需要签名内容寻址存储层，超出范围。

## 依赖

- `jq`
- `curl`
- `shasum` 或 `sha256sum`（自动检测）
- Bash 3.2+