# simplify-ignore hook

`/code-simplify` 的块级保护。标记不应被简化代码——模型不会看到它。

## 设置

1. 注释你要保护的代码块：

```js
/* simplify-ignore-start: perf-critical */
// manually unrolled XOR — 3x faster than a loop
result[0] = buf[0] ^ key[0];
result[1] = buf[1] ^ key[1];
result[2] = buf[2] ^ key[2];
result[3] = buf[3] ^ key[3];
/* simplify-ignore-end */
```

2. 将 hooks 添加到 `.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{ "type": "command", "command": "bash ${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "bash ${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh" }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "bash ${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh" }]
      }
    ]
  }
}
```

3. 运行 `/code-simplify`——受保护的块变为 `/* BLOCK_de115a1d: perf-critical */` 占位符。模型推理周围代码时不会看到受保护的实现。

> **注意：** Hook 在 `.claude/.simplify-ignore-cache/` 中存储临时备份。确保此路径在你的 `.gitignore` 中。

## 工作原理

一个脚本，三个 hook 事件：

| 事件 | 动作 |
|---|---|
| `PreToolUse Read` | 备份文件，就地将代码块替换为 `BLOCK_<hash>` 占位符 |
| `PostToolUse Edit\|Write` | 将占位符还原为真实代码，保存模型更改，重新过滤 |
| `Stop` | 会话结束时从备份还原所有文件 |

每个代码块通过内容哈希（8 位十六进制字符，通过 `shasum`/`sha1sum`）标识，使往返明确无误，即使模型复制或重排占位符。缓存按项目范围，防止跨会话干扰。

## 注解语法

```js
/* simplify-ignore-start */           // 基础——隐藏该块
/* simplify-ignore-start: reason */   // 附带原因——出现在占位符中
/* simplify-ignore-end */
```

任何注释风格都可用（`//`、`/*`、`#`、`<!--`）。支持每个文件多个块和单行块。占位符保留原始注释语法（例如 Python 用 `# BLOCK_xxx`，HTML 用 `<!-- BLOCK_xxx -->`）。

## 崩溃恢复

如果 Claude Code 崩溃且未触发 Stop hook，磁盘上的文件可能仍有 `BLOCK_<hash>` 占位符。手动恢复：

```bash
echo '{}' | bash hooks/simplify-ignore.sh
```

备份存储在项目目录内的 `.claude/.simplify-ignore-cache/` 中。

## 已知限制

- **单行块会隐藏整行。** 如果 `simplify-ignore-start` 和 `simplify-ignore-end` 出现在与其他代码相同的行上，整行从模型视角被隐藏，不仅仅是标注部分。注释应使用独立行。
- **注释后缀检测仅覆盖 `*/` 和 `-->`。** 使用非标准注释结束符的模板引擎（ERB `%>`、Blade `--}}`）可能产生不平衡的占位符。改用 `#` 或 `//` 风格注释。
- **回退还原是渐进式的，不是精确的。** 如果模型更改了占位符的格式（例如改变了原因文本），hook 逐步尝试更简的匹配：完整占位符 → 前缀+哈希+后缀 → 仅哈希。仅哈希回退可能留下外观残留（例如多余的 `:` 或原因文本）。发生时 stderr 会打印警告。
- **文件重命名会留下占位符。** 如果模型通过 shell 命令重命名或移动文件，新文件将保留 `BLOCK_<hash>` 占位符。原始代码在会话停止时保存为 `<old-filename>.recovered`。你必须手动将恢复的代码还原到新文件中。

## 依赖

- `jq`、`shasum` 或 `sha1sum`（自动检测），Bash 3.2+